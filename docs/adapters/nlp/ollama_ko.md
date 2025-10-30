# Ollama 서비스 문서

Ollama 서비스는 [Ollama](https://ollama.ai/)를 사용하여 Parlant에 로컬 LLM 기능을 제공합니다. 이 서비스는 다양한 오픈소스 모델을 사용하여 텍스트 생성 및 임베딩을 모두 지원합니다.

## 사전 요구사항

1. **Ollama 설치**: [ollama.ai](https://ollama.ai/)에서 다운로드 및 설치
2. **Ollama 서버 시작**: `ollama serve` 실행 (보통 자동으로 시작됨)
3. **필요한 모델 다운로드** ([권장 모델](#권장-모델) 섹션 참조)

## 환경 변수

다음 환경 변수를 사용하여 Ollama 서비스를 구성합니다:

```bash
# Ollama 서버 URL (기본값: http://localhost:11434)
export OLLAMA_BASE_URL="http://localhost:11434"

# 사용할 모델 크기 (기본값: 4b)
# 옵션: gemma3:1b, gemma3:4b, llama3.1:8b, gemma3:12b, gemma3:27b, llama3.1:70b, llama3.1:405b
export OLLAMA_MODEL="gemma3:4b"

# 임베딩 모델 (기본값: nomic-embed-text)
# 옵션: nomic-embed-text, mxbai-embed-large
export OLLAMA_EMBEDDING_MODEL="nomic-embed-text"

# API 타임아웃 (초 단위, 기본값: 300)
export OLLAMA_API_TIMEOUT="300"
```

### 예시 설정

```bash
# 개발용 (빠르고 균형잡힘)
export OLLAMA_MODEL="gemma3:4b"
export OLLAMA_EMBEDDING_MODEL="nomic-embed-text"
export OLLAMA_API_TIMEOUT="180"

# 높은 정확도의 클라우드
export OLLAMA_MODEL="gemma3:4b"
export OLLAMA_EMBEDDING_MODEL="nomic-embed-text"
export OLLAMA_API_TIMEOUT="600"
```

## 권장 모델

**⚠️ 중요**: 처음 사용 시 API 타임아웃을 피하기 위해 Parlant를 실행하기 전에 이러한 모델을 다운로드하세요:

### 텍스트 생성 모델

```bash
# 대부분의 사용 사례에 권장 (속도/정확도의 균형)
ollama pull gemma3:4b-it-qat

# 빠르지만 복잡한 스키마에 어려움을 겪을 수 있음
ollama pull gemma3:1b

# 임베딩 생성에 필요한 임베딩 모델
ollama pull nomic-embed-text
```

### 대형 모델 (클라우드/고성능 하드웨어 전용)

```bash
# 더 나은 추론 능력
ollama pull llama3.1:8b

# 복잡한 작업에 대한 높은 정확도
ollama pull gemma3:12b

# 매우 높은 정확도 (더 많은 리소스 필요)
ollama pull gemma3:27b-it-qat

# ⚠️ 경고: 40GB+ GPU 메모리 필요
ollama pull llama3.1:70b

# ⚠️ 경고: 200GB+ GPU 메모리 필요 (클라우드 전용)
ollama pull llama3.1:405b
```

### 임베딩 모델

사용자 정의 임베딩 모델을 사용하려면 OLLAMA_EMBEDDING_MODEL 환경 변수를 필요한 이름으로 설정하세요.
이 구현은 nomic-embed-text를 사용하여 테스트되었습니다.
**⚠️ 중요**:
자체 선택한 사용자 정의 임베딩 모델을 사용하는 것을 포함하여 다른 임베딩 모델 사용에 대한 지원이 추가되었습니다.
서버를 시작하기 전에 자체 임베딩 모델과 호환되는 OLLAMA_EMBEDDING_VECTOR_SIZE를 설정해야 합니다.
벡터 크기 1024로 `snowflake-arctic-embed`와 함께 테스트되었습니다.
지원되는 `nomic-embed-text`, `mxbai-embed-large` 또는 `bge-m3`을 사용하는 경우 OLLAMA_EMBEDDING_VECTOR_SIZE를 설정할 필요가 없습니다. 각각 벡터 크기는 기본적으로 768, 1024, 1024입니다.

```bash
# 대체 임베딩 모델 (512 차원)
ollama pull mxbai-embed-large:latest
```

## 사용 사례별 모델 권장사항

| 모델 크기 | 사용 사례 | 메모리 요구사항 | 성능 |
|------------|----------|-------------------|-------------|
| `1b` | 빠른 테스트, 간단한 작업 | ~2GB | 빠르지만 정확도 제한적 |
| `4b` | **개발용 권장** | ~4GB | 속도/정확도의 좋은 균형 |
| `8b` | 복잡한 추론 | ~8GB | Gemma보다 나은 추론 |
| `12b` | 고정확도 작업 | ~12GB | 높은 정확도, 느림 |
| `27b` | 복잡한 워크로드 | ~27GB | 매우 높은 정확도 |
| `70b` | 엔터프라이즈/클라우드 전용 | ~40GB+ | 뛰어난 정확도 |
| `405b` | 연구/클라우드 전용 | ~200GB+ | 최첨단 |

## 사용 예시

```python
import parlant.sdk as p
from parlant.sdk import NLPServices

async with p.Server(nlp_service=NLPServices.ollama) as server:
        agent = await server.create_agent(
            name="Healthcare Agent",
            description="Is empathetic and calming to the patient.",
        )
```

## 설정 팁

### 개발 설정
```bash
export OLLAMA_MODEL=gemma3:4b
export OLLAMA_API_TIMEOUT=180
```

### 고성능 설정 (클라우드)
```bash
export OLLAMA_MODEL=llama3.1:70b
export OLLAMA_API_TIMEOUT=300
```

### 사용자 정의 / 기타 모델
```bash
export OLLAMA_MODEL=llama3.2:3b
export OLLAMA_API_TIMEOUT=300
```

## 문제 해결

### 일반적인 문제

1. **모델을 찾을 수 없음 에러**
   ```
   Model gemma3:4b not found. Please pull it first with: ollama pull gemma3:4b
   ```
   **해결방법**: Parlant를 시작하기 전에 `ollama pull gemma3:4b-it-qat`를 실행하세요

2. **연결 에러**
   ```
   Cannot connect to Ollama server at http://localhost:11434
   ```
   **해결방법**: `ollama serve`로 Ollama가 실행 중인지 확인하세요

3. **타임아웃 에러**
   ```
   Request timed out after 300s
   ```
   **해결방법**: `OLLAMA_API_TIMEOUT`을 늘리거나 더 작은 모델을 사용하세요

4. **메모리 부족**
   ```
   CUDA out of memory
   ```
   **해결방법**: 더 작은 모델 크기를 사용하거나 GPU 메모리를 늘리세요

### 성능 최적화

1. **모델 사전 다운로드**: 처음 사용 전에 항상 모델을 다운로드하세요
2. **타임아웃 조정**: 큰 모델의 경우 타임아웃을 늘리세요
3. **모델 선택**: 정확도 요구사항을 충족하는 가장 작은 모델을 사용하세요
4. **GPU 메모리**: GPU 사용량을 모니터링하고 그에 따라 모델 크기를 조정하세요

## 사용 가능한 모델 클래스

서비스는 다음과 같은 사전 구성된 모델 클래스를 제공합니다:

- `OllamaGemma3_1B`: 빠름, 기본 정확도
- `OllamaGemma3_4B`: **권장** - 좋은 균형
- `OllamaLlama31_8B`: 더 나은 추론
- `OllamaGemma3_12B`: 높은 정확도
- `OllamaGemma3_27B`: 매우 높은 정확도
- `OllamaLlama31_70B`: 엔터프라이즈급 (높은 메모리)
- `OllamaLlama31_405B`: 연구급 (매우 높은 메모리)

## 보안 참고사항

- Ollama는 로컬에서 실행되므로 데이터가 기기 밖으로 나가지 않습니다
- API 키가 필요하지 않습니다
- 모델은 로컬에 다운로드되어 캐시됩니다
- Ollama 서버를 외부로 노출하는 경우 방화벽 규칙을 고려하세요
