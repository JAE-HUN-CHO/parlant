# Human Handoff

Human handoff는 특히 AI 에이전트를 사용할 때 고객 서비스 자동화의 중요한 측면입니다. 필요할 때 자동화된 응답에서 인간 전문가로 원활하게 전환할 수 있습니다. 이 가이드는 Parlant에서 human handoff를 구현하는 프로세스를 안내합니다.

### 콜 센터 모델: 계층 구조 이해

현대 콜 센터는 효율성과 비용 최적화를 위해 설계된 계층화된 시스템으로 운영됩니다. **Tier 1** 담당자는 대부분의 통화를 처리합니다—일반적으로 고객 문의의 80%—계정 질문, 기본 문제 해결, 표준 요청과 같은 일반적인 문제를 다룹니다. 이러한 담당자는 자주 묻는 질문과 표준 절차에 대해 교육을 받습니다.

**Tier 2** 담당자는 더 경험이 많으며 전문 지식, 확대된 문제 또는 미묘한 문제 해결이 필요한 복잡한 사례를 처리합니다. Tier 2 에이전트가 더 숙련되고 유능하지만, 고용하고 유지하는 데 비용도 훨씬 더 많이 듭니다.

## Parlant의 접근 방식

Parlant의 생성 AI와 깊이 작업한 경험을 바탕으로 한 우리의 믿음은, 오늘날의 AI 에이전트가 대부분의 **Tier 1 작업**을 효과적으로 자동화할 수 있으며, 잠재적으로 고객 서비스 인력의 80%를 줄이면서 다음을 통해 문의를 처리할 수 있다는 것입니다:

- 전문적인 커뮤니케이션 표준
- 높은 효율성과 24/7 가용성
- 우아한 대화 관리
- 비즈니스 규칙에 대한 완전한 준수

Parlant의 임무에는 궁극적으로 Tier 2 사용 사례를 자동화하는 것이 포함되지만, 가장 복잡한 시나리오에 대해서는 기술이 아직 거기에 도달하지 못했다고 정직하게 믿습니다. 그러나 **Tier 1을 자동화하는 것이 가장 중요한 비용 절감과 효율성 개선을 달성할 수 있는 곳입니다**.

## 통합된 Human Handoff

Tier 1 요청을 자동화하고 있으므로, human handoff(본질적으로 Tier 2로)를 통합된 방식으로 지원하는 것이 합리적입니다. Parlant를 사용하면 HubSpot, Zendesk 또는 맞춤형 지원 플랫폼과 같이 사용하는 외부 시스템과 원활하게 통합할 수 있습니다.

Parlant에서 human handoff를 구현하는 방법은 다음과 같습니다:

## 세션을 수동 모드로 설정

Human handoff의 첫 번째 단계는 AI 에이전트가 새 메시지에 자동으로 응답하지 않도록 중지하는 것입니다. 올바른 조건이 충족될 때마다(예: AI 에이전트가 고객을 적절히 도울 수 없는 경우) 툴을 사용하여 세션을 수동 모드로 설정하여 이를 달성할 수 있습니다:

### 툴을 사용하여 수동 모드 트리거

```python
@p.tool
async def initiate_human_handoff(context: p.ToolContext, reason: str) -> p.ToolResult:
    """AI가 고객을 적절히 도울 수 없을 때 인간 에이전트로 핸드오프를 시작합니다."""

    # 세션을 수동 모드로 설정하여 자동 AI 응답 중지
    return p.ToolResult(
        data=f"Human handoff initiated because: {reason}",
        control={
            "mode": "manual"  # 자동 에이전트 응답을 중지합니다
        }
    )
```
```python
# 툴을 가이드라인과 연결
await agent.create_guideline(
    condition="Customer requests human assistance",
    action="Initiate human handoff and explain the transition professionally",
    tools=[initiate_human_handoff]
)
```

### 세션을 수동으로 인계받기
Parlant 클라이언트 SDK를 사용하여 외부 시스템에서 세션을 수동으로 수동 모드로 설정할 수도 있습니다:

```typescript
// Parlant 클라이언트를 사용하여 세션 모드를 수동으로 설정
import { ParlantClient } from 'parlant-client';

const client = new ParlantClient({
    environment: "http://localhost:8800" // Parlant 서버 URL
});

async function setSessionToManual(sessionId: string) {
    await client.sessions.update(sessionId, {
        mode: "manual" // 자동 AI 응답 중지
    });

    console.log(`Session ${sessionId} set to manual mode`);
}
```

## 세션 이벤트 수동 관리

세션이 수동 모드에 있으면, Parlant의 REST API 및 클라이언트 SDK를 사용하여 이벤트를 수동으로 관리할 수 있습니다:

### 인간 운영자 메시지 추가

#### Python

```python
from parlant.client import AsyncParlantClient

client = AsyncParlantClient(base_url="http://localhost:8800")

# 인간 운영자로부터 메시지 추가
async def send_human_message(session_id: str, message: str, operator_name: str):
    event = await client.sessions.create_event(
        session_id=session_id,
        kind="message",
        source="human_agent",  # 인간 운영자로부터의 메시지
        message=message,
        participant={
            "id": OPTIONAL_ID_FOR_EXTERNAL_SYSTEM_REFERENCE,
            "display_name": operator_name
        }
    )
    return event
```

#### TypeScript

```typescript
import { ParlantClient } from 'parlant-client';

const client = new ParlantClient({
    environment: "http://localhost:8800"
});

// 인간 운영자로부터 메시지 추가
async function sendHumanMessage(sessionId: string, message: string, operatorName: string) {
    const event = await client.sessions.createEvent(sessionId, {
        kind: "message",
        source: "human_agent", // 인간 운영자로부터의 메시지
        message: message,
        participant: {
            id: OPTIONAL_ID_FOR_EXTERNAL_SYSTEM_REFERENCE,
            display_name: operatorName
        }
    });

    return event;
}
```

### AI 에이전트를 대신하여 메시지 추가

때로는 인간 운영자가 AI 에이전트로부터 온 것처럼 보이는 메시지를 보내고 싶을 수 있습니다:

#### Python

```python
# AI 에이전트를 대신하여 메시지 전송
async def send_message_as_ai(session_id: str, message: str):
    event = await client.sessions.create_event(
        session_id=session_id,
        kind="message",
        source="human_agent_on_behalf_of_ai_agent",  # 인간이 AI로 전송
        message=message
    )
    return event
```

#### TypeScript

```typescript
// AI 에이전트를 대신하여 메시지 전송
async function sendMessageAsAI(sessionId: string, message: string) {
    const event = await client.sessions.createEvent(sessionId, {
        kind: "message",
        source: "human_agent_on_behalf_of_ai_agent", // 인간이 AI로 전송
        message: message
    });

    return event;
}
```

## Parlant에서 이벤트 받기

외부 시스템과 통합하려면 Parlant 세션에서 새 이벤트를 모니터링해야 합니다:

### 이벤트 폴링 패턴 - Parlant를 단일 진실 공급원으로

적절한 통합의 핵심은 Parlant 세션을 대화 상태에 대한 **단일 진실 공급원**으로 취급하는 것입니다. Parlant에서 이벤트만 읽고 그에 따라 외부 시스템을 업데이트하세요. Parlant에 이벤트를 보낼 때도 외부 시스템 UI에 표시하기 전에 `list_events()`에서 반환될 때까지 기다리세요.

#### Python

```python
async def monitor_session_events(session_id: str, last_offset: int = 0):
    """
    Parlant 세션에서 새 이벤트를 폴링합니다.
    Parlant 세션은 단일 진실 공급원입니다 - 모든 메시지 표시는
    list_events()에서 반환된 이벤트를 기반으로 해야 합니다.
    """

    while True:
        try:
            # 타임아웃과 함께 새 이벤트 대기
            events = await client.sessions.list_events(
                session_id=session_id,
                kinds="message",
                min_offset=last_offset,
                wait_for_data=30  # 최대 30초 동안 새 이벤트 대기
            )

            for event in events:
                # Parlant의 모든 메시지 이벤트 처리
                await process_event_for_display(event)
                last_offset = max(last_offset, event.offset + 1)

        except Exception as e:
            # list_events()에서 타임아웃 후 다시 시도
            continue

async def process_event_for_display(event):
    """
    외부 시스템에 표시하기 위해 Parlant의 들어오는 이벤트를 처리합니다.
    """
    # 외부 시스템 채팅 UI에 모든 메시지 이벤트 표시
    await update_external_chat_display(
        message=event.data.get('message'),
        source=event.source,
        participant_name=event.data.get('participant', {}).get('display_name', 'Unknown'),
        timestamp=event.created_at,
        event_id=event.id
    )

async def update_external_chat_display(message: str, source: str, participant_name: str,
                                     timestamp: str, event_id: str):
    """외부 시스템(HubSpot, Zendesk 등)의 채팅 UI 업데이트"""

    # Parlant 소스를 UI 표시 로직에 매핑
    if source == "customer":
        await add_customer_message_to_ui(message, timestamp, event_id)
    elif source == "ai_agent":
        await add_ai_message_to_ui(message, timestamp, event_id)
    elif source == "human_agent":
        await add_human_agent_message_to_ui(message, participant_name, timestamp, event_id)
    elif source == "human_agent_on_behalf_of_ai_agent":
        # AI 메시지로 표시하되 인간이 보낸 것으로 추적
        await add_ai_message_to_ui(message, timestamp, event_id, sent_by_human=True)
```

#### TypeScript

```typescript
async function monitorSessionEvents(sessionId: string, lastOffset: number = 0): Promise<void> {
    // Parlant 세션은 대화 상태의 단일 진실 공급원입니다
    while (true) {
        try {
            // Parlant에서 새 이벤트 폴링
            const events = await client.sessions.listEvents(sessionId, {
                minOffset: lastOffset,
                kinds: "message",
                waitForData: 30 // 최대 30초 대기
            });

            for (const event of events) {
                // 표시를 위해 모든 메시지 이벤트 처리
                await processEventForDisplay(event);
                lastOffset = Math.max(lastOffset, event.offset + 1);
            }

        } catch (error) {
            // listEvents()에서 타임아웃 후 다시 시도
        }
    }
}

async function processEventForDisplay(event: any): Promise<void> {
    /**
     * 외부 시스템에 표시하기 위해 Parlant의 이벤트를 처리합니다.
     */
    await updateExternalChatDisplay({
        message: event.data.message,
        source: event.source,
        participantName: event.data.participant?.display_name,
        timestamp: event.createdAt,
        eventId: event.id
    });
}

interface DisplayMessageParams {
    message: string;
    source: string;
    participantName: string;
    timestamp: string;
    eventId: string;
}

async function updateExternalChatDisplay(params: DisplayMessageParams): Promise<void> {
    /**
     * Parlant가 권위 있는 대화 상태로 표시하는 것을 기반으로
     * 외부 시스템(HubSpot, Zendesk 등)의 채팅 UI를 업데이트합니다.
     */
    const { message, source, participantName, timestamp, eventId } = params;

    switch (source) {
        case "customer":
            await addCustomerMessageToUI(message, timestamp, eventId);
            break;
        case "ai_agent":
            await addAIMessageToUI(message, timestamp, eventId);
            break;
        case "human_agent":
            await addHumanAgentMessageToUI(message, participantName, timestamp, eventId);
            break;
        case "human_agent_on_behalf_of_ai_agent":
            // AI 메시지로 표시하되 인간이 보낸 것으로 추적
            await addAIMessageToUI(message, timestamp, eventId, true);
            break;
    }
}

async function addCustomerMessageToUI(message: string, timestamp: string, eventId: string): Promise<void> {
    // 고객 메시지에 대한 UI 업데이트 로직 구현
}

async function addAIMessageToUI(message: string, timestamp: string, eventId: string, sentByHuman: boolean = false): Promise<void> {
    // AI 메시지에 대한 UI 업데이트 로직 구현
}

async function addHumanAgentMessageToUI(message: string, agentName: string, timestamp: string, eventId: string): Promise<void> {
    // 인간 에이전트 메시지에 대한 UI 업데이트 로직 구현
}
```

## Human Handoff 모범 사례

1. **명확한 전환 메시지**: 고객이 인간 에이전트로 전환될 때 항상 알리고 이유를 설명하세요. 가이드라인을 사용하여 이를 달성할 수 있습니다.

2. **맥락 보존**: 인간 에이전트가 AI 상호작용의 전체 대화 기록에 액세스할 수 있도록 보장하세요. 세션의 이벤트를 외부 시스템과 동기화하여 이를 수행하세요.

3. **원활한 경험**: 단일 에이전트 경험의 환상을 유지하고 싶을 때 `human_agent_on_behalf_of_ai_agent` 소스를 사용하세요. 고객은 항상 인간과 상호작용하고 있다는 것을 알 필요가 없습니다.

4. **모니터링 및 분석**: handoff 비율, 이유 및 해결 결과를 추적하여 시간이 지남에 따라 AI 에이전트를 개선하세요. 인간 상호작용에서 학습한 교훈을 구현하여 에이전트 응답과 가이드라인을 개선하세요.

5. **AI로 복귀**: 적절할 때 세션을 자동 모드로 되돌리는 로직을 구현하는 것을 고려하세요.


이러한 통합 접근 방식을 통해 효율적인 Tier 1 지원을 위해 AI 에이전트를 활용하는 동시에 복잡한 문제를 인간 전문가에게 원활하게 확대할 수 있어 고객 서비스에 두 세계의 장점을 제공합니다.
