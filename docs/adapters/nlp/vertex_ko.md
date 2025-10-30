# Vertex AI 서비스 어댑터 문서

## 개요

Vertex AI 서비스 어댑터는 Google Cloud의 Vertex AI 플랫폼과의 통합을 제공하며, 각각의 API를 통해 Anthropic Claude 모델과 Google Gemini 모델을 모두 지원합니다. 이 어댑터는 텍스트 생성, 임베딩 및 토큰화를 위한 Parlant NLP 서비스 인터페이스를 구현합니다.

## 아키텍처

### 핵심 컴포넌트

- **VertexAIService**: NLPService 인터페이스를 구현하는 메인 서비스 클래스
- **VertexAIClaudeSchematicGenerator**: Anthropic Vertex API를 통한 Claude 모델용 생성기
- **VertexAIGeminiSchematicGenerator**: Google Gen AI API를 통한 Gemini 모델용 생성기
- **VertexAIEmbedder**: Google의 text-embedding-004 모델을 사용하는 텍스트 임베딩 서비스
- **VertexAIEstimatingTokenizer**: Claude 및 Gemini 모델 모두에 대한 토큰 카운팅

## 설정

### 환경 변수

```bash
# 필수
VERTEX_AI_PROJECT_ID=your-gcp-project-id
VERTEX_AI_REGION=us-central1  # 사용하는 리전을 입력하세요
VERTEX_AI_MODEL=claude-opus-4
```

### 인증

어댑터는 Google Application Default Credentials (ADC)를 사용합니다:

```bash
# 로컬 개발용
gcloud auth application-default login

# 프로덕션의 경우, 서비스 계정 키 또는 workload identity를 사용하세요
```

## 지원 모델

### Claude 모델 (Anthropic Vertex API 사용)

| 짧은 이름 | 전체 모델 이름 | 설명 |
|------------|-----------------|-------------|
| `claude-opus-4` | `claude-opus-4@20250514` | 가장 강력한 Claude 모델 |
| `claude-sonnet-4` | `claude-sonnet-4@20250514` | 균형잡힌 성능과 속도 |
| `claude-sonnet-3.5` | `claude-3-5-sonnet-v2@20241022` | 이전 세대 Sonnet |
| `claude-haiku-3.5` | `claude-3-5-haiku@20241022` | 가장 빠른 Claude 모델 |

### Gemini 모델 (Google Gen AI API 사용)

| 짧은 이름 | 전체 모델 이름 | 설명 |
|------------|-----------------|-------------|
| `gemini-2.5-flash` | `gemini-2.5-flash` | 최신 고속 Gemini 모델 |
| `gemini-2.5-pro` | `gemini-2.5-pro` | 최신 프로 Gemini 모델 |
| `gemini-2.0-flash` | `gemini-2.0-flash` | 이전 세대 flash |
| `gemini-1.5-flash` | `gemini-1.5-flash` | 1M 토큰 컨텍스트 |
| `gemini-1.5-pro` | `gemini-1.5-pro` | 2M 토큰 컨텍스트 |

## 사용법

### 기본 설정

```python
import parlant.sdk import p
from parlant.sdk import NLPServices

async with p.Server(nlp_service=NLPServices.vertex) as server:
        agent = await server.create_agent(
            name="Healthcare Agent",
            description="Is empathetic and calming to the patient.",
        )
```

### 직접 서비스 사용

```python
from parlant.adapters.nlp.vertex_service import VertexAIService
from parlant.core.loggers import Logger

# 서비스 초기화
logger = Logger()
service = VertexAIService(logger=logger)

# 스키마틱 생성기 가져오기
generator = await service.get_schematic_generator(YourSchemaClass)

# 콘텐츠 생성
result = await generator.generate(
    prompt="Your prompt here",
    hints={"temperature": 0.7, "max_tokens": 1000}
)
```

## API 레퍼런스

### VertexAIService

NLPService 인터페이스를 구현하는 메인 서비스 클래스입니다.

#### 생성자

```python
def __init__(self, logger: Logger) -> None
```

환경 변수로 서비스를 초기화합니다:
- `VERTEX_AI_PROJECT_ID`, `VERTEX_AI_REGION`, `VERTEX_AI_MODEL` 읽기
- Application Default Credentials 검증
- 로깅 설정

#### 메서드

##### get_schematic_generator

```python
async def get_schematic_generator(self, t: type[T]) -> SchematicGenerator[T]
```

설정된 모델에 따라 적절한 생성기를 반환합니다:
- Claude 모델 → VertexAIClaudeSchematicGenerator
- Gemini 모델 → VertexAIGeminiSchematicGenerator
- Claude Opus 4에 대한 폴백 로직 포함

##### get_embedder

```python
async def get_embedder(self) -> Embedder
```

VertexTextEmbedding004 임베더 인스턴스를 반환합니다.

##### get_moderation_service

```python
async def get_moderation_service(self) -> ModerationService
```

NoModeration 서비스를 반환합니다 (조정 기능은 아직 구현되지 않음).

### VertexAIClaudeSchematicGenerator

Anthropic Vertex API를 통한 Claude 모델용 스키마틱 생성기입니다.

#### 지원 힌트

- `temperature`: 무작위성 제어 (0.0-1.0)
- `max_tokens`: 최대 출력 토큰
- `top_p`: Nucleus 샘플링 파라미터
- `top_k`: Top-k 샘플링 파라미터

#### 속성

- `id`: `vertex-ai/{model_name}` 반환
- `tokenizer`: VertexAIEstimatingTokenizer 인스턴스 반환
- `max_tokens`: 200,000 반환 (Claude 컨텍스트 제한)

#### 메서드

##### generate

```python
async def generate(
    self,
    prompt: str | PromptBuilder,
    hints: Mapping[str, Any] = {},
) -> SchematicGenerationResult[T]
```

다음을 포함하여 Claude 모델을 사용해 구조화된 콘텐츠를 생성합니다:
- JSON 스키마 검증
- 속도 제한 및 에러에 대한 재시도 정책
- 사용량 추적

### VertexAIGeminiSchematicGenerator

Google Gen AI API를 통한 Gemini 모델용 스키마틱 생성기입니다.

#### 지원 힌트

- `temperature`: 무작위성 제어 (0.0-1.0)
- `thinking_config`: 추론 모델에 대한 설정

#### 속성

- `id`: `vertex-ai/{model_name}` 반환
- `tokenizer`: VertexAIEstimatingTokenizer 인스턴스 반환
- `max_tokens`: 1M (Flash) 또는 2M (Pro) 토큰 반환

#### 메서드

##### generate

```python
async def generate(
    self,
    prompt: str | PromptBuilder,
    hints: Mapping[str, Any] = {},
) -> SchematicGenerationResult[T]
```

다음을 포함하여 Gemini 모델을 사용해 구조화된 콘텐츠를 생성합니다:
- 네이티브 JSON 스키마 지원
- JSON 파싱 및 검증
- 사용량 메타데이터 추적

### VertexAIEmbedder

Google의 text-embedding-004 모델을 사용하는 텍스트 임베딩 서비스입니다.

#### 속성

- `id`: `vertex-ai/text-embedding-004` 반환
- `dimensions`: 768 반환 (임베딩 차원)
- `max_tokens`: 8,192 반환 (입력 토큰 제한)

#### 지원 힌트

- `title`: 더 나은 임베딩을 위한 문서 제목
- `task_type`: 임베딩 작업 타입 (기본값: "RETRIEVAL_DOCUMENT")

#### 메서드

##### embed

```python
async def embed(
    self,
    texts: list[str],
    hints: Mapping[str, Any] = {},
) -> EmbeddingResult
```

배치 처리를 지원하여 입력 텍스트에 대한 임베딩을 생성합니다.

### VertexAIEstimatingTokenizer

Claude 및 Gemini 모델을 모두 지원하는 토큰 카운팅 서비스입니다.

#### 메서드

##### estimate_token_count

```python
async def estimate_token_count(self, prompt: str) -> int
```

다음을 사용하여 토큰 수를 추정합니다:
- Claude 모델용 tiktoken
- Gemini 모델용 Google Gen AI API

## 에러 처리

### 인증 에러

```python
class VertexAIAuthError(Exception):
    """Vertex AI 인증 문제가 있을 때 발생합니다."""
```

일반적인 원인 및 해결방법:
- ADC 누락: `gcloud auth application-default login` 실행
- 권한 부족: "Vertex AI User" 역할 확인
- 모델이 활성화되지 않음: Vertex AI Model Garden 확인

### 속도 제한

어댑터는 포괄적인 재시도 정책을 구현합니다:

#### Claude 모델
- 재시도: APIConnectionError, APITimeoutError, RateLimitError, APIResponseValidationError
- 최대 시도: 지수 백오프로 3회 (1초, 2초, 4초)
- 서버 에러: 더 긴 지연으로 2회 시도 (1초, 5초)

#### Gemini 모델
- 재시도: NotFound, TooManyRequests, ResourceExhausted
- 최대 시도: 지수 백오프로 3회 (1초, 2초, 4초)
- 서버 에러: 더 긴 지연으로 2회 시도 (1초, 5초)

### 에러 메시지

어댑터는 일반적인 문제에 대한 상세한 에러 메시지를 제공합니다:

#### 속도 제한 초과
```
Vertex AI 속도 제한을 초과했습니다. 가능한 이유:
1. GCP 프로젝트에 할당량이 부족할 수 있습니다.
2. Vertex AI Model Garden에서 모델이 활성화되지 않았을 수 있습니다.
3. 분당 요청 제한을 초과했을 수 있습니다.

권장 조치:
- GCP 콘솔에서 Vertex AI 할당량을 확인하세요.
- Vertex AI Model Garden에서 모델이 활성화되어 있는지 확인하세요.
- 서비스 계정의 IAM 권한을 검토하세요.
- 방문: https://console.cloud.google.com/vertex-ai/model-garden
```

#### 권한 거부
```
Vertex AI 액세스가 거부되었습니다. 다음을 확인하세요:
1. ADC가 올바르게 구성되어 있는지 ('gcloud auth application-default login' 실행)
2. 서비스 계정에 'Vertex AI User' 역할이 있는지
3. Vertex AI Model Garden에서 {model_name} 모델이 활성화되어 있는지
```

## 성능 고려사항

### 토큰 제한

| 모델 타입 | 컨텍스트 제한 | 권장 사용 |
|------------|---------------|-------------------|
| Claude 모델 | 200K 토큰 | 긴 문서, 복잡한 추론 |
| Gemini Flash | 1M 토큰 | 대용량 컨텍스트 처리 |
| Gemini Pro | 2M 토큰 | 최대 컨텍스트 요구사항 |

## 모범 사례

### 모델 선택

1. **Claude Sonnet 3.5**: 성능과 비용의 최상의 균형
2. **Claude Opus 4**: 폴백이 있는 최대 성능
3. **Gemini 2.5 Flash**: 대용량 컨텍스트를 사용한 빠른 처리
4. **Gemini 2.5 Pro**: 복잡한 추론 작업

### 설정
```python
   export VERTEX_AI_PROJECT_ID=your-project-id
   export VERTEX_AI_REGION=us-central1
   export VERTEX_AI_MODEL=claude-sonnet-3.5
```

### 에러 처리

```python
from parlant.adapters.nlp.vertex_service import VertexAIAuthError

try:
    service = VertexAIService(logger=logger)
    generator = await service.get_schematic_generator(MySchema)
    result = await generator.generate(prompt)
except VertexAIAuthError as e:
    logger.error(f"Authentication failed: {e}")
    # 인증 설정 처리
except Exception as e:
    logger.error(f"Generation failed: {e}")
    # 기타 에러 처리
```

## 문제 해결

### 일반적인 문제

1. **인증 실패**
   - ADC 설정 확인: `gcloud auth application-default print-access-token`
   - GCP 콘솔에서 프로젝트 권한 확인
   - 서비스 계정에 필요한 역할이 있는지 확인

2. **모델 액세스 거부**
   - Vertex AI Model Garden에서 모델 활성화
   - 지역 가용성 확인
   - 청구 계정이 활성화되어 있는지 확인

3. **속도 제한**
   - GCP 콘솔에서 할당량 사용량 모니터링
   - 애플리케이션 레벨 속도 제한 구현
   - 서비스 티어 업그레이드 고려

### 디버깅

생성된 메시지를 검사하여 플레이그라운드 UI에서 사용량을 확인할 수 있습니다.

## 마이그레이션 가이드

### 다른 어댑터에서 마이그레이션

다른 NLP 어댑터에서 마이그레이션할 때:

1. **환경 변수 업데이트**
   ```bash
   # 이전 변수 제거
   unset OPENAI_API_KEY ANTHROPIC_API_KEY

   # Vertex AI 변수 설정
   export VERTEX_AI_PROJECT_ID=your-project-id
   export VERTEX_AI_REGION=us-central1
   export VERTEX_AI_MODEL=claude-opus-4
   ```

2. **모델 이름 매핑**
   - `gpt-4` → `claude-opus-4`
   - `gpt-3.5-turbo` → `gemini-2.5-flash`
   - `claude-3-sonnet` → `claude-opus-4`

## 기여하기

### 새 모델 추가

1. **제공자 결정**: 모델이 Anthropic 또는 Google API를 사용하는지 확인
2. **모델 클래스 생성**: 적절한 베이스 생성기에서 상속
3. **서비스 업데이트**: VertexAIService에 모델 매핑 추가
4. **테스트 추가**: 새 모델에 대한 통합 테스트 포함
5. **문서 업데이트**: 지원 모델 테이블에 모델 추가

### 코드 스타일

- 에러 처리를 위한 기존 패턴 따르기
- 포괄적인 로깅 포함
- 모든 메서드에 타입 힌트 추가
- docstring으로 공개 API 문서화
- 외부 API 호출에 대한 재시도 정책 사용

## 사전 요구사항 및 설치

### 설치

Parlant와 함께 Vertex AI 서비스 어댑터를 사용하려면 적절한 선택적 의존성을 설치해야 합니다:

```bash
pip install "parlant[vertex]"
```

이 설치에는 Vertex AI 플랫폼을 통한 Claude 및 Gemini 모델 모두에 대한 지원이 포함됩니다.

### 중요한 모델 지원 중단 공지

⚠️ **Claude 3.5 Sonnet 모델 지원 중단**: Claude Sonnet 3.5 모델 (claude-3-5-sonnet-20240620 및 claude-3-5-sonnet-20241022)은 2025년 10월 22일에 종료됩니다. 향상된 성능과 기능을 위해 Claude Sonnet 4 (claude-sonnet-4-20250514)로 마이그레이션하는 것을 권장합니다.

## 인증 설정

어댑터를 사용하기 전에 적절한 인증이 구성되어 있는지 확인하세요:

```bash
# 로컬 개발용
gcloud auth application-default login

# 인증 확인
gcloud auth application-default print-access-token
```

## 필요한 권한

서비스 계정 또는 사용자에게 다음 IAM 역할이 있는지 확인하세요:
- `Vertex AI User` - Vertex AI 서비스 액세스용
- `AI Platform User` - 모델 액세스용 (레거시 역할, 일부 모델에 필요할 수 있음)

## 라이선스

Apache License, Version 2.0에 따라 라이선스가 부여됩니다. 전체 라이선스 텍스트는 소스 파일 헤더를 참조하세요.

## 관리자

Agam Dubey - hello.world.agam@gmail.com
