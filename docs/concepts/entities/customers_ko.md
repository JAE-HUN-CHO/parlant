# Customers

Parlant에서 **customer**는 agent와 상호작용하는 모든 사람을 나타내는 코드 단어입니다. 실제 관계의 성격과 관계없이 사용됩니다. 다시 말해서, customer는 실제 사람, 봇 또는 심지어 인간 agent일 수도 있습니다.

Agent는 익명으로 작동할 수 있지만(누구와 대화하는지 모름), Parlant를 사용하면 등록된 customer를 추적하고 그들의 신원과 선호도를 기반으로 깊이 개인화된 경험을 제공할 수 있습니다.

Agent가 누구와 대화하는지 이해할 수 있게 함으로써 다양한 사용자 세그먼트에 대한 상호작용을 맞춤화할 수 있습니다: 주요 고객은 프리미엄 제안을 받을 수 있고, 신규 사용자는 집중적인 온보딩 지침을 받을 수 있으며, 기타 등등...

Parlant는 customer 등록을 간단하게 만들며, 최소한의 식별만 필요합니다. 이름만 있으면 시작할 수 있습니다.

```python
import parlant.sdk as p

async with p.Server() as server:
    # Register a new customer
    customer = await server.create_customer(name="Alice")
```

## 인증

Parlant는 백엔드 서비스로 존재하는 것을 목표로 하며, 인증 및 권한 부여를 애플리케이션 계층에 맡깁니다. 즉, customer를 등록할 수는 있지만 필요에 맞는 방식(예: OAuth, JWT 등)으로 애플리케이션 코드에서 인증을 처리해야 합니다.

Customer를 식별한 후에는 agent에게 그들의 ID를 전달하여 등록된 customer를 기반으로 상호작용을 개인화할 수 있습니다.

## 저장소

Customer를 저장할 위치를 선택할 수 있습니다.

기본적으로 Parlant는 customer를 유지하지 않으므로 메모리에 저장되고 서버가 재시작되면 손실됩니다. 이것은 테스트 및 개발 목적에 유용합니다.

Customer를 유지하려면 선택한 데이터베이스를 사용하도록 Parlant를 구성할 수 있습니다. 로컬 지속성을 위해 통합 JSON 파일 저장소를 사용하는 것이 좋습니다. 설정이 전혀 필요하지 않기 때문입니다. 프로덕션 사용을 위해서는 기본 제공되는 MongoDB 또는 다른 데이터베이스를 사용할 수 있습니다.

### 로컬 저장소에 유지

이것은 customer를 `$PARLANT_HOME/customers.json`에 저장합니다.

```python
import asyncio
import parlant.sdk as p

async def main():
    async with p.Server(customer_store="local") as server:
        # ...

asyncio.run(main())
```

### MongoDB에 유지

서버를 시작할 때 MongoDB 데이터베이스에 대한 연결 문자열을 지정하기만 하면 됩니다:

```python
import asyncio
import parlant.sdk as p

async def main():
    async with p.Server(customer_store="mongodb://path.to.your.host:27017") as server:
        # ...

asyncio.run(main())
```

## Customer 그룹

**태그**를 사용하여 customer를 다른 그룹으로 나누고 그룹별 개인화를 제어할 수도 있습니다.

예를 들어, VIP customer를 나타내는 태그를 만들 수 있습니다:

```python
# Create a new tag to represent VIP customers
vip_tag = await server.create_tag(name="VIP")

# Register a new customer
customer = await server.create_customer(name="Alice", tags=[vip_tag.id])
```

> **팁: 더 알아보기**
> 특정 customer 및 그룹에 대한 고급 개인화 가능성에 대해 더 알아보려면 [variables](https://parlant.io/docs/concepts/customization/variables) 섹션을 확인하세요.

## 메타데이터 추가

Customer에 사용자 정의 메타데이터를 첨부할 수도 있으며, 이는 그들에 대한 추가 정보를 저장하는 데 사용할 수 있습니다. 이 메타데이터는 상호작용을 더욱 개인화하거나 tool 호출에 대한 컨텍스트를 제공하는 데 사용할 수 있습니다.

```python
customer = await server.create_customer(name="Alice", metadata={
    "external_id": "12345",
    "location": "USA",
})
```

```python
@p.tool
async def get_customer_location(context: p.ToolContext) -> p.ToolResult:
    server = p.ToolContextAccessor(context).server

    if customer := await server.find_customer(id=context.customer_id):
        return p.ToolResult(customer.metadata.get("location", "Unknown location"))

    return p.ToolResult("Customer not found")
```

## Customer 등록

SDK 자체를 사용하여 customer를 등록할 수 있지만, 애플리케이션 계층을 통해 customer 등록을 처리하는 것이 더 실용적인 경우가 많습니다. 이를 통해 customer 관리를 기존 사용자 인증 및 권한 부여 시스템과 통합할 수 있습니다.

Parlant의 REST API 또는 기본 Client SDK를 사용하여 customer를 생성하고 관리할 수 있습니다.

```python
from parlant.client import ParlantClient

# Change localhost to your server's address
client = ParlantClient("http://localhost:8800")

client.customers.create(
    name="Alice",
    metadata={
        "external_id": "12345",
        "location": "USA",
        "hobby": "reading",
    },
    tags=[TAG_ID]  # Optional: specify tag IDs to assign to the customer
)
```

## Customer 데이터 업데이트

이름, 메타데이터 및 태그를 포함하여 언제든지 customer 데이터를 업데이트할 수 있습니다. 이것은 애플리케이션이 진화함에 따라 customer 정보를 최신 상태로 유지하는 데 유용합니다.

```python
client.customers.update(
    customer_id=CUSTOMER_ID,
    name="Alice Smith",
    metadata={
        "set": {
            "location": "Canada",
        },
        "remove": ["hobby"],
    },
    tags=[NEW_TAG_ID]  # Optional: specify new tag IDs to assign to the customer
)
```
