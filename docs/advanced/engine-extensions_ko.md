# 엔진 확장

외부 프레임워크를 사용하면서 자신의 필요에 맞게 조정하는 것이 항상 쉬운 일은 아닙니다. 특히 원본 설계가 예상하지 못한 방식으로 작동해야 할 때 더욱 그렇습니다.
프레임워크의 소스 코드를 직접 수정하는 것은 위험한 길입니다. 더 깊은 전문 지식이 필요할 뿐만 아니라, 로컬에서 수정한 버전과 업스트림 업데이트 간의 불일치로 이어지기 때문입니다.

그렇다면 미리 만들어진 프레임워크를 다르게 작동시키려면 어떻게 해야 할까요? 핵심은 프레임워크의 기본 가정을 깨뜨리지 않으면서 코드 커스터마이제이션을 포함한 시스템이나 소프트웨어를 실행할 수 있도록 하는 것입니다.

[개방-폐쇄 원칙(Open/Closed Principle)](https://en.wikipedia.org/wiki/Open%E2%80%93closed_principle)에 따르면, 소프트웨어는 확장에는 개방되어 있어야 하지만 수정에는 폐쇄되어 있어야 합니다. 이를 통해 소스 코드를 수정하지 않고도 동작을 확장할 수 있습니다. Parlant는 이 원칙을 준수하도록 신중하게 설계되었으며, 프레임워크 구조에 연결하여 극도의 확장 가능성을 달성할 수 있습니다.

확장을 통해 코어 엔진 컴포넌트를 수정하거나 업데이트를 기다리지 않고도 정확히 필요한 것을 구축할 수 있습니다. 이 기회에 [Discord](https://discord.gg/duxWqxKk6J) 커뮤니티에 참여하여 질문을 할 수 있다는 것을 상기시켜 드립니다.

## 엔진 훅(Engine Hooks)

에이전트가 고객에게 응답해야 할 때마다, 엔진은 응답을 생성하기 위해 일련의 단계를 거칩니다. 훅 함수를 등록하여 이 단계들에 연결하고 엔진의 동작을 수정할 수 있습니다.

다음은 간단한 예입니다:
1. 고객이 에이전트에게만 인사했다는 것을 감지하면 전체 엔진의 응답 생성 프로세스를 재정의합니다.
2. 커스텀 체커를 사용하여 메시지를 고객에게 보내기 전에 규정 준수 위반 여부를 점검합니다.

```python
import asyncio
from typing import Any
import parlant.sdk as p

async def intercept_message_generation_with_greeting(
    ctx: p.LoadedContext, payload: Any, exc: Exception | None
) -> p.EngineHookResult:
    if await is_only_greeting(ctx.interaction.last_customer_message):
        await ctx.session_event_emitter.emit_message_event(
            trace_id=ctx.tracer.trace_id,
            data="Hello! How can I help you today?",
        )
        return p.EngineHookResult.BAIL  # 나머지 프로세스 중단
    else:
        return p.EngineHookResult.CALL_NEXT  # 정상 프로세스 계속

async def check_message_compliance(
    ctx: p.LoadedContext, payload: Any, exc: Exception | None
) -> p.EngineHookResult:
    generated_message = payload

    if not await is_compliant(generated_message):
        ctx.logger.warning(f"Prevented sending a non-compliant message: '{generated_message}'.")
        return p.EngineHookResult.BAIL  # 이 메시지 전송 안 함

    return p.EngineHookResult.CALL_NEXT  # 정상 프로세스 계속

async def configure_hooks(hooks: p.EngineHooks) -> p.EngineHooks:
    hooks.on_acknowledged.append(intercept_message_generation_with_greeting)
    hooks.on_message_generated.append(check_message_compliance)

    return hooks

async def main():
    async with p.Server(
        configure_hooks=configure_hooks,
    ) as server:
        # 여기에 로직을 작성합니다
        ...
```

## 의존성 주입(Dependency Injection)

엔진을 소스 코드 수정 없이 확장하기 위해, Parlant는 [의존성 주입](https://en.wikipedia.org/wiki/Dependency_injection) 시스템을 사용합니다. 이를 통해 다양한 컴포넌트나 전체 처리 엔진(예: 특정 사용 사례에 맞춰 전체 파이프라인을 최적화하려는 경우)의 자신만의 구현을 주입할 수 있습니다.

단순함을 위해 몇 가지 기본적인 확장 메커니즘과 일반적인 확장 사용 사례를 살펴보겠습니다.

그러나 여기에 다루지 않은 사항에 도움이 필요하다면, [Discord](https://discord.gg/duxWqxKk6J), [GitHub Discussions](https://github.com/emcie-co/parlant/discussions), 또는 [Contact Page](https://parlant.io/contact)를 통해 연락해 주시기 바랍니다.

### 컨테이너 작업하기

Parlant의 의존성 주입 컨테이너를 사용하는 방법을 살펴봅시다. 컨테이너는 모든 컴포넌트가 등록되는 중앙 위치이며, 이를 사용하여 자신의 컴포넌트를 검색하거나 등록할 수 있습니다.

컨테이너와 관련하여 두 가지를 할 수 있습니다:

1. **자신의 컴포넌트 등록**: 다양한 컴포넌트의 자신의 구현을 컨테이너에 추가하여 애플리케이션 전체에서 주입 가능하게 만들 수 있습니다.
2. **기존 컴포넌트의 동작 조정**: 컨테이너에서 컴포넌트 인스턴스를 검색하여 자신의 코드에서 사용할 수 있습니다.

#### 컴포넌트 등록

컴포넌트를 등록하면 Parlant의 작동 방식의 거의 모든 측면을 재정의할 수 있습니다. 서버에 `configure_container` 훅을 전달하여 등록 단계 중에 컨테이너에 접근할 수 있습니다.

이 훅은 컨테이너의 기초 상태를 받아들이고 서버가 시작되기 전에 수정된 버전을 반환합니다.

```python
import asyncio
import parlant.sdk as p

async def configure_container(container: p.Container) -> p.Container:
    # 여기에 자신의 컴포넌트를 등록합니다
    # ...
    return container

async def main():
    async with p.Server(
        configure_container=configure_container,
    ) as server:
        # 여기에 로직을 작성합니다
        ...
```

#### 기존 컴포넌트 조정

기본 제공 컴포넌트의 동작을 조정하려면, 컨테이너에서 검색하여 동작을 수정할 수 있습니다. 이는 전체 컴포넌트를 교체하지 않고 기존 기능을 디버깅하거나 확장할 때 유용합니다.

이 훅은 `initialize_container`라고 불리며, 모든 클래스가 등록되고 결정된 후이지만 서버가 실제로 사용하기 전에 컨테이너 내의 컴포넌트를 수정할 수 있습니다.

이 훅은 컨테이너의 최종 상태를 받아들이고 `None`을 반환합니다. 컨테이너는 등록된 컴포넌트를 *접근*하기 위해서만 제공됩니다.

```python
import asyncio
import parlant.sdk as p

async def initialize_container(container: p.Container) -> None:
    # 여기에 자신의 컴포넌트를 등록합니다
    # ...
    return container

async def main():
    async with p.Server(
        configure_container=configure_container,
    ) as server:
        # 여기에 로직을 작성합니다
        ...
```

## 확장에 개방(Open for Extension)

Parlant 코드를 읽거나 디버깅할 때, 엔진 내에서 많은 다양한 유형의 컴포넌트를 만날 것입니다. 구성 및 초기화 훅을 사용하면, 이제 이들을 어떻게 접근하고 필요에 따라 확장, 수정 또는 완전히 재정의할 수 있는지 알게 됩니다.

#### 확장의 일반적인 사용 사례

1. 캔 응답의 불일치 동작 재정의. 이는 실제로 여기에 문서화되어 있습니다: [Canned Responses](https://parlant.io/docs/concepts/customization/canned-responses#no-match-responses).
2. 로깅, 모니터링 또는 기타 관심사를 추가하기 위해 모든 엔진 컴포넌트를 래핑합니다.
3. 특정 지침을 평가하는 방법 재정의. 예를 들어, 지침이 간단하고 충분한 데이터가 있다면, LLM을 거치는 대신 커스텀 학습된 BERT 모델로 평가할 수 있습니다.
4. 전체 메시지 생성 컴포넌트 재정의, Parlant의 지침 매칭과 도구 실행을 활용하면서 자신의 메시지 생성 로직을 사용합니다.

그러나 할 수 있는 것이 훨씬 더 많습니다. 엔진은 유연하고 확장 가능하도록 설계되었으므로, 핵심 코드베이스를 수정하지 않고도 특정 필요에 맞게 적응할 수 있습니다.
