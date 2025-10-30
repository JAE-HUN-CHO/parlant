# Azure OpenAI 서비스 문서

Azure 서비스는 Azure OpenAI 서비스와의 통합을 제공하며, 레거시 API 키 인증과 최신 Azure AD 인증을 모두 지원합니다. 이 통합을 통해 Parlant는 보안 모범 사례를 유지하면서 Azure의 엔터프라이즈급 AI 서비스를 활용할 수 있습니다.

## 전제조건

1. **Azure OpenAI 리소스**: Azure 구독에서 Azure OpenAI 리소스 생성
2. **인증 설정**: API 키 또는 Azure AD 인증 중 선택
3. **모델 배포**: Azure OpenAI 리소스에 필요한 모델 배포
4. **권한**: Azure AD 인증에 적절한 IAM 역할 보장

## 인증 방법

### 개발 (로컬 머신)
로컬 개발의 경우 Azure CLI 인증을 사용합니다:
```bash
# Azure CLI가 설치되지 않았으면 설치
# https://docs.microsoft.com/en-us/cli/azure/install-azure-cli

# Azure에 로그인
az login

# 엔드포인트 설정
export AZURE_ENDPOINT="https://your-resource.openai.azure.com/"
```

### 프로덕션 (서버 배포)
서버 배포의 경우 **`az login`을 사용하지 마세요**. 대신 다음 방법 중 하나를 사용하세요:

#### 옵션 1: 서비스 주체 (권장)
```bash
# 환경 변수 설정
export AZURE_ENDPOINT="https://your-resource.openai.azure.com/"
export AZURE_CLIENT_ID="your-service-principal-client-id"
export AZURE_CLIENT_SECRET="your-service-principal-secret"
export AZURE_TENANT_ID="your-azure-tenant-id"
```

#### 옵션 2: 관리 ID (Azure 리소스)
Azure VM, App Services 또는 기타 Azure 리소스에서 실행하는 경우:
```bash
# 엔드포인트만 설정 - 인증은 자동
export AZURE_ENDPOINT="https://your-resource.openai.azure.com/"
```

#### 옵션 3: 워크로드 ID (Kubernetes)
Kubernetes 배포의 경우:
```bash
export AZURE_ENDPOINT="https://your-resource.openai.azure.com/"
export AZURE_CLIENT_ID="your-workload-identity-client-id"
export AZURE_TENANT_ID="your-azure-tenant-id"
export AZURE_FEDERATED_TOKEN_FILE="/var/run/secrets/azure/tokens/azure-identity-token"
```

## 환경 변수

### 필수 변수
- `AZURE_ENDPOINT`: Azure OpenAI 리소스 엔드포인트

### 선택적 변수
- `AZURE_API_VERSION`: API 버전 (기본값: "2024-08-01-preview")
- `AZURE_GENERATIVE_MODEL_NAME`: 모델 이름 (기본값: "gpt-4o")
- `AZURE_GENERATIVE_MODEL_WINDOW`: 컨텍스트 윈도우 크기 (기본값: 4096)
- `AZURE_EMBEDDING_MODEL_NAME`: 임베딩 모델 (기본값: "text-embedding-3-large")
- `AZURE_EMBEDDING_MODEL_DIMS`: 임베딩 차원 (기본값: 3072)
- `AZURE_EMBEDDING_MODEL_WINDOW`: 임베딩 컨텍스트 윈도우 (기본값: 8192)

## 지원 모델

Azure 서비스는 Azure OpenAI 리소스에서 배포되고 사용 가능한 **모든 Azure OpenAI 모델**을 지원합니다. 아래에 나열된 모델은 미리 구성된 기본값이지만, 적절한 환경 변수를 설정하여 모든 모델을 사용할 수 있습니다.

### 미리 구성된 생성 모델

| 모델 이름 | 설명 | 컨텍스트 윈도우 | 사용 사례 |
|------------|-------------|---------------|----------|
| `gpt-4o` | 가장 강력한 GPT-4 모델 (기본값) | 128K 토큰 | 복잡한 추론, 높은 정확도 |
| `gpt-4o-mini` | 빠르고 비용 효율적인 GPT-4 | 128K 토큰 | 성능과 비용의 균형 |

### 미리 구성된 임베딩 모델

| 모델 이름 | 차원 | 컨텍스트 윈도우 | 설명 |
|------------|------------|---------------|-------------|
| `text-embedding-3-large` | 3072 | 8192 | 고품질 임베딩 (기본값) |
| `text-embedding-3-small` | 3072 | 8192 | 효율적인 임베딩 |

### 커스텀 모델 사용

Azure OpenAI 리소스에 배포된 **모든 Azure OpenAI 모델**을 사용할 수 있습니다:

```bash
# 모든 생성 모델 사용 (예시 - Azure 리소스에서 가용성 확인)
export AZURE_GENERATIVE_MODEL_NAME="gpt-35-turbo"  # GPT-3.5 Turbo
export AZURE_GENERATIVE_MODEL_NAME="gpt-4"        # GPT-4
export AZURE_GENERATIVE_MODEL_NAME="gpt-4-turbo"  # GPT-4 Turbo

# 모든 임베딩 모델 사용 (예시 - Azure 리소스에서 가용성 확인)
export AZURE_EMBEDDING_MODEL_NAME="text-embedding-ada-002"  # Ada 임베딩
export AZURE_EMBEDDING_MODEL_NAME="text-embedding-3-large" # 대형 임베딩
```

**중요**:
- 모델 가용성은 Azure OpenAI 리소스에 배포한 내용에 따라 다릅니다
- 모든 모델이 모든 Azure 지역에서 사용 가능한 것은 아닙니다
- 사용 가능한 모델을 보려면 Azure OpenAI 리소스 배포를 확인하세요

## 인증 우선순위

서비스는 다음과 같은 인증 우선순위를 따릅니다:

1. **API 키** (최우선 - 하위 호환성)
2. **Azure AD** (API 키가 없을 때의 대체)

## 필요한 Azure 권한

Azure AD 인증의 경우, 각 ID에 Azure OpenAI 리소스에 대한 다음 역할이 있는지 확인하세요:

- **Cognitive Services OpenAI User**: Azure OpenAI 서비스에 접근하는 데 필수

## 사용 예시

```python
import parlant.sdk as p
from parlant.sdk import NLPServices

async with p.Server(nlp_service=NLPServices.azure) as server:
        agent = await server.create_agent(
            name="Healthcare Agent",
            description="Is empathetic and calming to the patient.",
        )
```

## 서버 배포 가이드

### 프로덕션용 서비스 주체 설정

1. **서비스 주체 생성**:
   ```bash
   # 관리자 사용자로 로그인
   az login

   # 서비스 주체 생성
   az ad sp create-for-rbac --name "parlant-service-principal" --role "Cognitive Services OpenAI User" --scopes "/subscriptions/YOUR_SUBSCRIPTION_ID/resourceGroups/YOUR_RESOURCE_GROUP/providers/Microsoft.CognitiveServices/accounts/YOUR_OPENAI_RESOURCE"
   ```

2. **환경 변수 구성**:
   ```bash
   export AZURE_ENDPOINT="https://your-resource.openai.azure.com/"
   export AZURE_CLIENT_ID="step-1의-appId"
   export AZURE_CLIENT_SECRET="step-1의-password"
   export AZURE_TENANT_ID="step-1의-tenant"
   ```

3. **인증 테스트**:
   ```bash
   # 서비스 주체가 Azure OpenAI에 접근할 수 있는지 확인
   python -c "
   from parlant.adapters.nlp.azure_service import AzureService
   error = AzureService.verify_environment()
   print('Configuration OK' if error is None else f'Error: {error}')
   "
   ```

### 구성 팁

### 개발 설정
```bash
export AZURE_ENDPOINT="https://my-resource.openai.azure.com/"
export AZURE_API_KEY="your-api-key"
export AZURE_GENERATIVE_MODEL_NAME="gpt-4o-mini"
```

### 프로덕션 설정 (Azure AD)
```bash
export AZURE_ENDPOINT="https://my-resource.openai.azure.com/"
export AZURE_CLIENT_ID="your-client-id"
export AZURE_CLIENT_SECRET="your-client-secret"
export AZURE_TENANT_ID="your-tenant-id"
export AZURE_GENERATIVE_MODEL_NAME="gpt-4o"
```

## 문제 해결

### 일반적인 문제

1. **인증 실패**
   ```
   Azure authentication is not properly configured.
   ```
   **해결책**:
   - 개발의 경우: `az login` 실행 (로컬 개발에만 사용)
   - 프로덕션의 경우: 서비스 주체 변수 사용 (`az login` 사용 금지)
   - "Cognitive Services OpenAI User" 역할이 할당되었는지 확인
   - 서비스 주체에 올바른 권한이 있는지 확인

2. **속도 제한 오류**
   ```
   Azure API rate limit exceeded
   ```
   **해결책**:
   - Azure 계정 잔액 및 청구 상태 확인
   - Azure 대시보드에서 API 사용 제한 검토
   - 서비스 계층 업그레이드 고려

3. **모델 접근 거부**
   ```
   Model not found or access denied
   ```
   **해결책**:
   - Azure OpenAI 리소스에 모델이 배포되었는지 확인
   - 지역별 가용성 확인
   - 적절한 권한이 있는지 확인

4. **연결 오류**
   ```
   Cannot connect to Azure OpenAI endpoint
   ```
   **해결책**:
   - `AZURE_ENDPOINT`가 올바른지 확인
   - 네트워크 연결성 확인
   - 방화벽이 Azure OpenAI 트래픽을 허용하는지 확인

## 사용 가능한 모델 클래스

서비스는 편의를 위해 이러한 미리 구성된 모델 클래스를 제공하지만, 모든 Azure OpenAI 모델을 지원합니다:

### 미리 구성된 클래스
- `GPT_4o`: 가장 강력한 GPT-4 모델 (128K 컨텍스트) - **기본값**
- `GPT_4o_Mini`: 빠르고 비용 효율적인 GPT-4 (128K 컨텍스트)
- `AzureTextEmbedding3Large`: 고품질 임베딩 (3072 차원) - **기본값**
- `AzureTextEmbedding3Small`: 효율적인 임베딩 (3072 차원)

### 커스텀 모델 클래스
- `CustomAzureSchematicGenerator`: `AZURE_GENERATIVE_MODEL_NAME`을 통해 모든 생성 모델 사용
- `CustomAzureEmbedder`: `AZURE_EMBEDDING_MODEL_NAME`을 통해 모든 임베딩 모델 사용

**서비스는 환경 변수를 기반으로 자동으로 적절한 클래스를 선택합니다.**

### 모델 선택 작동 방식

서비스는 다음 논리를 사용하여 적절한 모델 클래스를 선택합니다:

```python
# 생성 모델 선택
if AZURE_GENERATIVE_MODEL_NAME is set:
    use CustomAzureSchematicGenerator  # 지정한 모든 모델
else:
    use GPT_4o  # 기본 모델

# 임베딩 모델 선택
if AZURE_EMBEDDING_MODEL_NAME is set:
    use CustomAzureEmbedder  # 지정한 모든 임베딩 모델
else:
    use AzureTextEmbedding3Large  # 기본 임베딩 모델
```

이는 코드 변경 없이 **모든 Azure OpenAI 모델**을 사용할 수 있음을 의미합니다 - 환경 변수만 설정하세요!

### 예시: 다양한 모델 사용

```bash
# GPT-3.5 Turbo 사용 (지역에서 사용 가능한 경우)
export AZURE_ENDPOINT="https://your-resource.openai.azure.com/"
export AZURE_GENERATIVE_MODEL_NAME="gpt-35-turbo"
export AZURE_EMBEDDING_MODEL_NAME="text-embedding-ada-002"

# GPT-4 Turbo 사용 (지역에서 사용 가능한 경우)
export AZURE_GENERATIVE_MODEL_NAME="gpt-4-turbo"
export AZURE_EMBEDDING_MODEL_NAME="text-embedding-3-large"

# 기본 모델 사용 (GPT-4o 및 text-embedding-3-large)
export AZURE_ENDPOINT="https://your-resource.openai.azure.com/"
# AZURE_GENERATIVE_MODEL_NAME 또는 AZURE_EMBEDDING_MODEL_NAME 설정할 필요 없음
```

## 보안 참고

- **API 키**: 안전하게 저장하고 정기적으로 회전
- **Azure AD**: 프로덕션에서 관리 ID 사용
- **네트워크**: 적절한 네트워크 보안 그룹 확인
- **모니터링**: 사용량 및 접근 패턴 모니터링
- **규정 준수**: 조직의 보안 정책 준수

## 마이그레이션 가이드

### API 키에서 Azure AD로

1. 지원되는 방법 중 하나를 사용하여 Azure AD 인증 설정
2. 환경 변수에서 API 키 제거
3. 권한 확인 - ID에 "Cognitive Services OpenAI User" 역할이 있는지 확인
4. `AzureService.verify_environment()`를 사용하여 구성 테스트

### 하위 호환성

서비스는 완전한 하위 호환성을 유지합니다:
- 기존 API 키 구성은 계속 작동합니다
- 기존 배포에는 변경 필요 없음
- Azure AD로의 점진적 마이그레이션이 지원됩니다
