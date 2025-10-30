# Relationships

**가이드라인**과 **journeys**가 서로 어떻게 관련되는지 정의하는 것은 행동 모델링의 강력하고 (고급) 부분입니다.

#### 배경 및 동기

예전에 우리 팀은 다음과 같은 가이드라인(그 외 다른 것들)을 가진 피자 판매 에이전트를 구축하고 있었습니다:
```python
offer_pepsi_instead_of_coke = await agent.create_guideline(
    condition="The customer wants a coke",
    action="Tell them we only have Pepsi",
)

handoff_if_upset = await agent.create_guideline(
    condition="The customer is becoming upset",
    action="Apologize and tell them you will transfer them to a manager",
    tools=[handoff_to_human_manager],
)
```

이것은 처음에는 잘 작동했지만, 다음과 같은 시나리오를 만날 때까지였습니다:

> **Agent:** 주문과 함께 음료를 원하시나요?
>
> **User:** 콜라 주세요
>
> **Agent:** 죄송합니다. 펩시만 있습니다. 대신 그것을 원하시나요?
>
> **User:** 잠깐, 뭐라고요? 저는 펩시를 싫어합니다. 왜 콜라가 없나요?
>
> **Agent:** 이 불편을 드려 죄송합니다. 매니저에게 연결해 드리겠습니다. 그동안 펩시를 드릴까요?
>
> **User:** 저를 놀리고 있는 건가요?

에이전트의 응답은 분명히 우리가 원하는 것이 아니었습니다. 비꼬는 듯 적대적으로 보일 수 있기 때문입니다. 그러나 불쌍한 AI 에이전트는 우리가 설정한 가이드라인의 조건에 따라 우리가 준 가이드라인만 따르고 있었습니다.

그리고 여기에 핵심이 있습니다. 지시를 관리하는 것은 단순히 기술적 도전만이 아니라, 인간 모델링 도전이기도 합니다. 지시를 꽤 문자 그대로 받아들여야 하는 자동 에이전트에게 우리의 지시가 서로 어떻게 관련되는지 고려해야 합니다. 특히 더 미묘한 상황에서 다른 맥락에서 어떻게 관련되어야 하는지는 궁극적으로 우리만이 결정할 수 있는 것입니다.

위의 경우, 두 번째 가이드라인이 첫 번째 가이드라인보다 우선순위가 *높아야* 한다는 것을 보장하고 싶었습니다. 우선순위 관계가 필요했으며, 오늘날 Parlant에서는 다음과 같이 매우 간단하게 표현할 수 있습니다:

```python
await handoff_if_upset.prioritize_over(offer_pepsi_instead_of_coke)
```

## 관계 종류

이러한 관계는 처음에는 복잡하게 들릴 수 있지만, 모델러로서 훨씬 더 많은 권한을 제공하여 일관되게 정확한 응답을 생성하는 데 훨씬 더 유능하게 만듭니다.

이러한 관계를 간략하게 검토하여 그 목적을 이해하는 것을 권장합니다.

#### 관계 유형
다음은 지원되는 관계입니다. 각 관계는 _source_(기호 **S**)와 _target_(기호 **T**) 사이에 있습니다.

관계 유형을 클릭하여 자세히 알아보세요.

- [Entailment](https://parlant.io/docs/guidelines/relationships#entailment): **S**가 활성화되면, **T**도 항상 활성화되어야 함
- [Priority](https://parlant.io/docs/guidelines/relationships#priority): **S**와 **T**가 모두 활성화되면, **S**만 활성화되어야 함
- [Dependency](https://parlant.io/docs/guidelines/relationships#dependency): **S**가 활성화되면, **T**도 활성화되지 않는 한 비활성화
- [Disambiguation](https://parlant.io/docs/guidelines/relationships#disambiguation): **S**가 활성화되고 두 개 이상의 타겟 **T ∈ {T₁, T₂, ...}**가 활성화되면, 고객에게 원하는 액션을 명확히 하도록 요청

### Entailment
> **S**가 활성화되면, **T**도 항상 활성화되어야 함

```python
await source.entail(target)
```

Entailment의 필요성을 이해하려면 먼저 Parlant가 에이전트가 고객에게 무언가를 말하려고 할 때 어떤 가이드라인을 활성화할지 선택하는 방법을 이해해야 합니다.

기본적으로 Parlant는 현재 상태의 세션을 검사하고 다음과 같은 질문을 합니다: "이 가이드라인이 지금 관련이 있는가?", "저 가이드라인이 지금 관련이 있는가?".

이를 위해 주로 가이드라인의 _조건_을 테스트합니다.

이것은 그 자체로는 잘 작동하는 것처럼 보일 수 있지만, 다음 형식의 두 가이드라인을 고려할 때까지입니다:

> * **Guideline A:** X일 때, Y를 수행
> * **Guideline B:** Y일 때, Z를 수행

이제 세션을 보고 _X_가 실제로 적용되지만 _Y_는 적용되지 않는다고 결정하는 상황을 상상해보세요. 위의 순진한 로직으로는 에이전트에게 _Y_를 수행하라는 가이드라인만 제공했을 것입니다.

그러나 이 경우를 뒤로 물러나 분석하면, 에이전트가 막 _Y_를 수행하려고 한다는 것을 알고 있으며, 이는 우리가 설치한 가이드라인에 따라 _Z_도 적용되어야 함을 의미합니다.

이것이 entailment가 달성하는 것입니다: _A_가 활성화될 때마다 _B_도 활성화되도록 요구합니다.



### Priority
> **S**와 **T**가 모두 활성화되면, **S**만 활성화되어야 함

```python
await source.prioritize_over(target)
```

Priority는 여러 사용 사례에 사용될 수 있습니다. 가장 일반적인 두 가지는 다음과 같습니다:
1. 상호 배타적인 가이드라인 생성
1. 대화 내에서 액션의 흐름과 우선순위 제어

#### 우선순위 제어에 관하여
동시에 활성화되는 두 개의 가이드라인이 있을 수 있습니다. 예를 들어:
> * 고객이 거래를 하려고 할 때, 완료될 때까지 프로세스를 안내
> * 고객의 계좌에 $1,000 미만이 있을 때, 저축 계획 제안

예를 들어, 사용자가 거래를 제출하는 과정에 있는 동안 계좌 잔액 세부 정보가 세션에 도입되면 위의 가이드라인이 동시에 활성화되는 것을 발견할 수 있습니다.

저축 계획이 제공되도록 보장하되—좋은 타이밍으로 거래가 완료된 후에만—거래 완료를 저축 계획 제공보다 우선시할 수 있습니다. 거래가 완료되면 저축 관련 가이드라인이 활성화될 수 있습니다.

### Dependency
> **S**가 활성화되면, **T**도 활성화되지 않는 한 비활성화

```python
await source.depend_on(target)
```

Dependency는 다른 기본 조건도 유지되는 경우에만 가이드라인이 활성화되도록 보장하는 데 도움이 됩니다.

가장 일반적인 사용 사례는 더 구체적인 조건이 적절한 기본 맥락에서만 활성화되도록 보장하는 것입니다.

#### 특정 조건 맥락화

흐름을 구축할 때, 흐름 기본 가이드라인에 종속되도록 하여 특수화되거나 엣지 케이스 시나리오를 처리할 수 있습니다. 예:

##### 기본 가이드라인
> 고객이 주문을 반품하려고 할 때, 반품 프로세스를 완료하도록 도움

##### 종속 가이드라인
> * 고객이 주문 번호를 제공할 수 없을 때, 마지막 주문의 항목을 로드하고 그것이 그들의 주문인지 확인하도록 요청
> * 고객이 정확한 주문 번호를 지정했을 때, 해당 주문의 항목을 로드하고 그것이 그들의 주문인지 확인하도록 요청

이러한 가이드라인을 기본 가이드라인에 종속되도록 하면, 평가가 항상 올바른 맥락에서 수행되도록 보장할 수 있습니다.

### Disambiguation
> **S**가 활성화되고 두 개 이상의 타겟 **T ∈ {T₁, T₂, ...}**가 활성화되면, 고객에게 원하는 액션을 명확히 하도록 요청

```python
await source.disambiguate([target_1, target_2, ...])
```

모호성으로 인해 일부 또는 전부가 동시에 활성화되어 지시 따르기 혼란으로 이어지는 두 개(또는 그 이상)의 경쟁 가이드라인 사이의 상황이 있을 수 있습니다.

예를 들어, 고객이 은행 에이전트에게 _"제 한도는 무엇인가요?"_라는 메시지를 보냈고, 다음과 같은 가이드라인이 있었으며, 각각은 엔진의 해석에 따라 낙관적으로 활성화된 경우:

> * 고객이 ATM 한도에 대해 문의할 때, 계좌 프로필에서 데이터 가져오기
> * 고객이 신용카드 한도에 대해 문의할 때, 카드 제공업체에서 가져오기

고객의 의도를 명확히 하기 위해 [관찰 가이드라인](https://parlant.io/docs/guidelines/relationships#observational-guidelines)을 추가하여 두 액션을 명확히 할 수 있습니다:

```python
ambiguous_limits = await agent.create_observation(
    condition="The customer is inquiring about limits but it isn't clear which kind",
)

await ambiguous_limits.disambiguate([fetch_atm_limits, fetch_credit_card_limits])
```

> **정보: 관찰**
>
> `Agent.create_observation()`은 액션 없이 가이드라인을 생성하는 단축키입니다. 이 가이드라인은 여전히 맥락 내에서 매칭되지만, 수행할 액션이 없습니다. 위의 예와 같이 특정 시나리오에서 가이드라인 간의 관계를 생성하는 데 유용합니다.

## 관찰 가이드라인
관계를 사용하여 대화 엣지 케이스를 모델링할 때, 특정 상황이 적용된다는 것을 확립하기 위해(조건을 사용하여) 가이드라인을 추가하고—이러한 경우에만—관계를 사용하여 다른 가이드라인이나 journeys를 활성화하거나 비활성화하고 싶을 수 있습니다.

이를 위해 Parlant는 **관찰 가이드라인**이라는 특수한 유형의 가이드라인을 지원합니다. 이는 액션이 없는 가이드라인이며, 일반적으로 특정 조건이 적용된다는 것을 확립하고 그 주위에 관계를 생성하는 데만 사용됩니다.

```python
observation = await agent.create_observation(condition=CONDITION)
```

그런 다음 이 관찰을 다음과 같은 흥미로운 방식으로 사용할 수 있습니다:

1. 관찰을 다른 가이드라인보다 우선시하여 다른 가이드라인 비활성화.
```python
await observation.prioritize_over(other_guideline)
```

2. 관찰이 활성화된 경우에만 다른 가이드라인이 적용되도록 범위 지정.
```python
await other_guideline.depend_on(observation)
```

그리고 다른 창의적인 사용!
