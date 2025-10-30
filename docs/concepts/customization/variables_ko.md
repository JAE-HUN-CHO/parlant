# Variables

모든 고객은 고유하며, 에이전트는 적절한 경우 그들을 그렇게 대우하도록 선택할 수 있습니다.

Variables는 에이전트가 대화하는 고객에 대해 보는 맥락을 풍부하게 합니다. 이는 사려 깊은 고객 서비스 담당자가 각 고객에 대한 중요한 세부 정보를 알고 기억하여 그들이 경청받고 이해받는다고 느끼도록 하는 것처럼, 에이전트가 서비스를 개인화하는 데 도움이 되는 정보에 대한 인식을 제공하기 위한 것입니다.

고객이 에이전트와 상호작용할 때, 그들의 변수가 자동으로 맥락에 로드되어 에이전트가 특정 상황에 따라 응답을 맞춤화할 수 있습니다.

### 실제 애플리케이션
변수가 고객 상호작용을 어떻게 변환하는지 살펴보겠습니다. SaaS 플랫폼의 지원 에이전트를 운영한다고 상상해보세요. 구독 플랜, 마지막 로그인 날짜, 각 고객이 사용하는 기능, 회사 규모와 같은 변수를 추적할 수 있습니다.

데이터 내보내기에 대해 문의하는 두 명의 다른 고객을 고려하세요. Sarah는 무료 플랜을 사용하는 스타트업 창업자로, "내 데이터를 Excel로 내보낼 수 있나요?"라고 묻습니다. 에이전트는 그녀의 무료 플랜 상태를 인식하고 사려 깊게 응답할 수 있습니다: "Excel 내보내기는 프리미엄 기능이지만, 기본 CSV 내보내기를 사용하는 방법을 보여드릴 수 있습니다. 프리미엄 플랜의 고급 보고 기능에 대해서도 알아보시겠습니까?"

이제 엔터프라이즈 계정의 Tom이 같은 질문으로 연락한다고 상상해보세요. 에이전트는 그의 엔터프라이즈 상태를 보지만, 그의 팀이 많은 고급 기능을 탐색하지 않았다는 것도 알아차립니다. 다음과 같이 응답할 수 있습니다: "Excel 내보내기를 도와드리겠습니다! 귀하의 팀이 아직 자동화된 보고 제품군을 사용해보지 않은 것으로 보입니다—이것은 엔터프라이즈 플랜에 포함되어 있으며 매주 몇 시간을 절약할 수 있습니다. 두 기능을 모두 보여드릴까요?"


### Variables로 작업하기

단일 변수는 특정 정보 조각을 식별합니다. 예를 들어, `"subscription_plan"`이라는 변수를 만들 수 있습니다.

그런 다음 각 고객은 해당 변수 아래에 고유한 값을 할당받을 수 있습니다. 예를 들어, Tom은 변수의 값으로 `"enterprise"`를 가질 수 있습니다.

변수 값을 설정하는 두 가지 방법이 있습니다.

1. 값을 수동으로 설정
2. 동적 데이터를 기반으로 값을 자동으로 검색하는 툴에 연결

#### 수동 설정 변수 생성

```python
variable = await agent.create_variable(
    name=NAME,
    description=DESCRIPTION,
)
```

#### 툴 활성화 변수 생성
툴과 연결된 변수는 툴의 출력을 기반으로 값을 자동으로 업데이트합니다. 이는 자주 변경되고 에이전트가 항상 인식해야 하는 동적 데이터에 유용합니다.

다음 툴이 있다고 가정합니다:
```python
@p.tool
async def get_variable_value(context: p.ToolContext) -> p.ToolResult:
  ...
```

그러면 다음과 같이 이 툴을 기반으로 자동 업데이트 변수를 생성할 수 있습니다:

```python
variable = await agent.create_variable(
    name=NAME,
    description=DESCRIPTION,
    tool=get_variable_value,
)
```

기본적으로 연결된 툴은 모든 에이전트 응답 전에 변수의 값을 업데이트합니다. 이러한 빈번한 데이터 재로드가 불필요한 경우, _freshness rules_를 지정하여 값의 새로 고침 간격을 제어할 수 있습니다(연결된 툴이 새 값을 생성하기 위해 호출되는 빈도 제어).

```python
variable = await agent.create_variable(
    name=NAME,
    description=DESCRIPTION,
    tool=get_variable_value,
    freshness_rules=CRON_EXPRESSION,
)
```

Freshness rules는 [cron expression](https://en.wikipedia.org/wiki/Cron) 구문을 따릅니다. Cron이 처음이라면 [crontab generator](https://crontab.cronhub.io/)와 같은 도구를 사용하여 기간 구문을 더 쉽게 정의할 수 있습니다.

> **수동 값**
>
> 툴 활성화 변수도 필요한 경우 값을 수동으로 설정할 수 있습니다. 이는 특정 고객 또는 고객 그룹에 대해 툴의 출력을 재정의하려는 경우에 유용합니다.

#### 고객에 대한 변수 값 설정

```python
await variable.set_value_for_customer(
    customer=CUSTOMER,
    value=VALUE,
)
```

#### 고객 그룹에 대한 변수 값 설정
그룹의 태그를 지정하여 [고객 그룹](https://parlant.io/docs/concepts/entities/customers#customer-groups)에 대한 변수 값을 설정할 수도 있습니다.

```python
await variable.set_value_for_tag(
    tag=TAG_ID,
    value=VALUE,
)
```


## 실습 예제
구독 플랜 변수를 구현하는 방법은 다음과 같습니다.

```python
@p.tool
async def get_subscription_plan(context: p.ToolContext) -> p.ToolResult:
    # 데이터베이스에서 고객의 구독 플랜 가져오기
    return p.ToolResult(await get_plan_from_database(context.customer_id))
```

```python
variable = await agent.create_variable(
    name="subscription_plan",
    description="The customer's subscription plan",
    tool=get_subscription_plan,
)

await variable.set_value_for_customer(
    customer=p.Customer.guest(),
    value="Free Plan",  # 미등록 고객을 위한 기본값
)
```


### 가이드라인과 맥락 변수 결합

가이드라인과 맥락 변수가 함께 작동하여 진정으로 지능적인 상호작용을 생성하는 방법을 살펴보겠습니다. 고객이 다른 계정 등급과 거래 패턴을 가진 디지털 은행용 AI 지원 에이전트를 운영한다고 상상해보세요.

다음은 집중된 가이드라인입니다:

```python
await agent.create_guideline(
    condition="the customer's account_tier is 'basic' "
      "AND they ask about instant international transfers",
    action="highlight our same-day domestic transfers that are free on their plan, "
      "then mention how the premium tier enables instant global payments with lower fees",
)
```

Basic 등급인 Mark가 해외에서 공부하는 딸에게 돈을 보내는 것에 대해 물어보면, 평범한 "그것은 프리미엄 전용입니다" 응답 대신 다음을 듣습니다: "오늘 표준 국제 송금을 사용하여 그 돈을 보낼 수 있도록 도와드릴 수 있습니다. 그런데 프리미엄 계정은 더 낮은 수수료로 즉시 처리됩니다—더 알아보시겠습니까?"
