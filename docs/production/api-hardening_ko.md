# API Hardening

Parlant는 무단 액세스 및 남용으로부터 API를 보호하는 강력한 인증 및 속도 제한 시스템을 제공합니다. 이 가이드는 프로덕션 배포를 보호하기 위한 맞춤형 인증 정책과 속도 제한기를 구현하는 방법을 설명합니다.

## 개요

API hardening 시스템은 두 가지 주요 구성 요소로 이루어져 있습니다:

1. **인증 정책** - 누가 어떤 리소스에 액세스하고 어떤 작업을 수행할 수 있는지 제어
2. **속도 제한기** - 요청 빈도를 제한하여 남용 방지

두 구성 요소는 함께 작동하여 포괄적인 API 보호를 제공하며, 액세스 토큰 또는 사용자 등급에 따라 다른 제한을 지원합니다.

## 인증 정책

### AuthorizationPolicy 추상 클래스 이해

모든 인증 정책은 세 가지 주요 메서드를 정의하는 `AuthorizationPolicy` 추상 기본 클래스를 상속합니다:

```python
class AuthorizationPolicy:
    @abstractmethod
    async def check_permission(
        self,
        request: fastapi.Request,
        permission: AuthorizationPermission
    ) -> bool:
        """요청이 작업을 수행할 권한이 있는지 확인"""
        ...

    @abstractmethod
    async def check_rate_limit(
        self,
        request: fastapi.Request,
        permission: AuthorizationPermission
    ) -> bool:
        """요청이 속도 제한 내에 있는지 확인"""
        ...

    async def authorize(
        self,
        request: fastapi.Request,
        permission: AuthorizationPermission
    ) -> None:
        """결합된 인증 확인 (권한 + 속도 제한)"""
        # 이 메서드는 일반적으로 재정의되지 않습니다. 기본 구현이
        # 두 추상 메서드를 순서대로 호출하고 거부되면 인증 오류를 발생시키기 때문입니다.
        ...
```

### 인증 권한

Parlant는 모든 API 작업을 다루는 포괄적인 권한 집합을 열거형으로 정의합니다:

- 에이전트 작업 (생성, 읽기, 업데이트, 삭제)
- 고객 관리
- 세션 처리
- 그 외 다수...

### 내장 인증 정책

#### DevelopmentAuthorizationPolicy
모든 작업을 허용 - 개발 환경에만 적합:

```python
class DevelopmentAuthorizationPolicy(AuthorizationPolicy):
    async def check_permission(
        self,
        request: fastapi.Request,
        permission: AuthorizationPermission
    ) -> bool:
        return True

    async def check_rate_limit(
        self,
        request: fastapi.Request,
        permission: AuthorizationPermission
    ) -> bool:
        return True
```

#### ProductionAuthorizationPolicy
구성 가능한 규칙으로 프로덕션 사용을 위한 더 엄격한 제어를 구현합니다.

## 맞춤형 인증 정책 구현

실제 배포에서 자체 인증 정책을 구현할 때, 처음부터 빌드하는 것보다 기존 프로덕션 정책을 확장하는 것이 일반적으로 좋습니다. 권장 접근 방식은 `ProductionAuthorizationPolicy`를 서브클래스화하고 특정 요구 사항에 맞게 커스터마이징하는 것입니다.

다음은 JWT 인증으로 맞춤형 정책을 생성하는 방법을 보여주는 참조 구현입니다:

```python
import parlant.sdk as p

import jwt
from fastapi import HTTPException
from limits import RateLimitItemPerMinute, RateLimitItemPerHour
from limits.storage import RedisStorage
from limits.strategies import SlidingWindowCounterRateLimiter

class CustomAuthorizationPolicy(p.ProductionAuthorizationPolicy):
    def __init__(self, secret_key: str, algorithm: str = "HS256"):
        super().__init__()
        self.secret_key = secret_key
        self.algorithm = algorithm

    async def _extract_token(self, request: fastapi.Request) -> dict | None:
        """요청에서 JWT 토큰 추출 및 검증"""
        auth_header = request.headers.get("Authorization")
        if not auth_header or not auth_header.startswith("Bearer "):
            return None

        token = auth_header.split(" ")[1]
        try:
            payload = jwt.decode(token, self.secret_key, algorithms=[self.algorithm])
            return payload
        except jwt.JWTError:
            # 유효하지 않은 토큰에 대해 403 발생, 누락된 토큰은 None이 괜찮음
            raise HTTPException(
                status_code=403,
                detail="Invalid access token"
            )

    async def check_permission(
        self,
        request: fastapi.Request,
        operation: p.Operation
    ) -> bool:
        """M2M 토큰 지원으로 향상된 권한 확인"""
        token_payload = await self._extract_token(request)

        # 유효한 M2M(machine-to-machine) 토큰이 있으면 추가 작업 허용
        if token_payload and token_payload.get("type") == "m2m":
            m2m_operations = {
                # M2M 토큰이 관리 작업을 수행할 수 있도록 허용
                p.Operation.CREATE_AGENT,
                p.Operation.READ_AGENT,
                p.Operation.UPDATE_AGENT,
                p.Operation.DELETE_AGENT,
                p.Operation.CREATE_CUSTOMER,
                p.Operation.READ_CUSTOMER,
                p.Operation.UPDATE_CUSTOMER,
                p.Operation.DELETE_CUSTOMER,
                p.Operation.CREATE_CUSTOMER_SESSION,
                p.Operation.LIST_SESSIONS,
                p.Operation.UPDATE_SESSION,
                p.Operation.DELETE_SESSION,
                # M2M 통합에 필요한 다른 작업 추가
            }

            if operation in m2m_operations:
                return True

        # 다른 모든 경우, 부모 ProductionAuthorizationPolicy에 위임
        return await super().check_permission(request, operation)
```

## 속도 제한 커스터마이제이션 옵션

`ProductionAuthorizationPolicy`는 속도 제한 동작을 커스터마이징하는 여러 방법을 제공합니다:

### 1. 기본 속도 제한기 재정의 (권장)

가장 일반적인 접근 방식은 자체 `BasicRateLimiter` 구성으로 `self.default_limiter`를 재정의하는 것입니다. **BasicRateLimiter 제한은 IP 주소당 적용된다는 점에 유의하세요** - 따라서 `RateLimitItemPerMinute(100)`을 구성하면 IP 주소당 분당 100개의 요청을 의미합니다.

```python
from limits import RateLimitItemPerMinute, RateLimitItemPerHour
from limits.storage import RedisStorage
from limits.strategies import SlidingWindowCounterRateLimiter

# Redis 스토리지 및 맞춤형 제한이 있는 예제
class CustomAuthorizationPolicy(p.ProductionAuthorizationPolicy):
    def __init__(self, ...):
        super().__init__()

        # ...

        self.default_limiter = p.BasicRateLimiter(
            rate_limit_item_per_operation={
                # 대부분의 작업에 기본 속도 제한 사용
                **self.default_limiter.rate_limit_item_per_operation,
                # 맞춤형 제한으로 특정 작업 재정의
                p.Operation.READ_SESSION: RateLimitItemPerMinute(200),
                p.Operation.LIST_EVENTS: RateLimitItemPerMinute(1000),
            },
            # 맞춤형 스토리지 백엔드 사용 (예: Redis)
            storage=RedisStorage("redis://localhost:6379"),
            # 맞춤형 윈도우 전략 사용
            limiter_type=SlidingWindowCounterRateLimiter,
        )
```

`BasicRateLimiter`는 `limits` 라이브러리를 사용하며 다음을 지원합니다:
- **속도 제한 항목**: `RateLimitItemPerMinute(n)`, `RateLimitItemPerSecond(n)`, `RateLimitItemPerHour(n)`
- **스토리지 옵션**: `RedisStorage()`, `MemoryStorage()`, 그리고 limits 라이브러리의 다른 옵션들
- **제한기 전략**: `MovingWindowRateLimiter`, `FixedWindowRateLimiter`, `SlidingWindowCounterRateLimiter`

완전한 제어를 위해, 추상 `RateLimiter` 클래스를 서브클래스화하여 자체 `RateLimiter`를 처음부터 구현하고 `self.default_limiter`에 할당할 수 있습니다.

### 2. 특정 작업에 대한 맞춤형 제한기 함수

`self.specific_limiters`를 사용하여 특정 작업에 대한 맞춤형 속도 제한 함수를 제공합니다. 이는 요청과 작업을 받아 속도가 제한 내에 있는지를 나타내는 boolean을 반환하는 함수입니다.

```python
class CustomAuthorizationPolicy(p.ProductionAuthorizationPolicy):
    def __init__(self, ...):
        super().__init__()

        # ...

        self.specific_limiters[p.Operation.DELETE_AGENT] = self._custom_delete_limiter

    async def _custom_delete_limiter(
        self,
        request: fastapi.Request,
        operation: p.Operation
    ) -> bool:
        # 여기에 맞춤형 로직 구현
        ...
```

권한 확인과 속도 제한 모두에 대한 완전한 제어가 필요한 경우, 추상 `AuthorizationPolicy`를 직접 서브클래스화하고 모든 메서드를 처음부터 구현할 수도 있습니다. 이는 완전한 유연성을 제공하지만 더 많은 구현 작업이 필요합니다. 위에 표시된 접근 방식은 `ProductionAuthorizationPolicy`의 강력한 기반 위에 구축되므로 대부분의 사용 사례에 권장됩니다.

## 맞춤형 인증 정책 통합

### configure_container 사용

Parlant 에이전트와 맞춤형 인증 정책 및 속도 제한기를 통합합니다:

```python
async def configure_container(
    container: p.Container
) -> p.Container:
    container[p.AuthorizationPolicy] = CustomAuthorizationPolicy(
        secret_key="your-jwt-secret-key",
        algorithm="HS256",
    )

    return container
```
```python
async def main():
    # 맞춤형 인증으로 Parlant 서버 생성
    async with p.Server(
        configure_container=configure_container,
    ) as server:
        # 에이전트 로직 여기
        await server.serve()

if __name__ == "__main__":
    asyncio.run(main())
```
