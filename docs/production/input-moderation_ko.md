# 사용자 입력 검열

AI 에이전트에 콘텐츠 필터링을 추가하면 보다 전문적인 수준의 고객 상호작용을 구현할 수 있습니다. 그 중요성에 대해 알아보겠습니다.

### 과제 이해하기

LLM 기반의 AI 에이전트는 통계적 패턴 매칭을 수행하며, 수신하는 입력의 특성에 영향을 받을 수 있습니다. 마치 어떤 대화에 참여해야 하고 참여하지 말아야 하는지에 대한 명확한 경계가 있을 때 도움이 되는 고객 서비스 직원과 같다고 생각하면 됩니다.

#### 민감한 주제

정신 건강이나 불법 활동과 같은 일부 주제는 전문적인 인간의 처리가 필요합니다. 에이전트가 기술적으로는 이러한 주제를 다룰 수 있지만, 실제 사용 사례에서는 이러한 대화를 피하거나 적절한 인적 리소스로 리디렉션하는 것이 더 나은 경우가 많습니다.

#### 괴롭힘으로부터 보호

고객 상호작용은 전문적으로 유지되어야 하지만, 일부 사용자는 에이전트(또는 다른 사람)를 괴롭히거나 학대하려고 시도할 수 있습니다. 이것은 단순히 예의를 유지하는 문제가 아닙니다. LLM은 (인간처럼) 경우에 따라 공격적이거나 부적절한 언어의 영향을 받아 응답에 영향을 미칠 수 있습니다.

이러한 경우를 처리하기 위해 Parlant는 [OpenAI의 Omni Moderation](https://openai.com/index/upgrading-the-moderation-api-with-our-new-multimodal-moderation-model/)과 같은 검열 API와 통합하여 이러한 상호작용이 에이전트에 도달하기 전에 필터링합니다.

### 입력 검열 활성화
검열을 활성화하려면 이벤트를 생성할 때 쿼리 매개변수를 설정하기만 하면 됩니다.

#### Python
```python
from parlant.client import ParlantClient

client = ParlantClient(base_url=SERVER_ADDRESS)

client.sessions.create_event(
    SESSION_ID,
    kind="message",
    source="customer",
    message=MESSAGE,
    moderation="auto",
)
```

#### TypeScript
```typescript
import { ParlantClient } from 'parlant-client';

const client = new ParlantClient({ environment: SERVER_ADDRESS });

await client.sessions.createEvent(SESSION_ID, {
     kind: "message",
     source: "customer",
     message: MESSAGE,
     moderation: "auto",
});
```

고객이 부적절한 메시지를 보내면 Parlant는 해당 콘텐츠가 에이전트에게 보이지 않도록 보장합니다. 대신 에이전트는 고객이 특정 이유(예: 괴롭힘, 불법 행위 등)로 "검열된" 메시지를 보냈다는 것만 볼 수 있습니다.

이것은 가이드라인과 잘 통합됩니다. 예를 들어, 다음과 같은 가이드라인을 설치할 수 있습니다:

> * **조건:** 고객의 마지막 메시지가 검열됨
> * **행동:** 이 문의에 대해 도움을 드릴 수 없다고 알리고, 인간 지원팀에 연락하도록 제안

UX 관점에서 이 접근 방식은 이러한 메시지를 만났을 때 단순히 "오류를 표시"하는 것보다 우수합니다. 오류를 보는 대신, 고객은 정중하고 유익한 응답을 받습니다. 더 나아가, 응답은 다른 상황과 마찬가지로 가이드라인과 도구로 제어될 수 있습니다.

## Jailbreak 보호

에이전트의 가이드라인은 엄격한 보안 조치는 아니지만 (이는 백엔드 권한에 의해 더 강력하게 처리됨), 일부 사용자가 에이전트를 속여 지침을 드러내거나 의도된 경계 밖에서 행동하도록 시도하더라도 제시 가능한 행동을 유지하는 것이 중요합니다.

Parlant의 검열 시스템은 특별한 `paranoid` 모드를 지원하며, [Lakera Guard](https://www.lakera.ai/lakera-guard) ([Gandalf Challenge](https://gandalf.lakera.ai/baseline)의 창시자)와 통합하여 이러한 조작 시도를 방지합니다.

#### Python
```python
from parlant.client import ParlantClient

client = ParlantClient(base_url=SERVER_ADDRESS)

client.sessions.create_event(
    SESSION_ID,
    kind="message",
    source="customer",
    message=MESSAGE,
    moderation="paranoid",
)
```

#### TypeScript
```typescript
import { ParlantClient } from 'parlant-client';

const client = new ParlantClient({ environment: SERVER_ADDRESS });

await client.sessions.createEvent(SESSION_ID, {
     kind: "message",
     source: "customer",
     message: MESSAGE,
     moderation: "paranoid",
});
```

`paranoid` 모드를 활성화하려면 Lakera에서 API 키를 받아 서버를 시작하기 전에 환경 변수 `LAKERA_API_KEY`에 할당해야 합니다.
