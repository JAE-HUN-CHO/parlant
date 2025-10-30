# Tools

Parlant는 가이던스 시스템과 긴밀하게 통합된 안내형 툴 사용 접근 방식을 제공합니다.

Parlant의 툴 호출 접근 방식은 처음부터 구축되었습니다. 대부분의 LLM API([MCP](https://modelcontextprotocol.io/introduction) 포함)에서 익숙할 수 있는 툴 호출 메커니즘보다 더 포괄적일 것입니다. 이는 가이던스 기반 행동 제어와의 깊고 원활한 통합을 가능하게 하기 위해 구축되었기 때문입니다.

### 툴 사용 이해하기
Parlant에서 툴은 항상 특정 가이드라인과 연결됩니다. 툴은 연결된 가이드라인이 대화에 매칭될 때만 실행됩니다. 이 설계는 명확한 의도의 체인을 생성합니다: 가이드라인이 툴이 언제 왜 사용되는지를 결정하며, LLM의 판단에 맡기지 않습니다(이는 오류가 많습니다).

Parlant에서 **비즈니스 로직**(툴에 인코딩됨)은 **프레젠테이션**(사용자 인터페이스) 관심사, 또는 가이드라인에 의해 제어되는 고객 대면 행동과 의식적으로 분리됩니다.

이를 통해 개발자는 코드에서 API 로직을 완전히 제어하면서 작업할 수 있으며, 이러한 툴을 에이전트의 "툴 창고"에 제공합니다. 그런 다음 비즈니스 전문가는 기본 코드에 관여할 필요 없이 이러한 툴이 언제 어떻게 사용되는지를 결정하는 자연어 가이드라인을 독립적으로 정의할 수 있습니다. 이러한 관심사의 분리는 Parlant의 핵심 설계 원칙으로, 더 깨끗하고 유지 관리가 쉬운 시스템을 가능하게 합니다.

#### 대화형 UI vs. 그래픽 UI
비유하자면, Guidelines와 Tools를 그래픽 UI 프레임워크의 위젯과 이벤트 핸들러처럼 생각할 수 있습니다. GUI 버튼에는 `onClick` 이벤트가 있으며, 이를 일부 API 함수와 연결하여 _"이 버튼을 클릭하면 이 함수를 실행해라"_ 라고 말할 수 있습니다. 마찬가지로 Parlant(본질적으로 대화형 UI 프레임워크)에서 Guideline은 버튼과 같고, Tool은 API 함수와 같으며, 연결은 둘을 연결하여(이벤트 핸들러 등록처럼) _"이 가이드라인이 적용되면 이 툴을 실행해라"_ 라고 말합니다.

이러한 개념을 설명하는 구체적인 예는 다음과 같습니다:
> * **Condition:** 사용자가 서비스 기능에 대해 질문할 때
> * **Action:** 그들의 의도를 이해하고 문서를 참조하여 답변하기
> * **Tools Associations:** `[query_docs(user_query)]`

여기서 문서 쿼리 툴은 가이드라인이 이 사용자 상호작용에 대해 문서를 참조해야 한다고 지시한 후에만 실행됩니다.


## 툴 작성하기

툴을 작성하려면 `ToolContext`를 첫 번째 인수로 받고, 그 다음에 툴에 전달하려는 다른 매개변수를 받는 함수를 정의해야 합니다. 함수는 `ToolResult`를 반환해야 합니다.

다음은 Parlant 에이전트의 모든 툴의 기본 구조입니다:

```python
import parlant.sdk as p

@p.tool
async def tool_name(context: p.ToolContext, param1: str, param2: int) -> p.ToolResult:
  """여러 줄의 툴 설명.
  에이전트가 읽을 수 있으며 이 툴을 실행할지 여부를 결정하는 데 도움이 됩니다.
  """
  ...
```

더 구체적으로 설명하기 위해, 다음은 에이전트의 맥락에 제품을 가져오는 간단한 툴의 예입니다:

```python
import parlant.sdk as p

@p.tool
async def find_products(context: p.ToolContext, query: str) -> p.ToolResult:
    """자연어 검색 쿼리를 기반으로 제품을 가져옵니다."""

    # 데이터베이스 또는 API에서 잔액을 가져오는 시뮬레이션
    products = await MY_DB.get_products(query=query)

    return p.ToolResult(balance)
```

#### 선택적 매개변수
`typing` 모듈의 `Optional` 타입을 사용하여 툴에서 선택적 매개변수를 정의할 수도 있습니다. 이를 통해 해당 매개변수에 값을 제공하지 않고도 툴을 호출할 수 있습니다.

```python
from typing import Optional
import parlant.sdk as p

@p.tool
async def find_products(
  context: p.ToolContext,
  query: str,
  limit: Optional[int]
) -> p.ToolResult:
    default_limit = 10
    products = await MY_DB.get_products(query=query, limit=limit or default_limit)

    return p.ToolResult(products)
```

## 에이전트에 툴 연결하기
에이전트가 툴을 실행할 수 있도록 하려면, 툴이 평가되고 호출될 수 있는 조건을 지정해야 합니다. 위에서 언급했듯이, 이 접근 방식은 LLM에서 흔한 문제인 거짓 양성 툴 호출을 제거하는 데 도움이 됩니다.

```python
@p.tool
async def my_tool(context: p.ToolContext) -> p.ToolResult:
  ...

await agent.create_guideline(
    condition=CONDITION,
    action=ACTION,
    tools=[my_tool],
)
```

툴 자체가 액션을 암시하는 경우, Parlant가 툴의 설명에서 이를 파악하도록 하여 수동으로 지정하는 것을 건너뛸 수 있습니다. 다음과 같이 보입니다:

```python
await agent.attach_tool(condition=CONDITION, tool=my_tool)
```

## Tool Result
`ToolResult`는 툴 호출의 결과를 캡슐화하는 특수 객체입니다.

사용할 수 있는 다섯 가지 속성이 있습니다. 대부분의 경우 `data` 속성만 사용하지만, 다른 속성들도 흥미로운 사용 사례를 가능하게 하므로 알아둘 가치가 있습니다: `data`, `metadata`, `control`, `canned_responses`, `canned_response_fields`.

> **Tool Result 수명**
>
> 많은 범용 에이전트 프레임워크와 달리, Parlant는 대화형 에이전트 구축에 특별히 그리고 의도적으로 최적화되어 있습니다. 따라서 그 아키텍처는 대화형 애플리케이션에 대한 툴 호출의 기본 동작을 최적화합니다.
>
> 이를 위해 툴 결과는 기본적으로 세션에 저장되므로 전체 상호작용 세션 동안 에이전트 참조에 사용할 수 있습니다. 이는 후속 가이드라인 매칭과 툴 호출이 자동으로 이전 툴의 결과에 의해 정보를 받는다는 것을 의미합니다. 이는 계정 잔액, 항목 ID 또는 기타 제품 정보와 같이 나중에 참조해야 하는 결과에 유용합니다.
>
> 예를 들어, 제품 이름과 ID를 반환하는 툴을 호출하고, 고객이 특정 제품을 선택하여 응답한 후 `product_id` 매개변수를 받는 다른 툴을 호출하는 경우, 에이전트는 이전 툴의 결과를 사용하여 맥락에서 해당 매개변수를 자동으로 채울 수 있습니다. 예:
>
> ```python
> @p.tool
> async def get_products(context: p.ToolContext, query: str) -> p.ToolResult:
>     products = await MY_DB.get_products(query=query)
>     return p.ToolResult(data=products)
>
> @p.tool
> async def get_product_details(context: p.ToolContext, product_id: str) -> p.ToolResult:
>     # 세션에서 이전 툴 호출이 이미 수행된 경우
>     # 에이전트는 올바른 `product_id`를 매개변수화할 수 있습니다.
>     product = await MY_DB.get_product(product_id=product_id)
>     return p.ToolResult(data=product)
> ```


### Tool Result 속성
이러한 각 속성이 무엇에 사용되고 어떻게 사용하는지 살펴보겠습니다:


#### Data
`data` 속성에는 툴의 주요 출력이 포함됩니다. 이는 문자열, 목록 또는 딕셔너리와 같은 JSON 직렬화 가능한 모든 타입이 될 수 있습니다.

이것은 `ToolResult`의 _항상_ 필수인 유일한 속성입니다. 에이전트가 상호작용 이벤트의 기록을 이해하는 데 사용하는 유일한 속성이기 때문입니다. 즉, `data` 속성에 아무것도 반환하지 않으면 에이전트에게 결과에 대해 알려지지 않으며 상호작용을 탐색하는 데 사용할 수 없습니다.

```python
# 예제 1
return p.ToolResult(data="This is the result of my tool call")

# 예제 2
return p.ToolResult(data={"appointments": [
  { "id": "123", "date": "2023-10-01 10:00" },
  { "id": "456", "date": "2023-10-02 11:00" },
]})
```

#### Metadata
`metadata` 속성은 툴 호출에 대한 추가 정보를 저장하는 데 사용할 수 있는 선택적 딕셔너리입니다.

에이전트는 이 메타데이터를 전혀 인식하지 못하지만, REST API 클라이언트를 사용하여 응답에서 가져올 수 있습니다. 이는 프론트엔드에서 가치를 추가할 수 있는 응답에 대한 추가 정보를 보내는 데 유용합니다.

여기서 고전적인 사용 사례는 프론트엔드에 정보 소스를 표시하는 데 사용할 수 있는 RAG 정보 소스(예: URL, 문서 ID 등)를 반환하는 것입니다. 또 다른 사용 사례는 프론트엔드에 표시할 수 있는 생성된 차트나 기타 시각화에 대한 이미지 링크를 반환하는 것입니다.

```python
return p.ToolResult(
  data=ANSWER,
  metadata={ "sources": [{"url": s.url, "title": s.title} for s in ANSWER_SOURCES]},
)
```

```python
return p.ToolResult(
  data="The profit margin is 20%",
  metadata={ "generated_chart_url": "https://example.com/chart.png" },
)
```

> **수명에 유의하세요**
>
> 메타데이터는 주로 세션 이벤트에 액세스할 때 유용하므로, 일반적으로 `lifespan: "session"`(기본값)과 함께 사용하는 것이 합리적입니다. `lifespan: "response"`를 사용하면 메타데이터가 세션 이벤트에서 사용할 수 없으므로 프론트엔드에서 액세스할 수 없습니다.

#### Control
`control` 속성을 사용하면 에이전트와 엔진에 대한 제어 지시문을 지정할 수 있습니다.

현재 사용할 수 있는 두 가지 제어 지시문이 있습니다:
- `"mode": p.SessionMode`: 이를 통해 세션을 수동 모드로 전환할 수 있으며, 이는 에이전트가 자동으로 응답을 생성하지 않음을 의미합니다. 이는 에이전트의 자동 응답을 일시 중지하고 인간 운영자가 인계받도록 하려는 인간 핸드오프 시나리오에 특히 유용합니다.
- `"lifespan": p.Lifespan`: `ToolResult`가 얼마나 오래 살아있어야 하는지를 제어합니다. 두 가지 옵션이 있습니다:
  - `"session"`: 결과가 저장되고 전체 세션 동안 에이전트에 제공됩니다. 이것이 기본값입니다.
  - `"response"`: 결과는 현재 응답에만 사용할 수 있습니다. 이는 오류 보고 또는 매우 일시적인 정보 제공과 같이 현재 응답을 넘어서 필요하지 않은 임시 결과에 유용합니다.

```python
return p.ToolResult(
  data="Transferring to a human agent",
  # 이 툴 결과가 반환되면, 세션이 API 호출을 통해
  # 자동 모드로 다시 전환될 때까지 에이전트는 더 이상 응답을 생성하지 않습니다.
  control={ "mode": "manual" },
)
```

```python
return p.ToolResult(
  data="Encountered an error while fetching data",
  # 이 툴 결과는 세션에 저장되지 않습니다.
  # 에이전트는 현재 응답 동안만 이를 인식합니다.
  control={ "lifespan": "response" },
)
```

#### Canned Responses
툴은 또한 고려할 완전한 canned responses와 canned response 렌더링 중에 대체될 필드를 반환할 수 있습니다. Canned response 속성에 대한 자세한 내용은 [Canned Responses](https://parlant.io/docs/concepts/customization/canned-responses) 섹션을 참조하세요.

## 툴 결과에 따른 가이드라인 재평가

경우에 따라 툴 결과가 어떤 가이드라인이 관련성을 갖게 되는지에 영향을 줄 수 있습니다. 다음은 예입니다:

송금을 처리하는 은행 에이전트를 생각해보세요. 사용자가 송금을 요청하면, 사용자가 송금을 원함이라는 조건을 가진 가이드라인이 `get_user_account_balance()` 툴을 활성화하여 사용 가능한 자금을 확인합니다. 이 툴은 현재 잔액을 반환하며, 이는 반환 값에 따라 추가 가이드라인 매칭을 트리거할 수 있습니다.

예를 들어, 잔액이 $500 미만인 경우, 낮은 잔액 가이드라인이 활성화되어 에이전트에게 다음과 같이 말하도록 지시할 수 있습니다: _"현재 잔액이 낮은 것으로 보입니다. 이 송금을 진행하시겠습니까? 이 거래로 인해 초과 인출 수수료가 발생할 위험이 있습니다."_

Parlant에서는 툴 호출 후 재평가를 위해 특정 가이드라인을 표시할 수 있습니다. 이는 툴이 호출되면, 가이드라인 매처가 툴의 결과에 따라 새로운 가이드라인이 활성화되어야 하는지 확인하기 위해 세션을 재평가한다는 것을 의미합니다—툴을 실행한 *후* 응답을 생성하기 *전*에.

다음과 같이 할 수 있습니다:

```python
# 에이전트는 이 툴을 실행한 후 이 가이드라인을 재평가하도록 보장합니다
await guideline.reevaluate_after(my_tool)
```


## 모범 사례

#### 자연어 프로그래밍에 대한 참고사항
LLM은 대화 작업에 뛰어나지만, 복잡한 논리 연산과 다단계 계획에 어려움을 겪습니다. LLM 아키텍처에 대한 최근 연구에 따르면 고급 모델도 일관된 논리적 추론과 순차적 의사 결정에 어려움을 겪습니다. LLM의 "계획 문제"—복잡한 작업을 순서가 있는 단계로 분해하고 결론을 종합하는 것—은 규모에 맞게 일관성이 필요할 때 여전히 해결되지 않은 중요한 문제로 남아 있습니다.

이러한 한계를 감안할 때, Parlant는 실용적인 접근 방식을 취합니다: 코드의 로직을 행동 모델링과 분리합니다. 가이드라인에 비즈니스 로직을 포함하는 대신, Parlant는 대화 행동과 기본 비즈니스 운영 간의 깨끗한 분리를 권장합니다.

툴을 결정론적이고 프로그래밍적인 비즈니스 로직을 위한 장소로, 가이드라인을 대화형 인터페이스 설계로 생각하세요. 이러한 분리는 더 깨끗하고 유지 관리가 쉬우며 더 안정적인 시스템을 만듭니다.

> **Agentic API 설계**
>
> 에이전트의 툴 설계에 대한 모범 사례를 자세히 알아보려면 [Agentic Backends](https://parlant.io/blog/what-no-one-tells-you-about-agentic-api-design)에 대한 블로그 게시물을 읽어보는 것을 권장합니다.

#### 예제

**1. 전자상거래 제품 추천:**

하지 마세요

가이드라인의 복잡한 비즈니스 로직, LLM에 과도하게 의존
> * **Guideline Action:**
> 사용자가 스포츠를 언급하면 구매 기록을 확인하세요.
> 러닝 기어를 구매했다면 프리미엄 신발을 추천하세요.
> 신규 고객이라면 스타터 키트를 제안하세요.
> * **Tool Associations:** `[get_product_catalog]`

하세요

로직은 코드화된 추천 엔진에 있어야 합니다

> * **Guideline Action:** 개인화된 추천 제공
> * **Tool Associations:** `[get_personalized_recommendations]`

**2. 금융 자문:**

하지 마세요

금융 분석 로직이 신뢰할 수 없는 LLM 숫자 이해에 의존
> * **Guideline Action:** 계좌 잔액과 최근 거래를 확인하세요.
>           지출이 평소 패턴의 80%를 초과하면 예산 검토를 제안하세요.
>           투자 수익이 하락하면 포트폴리오 조정을 권장하세요.
> * **Tool Associations:** `[get_account_data]`

하세요

금융 분석 로직은 코드에서 안정적으로 처리됨

> * **Guideline Action:** 개인화된 금융 인사이트 얻기
> * **Tool Associations:** `[get_financial_insights]`


## Tool Context

`ToolContext` 매개변수는 툴에 맥락 정보와 유틸리티를 제공하는 특수 객체입니다.

`ToolContext`에서 사용할 수 있는 가장 유용한 속성과 메서드를 살펴보겠습니다:

1. `agent_id`: 툴을 호출하는 에이전트의 고유 식별자입니다.
2. `customer_id`: 에이전트와 상호작용하는 고객의 고유 식별자입니다.
3. `session_id`: 현재 세션의 고유 식별자입니다.
4. `emit_message(message: str)`: 고객에게 메시지를 보내는 메서드입니다. 이는 장기 실행 툴 호출 중 진행 상황을 보고하는 데 사용할 수 있습니다.
5. `emit_status(status: p.SessionStatus)`: [세션 상태](https://parlant.io/docs/concepts/sessions#status-event)를 업데이트하는 메서드입니다.

#### 서버 객체(Agent, Customer 등) 액세스하기

`ToolContext`에서 서버 객체에도 액세스할 수 있어 에이전트, 가이드라인 및 기타 서버 수준 리소스에 액세스할 수 있습니다.

서버 객체에 액세스하려면 다음과 같이 `p.ToolContextAccessor`를 사용하세요:

```python
import parlant.sdk as p

@p.tool
async def my_tool(context: p.ToolContext) -> p.ToolResult:
    server = p.ToolContextAccessor(context).server

    # 컨텍스트의 agent_id를 사용하여 현재 에이전트에 액세스
    agent = await server.get_agent(id=context.agent_id)

    # 현재 고객에 액세스
    customer = await server.get_customer(id=context.customer_id)

    # ...

    return p.ToolResult(...)
```

#### 보안 데이터 액세스

다른 고객에게 비공개인 데이터를 검색하거나 표시하는 툴을 빌드해야 한다고 가정해보겠습니다.

순진한 접근 방식은 고객에게 자신을 식별하도록 요청하고 이를 올바른 데이터에 대한 액세스 토큰으로 사용하는 것입니다. 그러나 이 접근 방식은 매우 안전하지 않습니다. 사용자를 식별하는 데 LLM에 의존하기 때문입니다. LLM은 잘못될 수 있거나, 더 나쁜 경우 악의적인 사용자에 의해 조작될 수 있습니다.

이를 수행하는 더 나은 방법은 Parlant에 [고객을 등록](https://parlant.io/docs/concepts/entities/customers#registering-customers)하고 툴의 `ToolContext` 매개변수에 포함된 프로그래밍 방식으로 사용 가능한 정보를 사용하는 것입니다.

실제로 다음과 같이 보입니다:

```python
@p.tool
async def get_transactions(context: p.ToolContext) -> p.ToolResult:
    transactions = await DB.get_transactions(context.customer_id)
    return p.ToolResult(transactions)
```

## Tool Insights와 매개변수 옵션

Parlant의 아키텍처는 근본적으로 모듈식이기 때문에, 가이드라인 매칭, 툴 호출 및 메시지 구성과 같은 구성 요소가 독립적으로 작동합니다. 이러한 비단일식 접근 방식은 복잡한 의미 로직을 관리하는 데 많은 이점을 제공하지만, 이러한 구성 요소 간에 맥락 인식을 전달해야 합니다.

**Tool Insights**는 툴 호출과 메시지 구성 사이의 브릿징 구성 요소로, 필수 매개변수가 누락되는 등의 이유로 툴을 호출할 수 없을 때 구성 구성 요소에 정보를 제공합니다.

이를 통해 에이전트가 더 지능적으로 응답할 수 있습니다. 예를 들어, 적절한 툴을 호출할 수 없을 때에 대한 지식이 없다면 오해의 소지가 있는 응답을 생성할 수 있습니다. 그러나 툴 인사이트를 사용하면 에이전트는 누락된 정보를 인식하고 필요한 경우 고객에게 필요한 툴 인수를 자동으로 요청할 수 있습니다.

#### Tool Parameter Options

Tool Insights의 기본 동작을 향상시키기 위해 **ToolParameterOptions**를 사용할 수 있습니다. 이는 툴 매개변수가 처리되고 전달되는 방식을 더 잘 제어할 수 있는 특수 매개변수 주석입니다.

Tool Insights가 에이전트가 툴 호출이 실패하는 시기와 이유를 인식하도록 돕는 반면, **ToolParameterOptions**는 에이전트가 특정 누락된 매개변수를 언제 어떻게 설명할지를 안내하여 한 걸음 더 나아갑니다.

```python
from typing import Annotated
import parlant.sdk as p

@p.tool
async def transfer_money(
  context: p.ToolContext,
  amount: Annotated[float, p.ToolParameterOptions(
    source="customer",  # 고객만 이 값을 제공할 수 있습니다 - 에이전트는 추론할 수 없습니다
  )],
  recipient: Annotated[str, p.ToolParameterOptions(
    source="customer",
  )]
) -> ToolResult:
    # ...
```

**ToolParameterOptions**는 여러 선택적 인수로 구성되며, 각각 에이전트의 매개변수 이해와 적용을 정제합니다:

- `hidden` `True`로 설정하면 이 매개변수는 메시지 구성에 노출되지 않습니다. 즉, 누락된 경우 에이전트가 고객에게 알리지 않습니다. 불투명한 제품 ID 또는 비공개로 유지되어야 하는 기타 정보와 같은 내부 매개변수에 일반적으로 사용됩니다.

- `precedence` 툴에 여러 필수 매개변수가 있는 경우, 고객에게 전달되는 툴 인사이트가 압도적일 수 있습니다(예: 단일 메시지에서 5개의 다른 항목을 요청). 우선순위를 사용하면 그룹(동일한 값을 공유하는)을 생성하여 고객이 선택한 순서대로 한 번에 몇 개(우선순위 값을 공유하는)만 배울 수 있습니다.

- `source` 인수의 소스를 정의합니다. 에이전트가 고객에게 직접 값을 요청해야 합니까("customer"), 아니면 주변 맥락에서 추론해야 합니까("context")?
지정하지 않으면 기본값은 "any"이며, 에이전트가 어디서든 검색할 수 있음을 의미합니다.

- `description` 이는 에이전트가 맥락에서 인수를 추출할 때 매개변수를 올바르게 해석하는 데 도움이 됩니다. 매개변수 이름이 모호하거나 불분명한 경우 이를 채우세요.

- `significance` 이 매개변수가 왜 필요한지에 대한 고객 대면 설명입니다. 이는 고객이 제공해야 하는 정보와 그 이유를 이해하고 관련시키는 데 도움이 됩니다.

- `examples` 인수를 추출해야 하는 방법을 설명하는 샘플 값 목록입니다. 이는 형식을 강제하는 데 유용합니다(예: "YYYY-MM-DD"와 같은 날짜 형식).

- `adapter` 추론된 값을 툴에 전달하기 전에 올바른 타입으로 변환하는 함수입니다. 제공된 경우, 에이전트는 예상 형식과 일치하는지 확인하기 위해 추출된 인수를 이 함수를 통해 실행합니다. 매개변수 타입이 코드베이스의 사용자 정의 타입인 경우 사용하세요.

- `choice_provider` 매개변수의 인수에 대한 유효한 선택지를 제공하는 함수입니다. 이를 사용하여 에이전트가 이 함수가 반환하는 특정 세트에서 동적으로 값을 선택하도록 제한하세요.


## 매개변수 값 제약

툴의 인수가 특정 선택지 세트에 포함되어야 하는 경우, Parlant는 해당 선택지에 따라 툴 호출이 매개변수화되도록 보장하는 데 도움을 줄 수 있습니다. 이를 수행하는 세 가지 방법이 있습니다:

1. 하드코딩된 선택지를 제공할 수 있을 때 enums 사용
1. 선택지가 동적일 때(예: 고객별) `choice_provider` 사용
1. 매개변수가 더 복잡한 구조를 따를 때 Pydantic 모델 사용

#### Enum 매개변수
미리 알려진 고정된 선택지 세트를 `Enum` 클래스를 사용하여 지정합니다.

```python
import enum

class ProductCategory(enum.Enum):
  LAPTOPS = "laptops"
  PERIPHERALS = "peripherals"
  MONITORS = "monitors"

@p.tool
async def get_products(
  context: p.ToolContext,
  category: ProductCategory,
) -> p.ToolResult:
  # 여기에 코드 작성
  return p.ToolResult(returned_data)
```

#### Choice Provider
현재 실행 맥락을 기반으로 동적으로 선택지 세트를 제공합니다.

```python
async def get_last_order_ids(context: p.ToolContext) -> list[str]:
  return await load_last_order_ids_from_db(customer_id=context.customer_id)

@p.tool
async def load_order(
  context: p.ToolContext,
  order_id: Annotated[Optional[str], p.ToolParameterOptions(
    choice_provider=get_last_order_ids,
  )],
) -> p.ToolResult:
  # 여기에 코드 작성
  return p.ToolResult({...})
```

#### Pydantic 모델
Pydantic 모델을 사용하여 검증 및 제약을 포함할 수 있는 매개변수의 복잡한 구조를 정의합니다.
```python
from pydantic import BaseModel

class ProductSearchQuery(BaseModel):
  category: str
  price_range: tuple[float, float]

@p.tool
async def search_products(
  context: p.ToolContext,
  query: ProductSearchQuery,
) -> p.ToolResult:
  # 여기에 코드 작성
  return p.ToolResult(...)
```
