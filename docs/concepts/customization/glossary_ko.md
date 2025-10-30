# Glossary

용어집은 에이전트의 도메인 이해를 형성하는 데 기본적인 부분입니다. 이는 에이전트의 전문 사전과 같습니다: 귀하의 비즈니스 또는 서비스 맥락에 특정한 용어 집합입니다.

### 용어집을 사용해야 하는 경우
특정 작업을 처리하는 에이전트를 만들 때, 종종 도메인의 고유한 어휘를 이해해야 합니다. 예를 들어, 에이전트가 Boogie Nights 호텔의 객실 예약을 돕는 경우, "Boogie Nights"가 귀하의 맥락에서 무엇을 의미하는지 알아야 합니다—이 경우, 단순한 영화 제목이 아니라 귀하의 호텔 이름입니다.

#### 용어집 용어 생성

새로운 용어집 용어를 생성하는 방법은 다음과 같습니다:

```python
await agent.create_term(
    name=TERM,
    description=DESCRIPTION,
    synonyms=[SYNONYM_1, SYNONYM_2, ...],
)
```


### 용어의 구조
각 용어집 항목은 세 가지 구성 요소로 이루어집니다:

> * **Term:** 정의되는 단어 또는 구
> * **Description:** 이 용어가 귀하의 특정 맥락에서 의미하는 것
> * **Synonyms:** 사용자가 이 용어를 지칭할 수 있는 대체 방법

예를 들어:
```python
await agent.create_term(
    name="Boogie Nights",
    description="Our luxury beachfront hotel located in Miami",
    synonyms=["BN Hotel", "The Boogie", "Boogie Hotel"],
)
```

### 에이전트가 용어집을 사용하는 방법
용어집은 에이전트 상호작용에서 두 가지 중요한 목적을 수행합니다.

첫째, 에이전트가 상호작용할 때 고객을 더 잘 이해하도록 도와줍니다. 손님이 _"The Boogie에 머물고 싶어요"_라고 말하면, 에이전트는 그들이 귀하의 호텔을 지칭하고 있다는 것을 압니다.

둘째, 에이전트가 가이드라인을 올바르게 해석하도록 도와줍니다. 다음 구성을 고려하세요:

```python
await agent.create_guideline(
    condition="the user asks about Ocean View rooms",
    action="explain the Sunrise Package benefits",
)

await agent.create_term(
    name="Ocean View",
    description="Our premium rooms on floors 15-20 facing the Atlantic",
    synonyms=["seaside rooms", "beach view"],
)

await agent.create_term(
    name="Sunrise Package",
    description="Complimentary breakfast and early check-in for Ocean View bookings",
    synonyms=["morning special", "sunrise special"],
)
```

여기서 조건과 액션 모두 에이전트가 이러한 용어가 무엇을 의미하는지 이해하는 것에 의존합니다.

고객이 들어와서 다음과 같이 질문하면,

> **Customer:** 대서양을 바라보는 방이 있다고 들었어요. 그게 뭔가요?

에이전트는 용어집 용어를 기반으로 _"사용자가 Ocean View 객실에 대해 질문함"_ 조건이 충족되었음을 이해할 수 있으며, 그런 다음 _"Sunrise Package 혜택 설명"_ 액션으로 응답할 수 있습니다.

## 용어집 vs 가이드라인 vs 에이전트 설명
각 구성 요소는 에이전트의 행동을 형성하는 데 별개의 목적을 수행합니다:

1. 용어집은 에이전트에게 "사물이 무엇인지"를 가르칩니다. 예를 들어, _"Club Member는 5회 이상 숙박한 손님입니다."_ 원하는 만큼 많은 용어를 가질 수 있습니다.
1. 가이드라인은 에이전트에게 "상황에서 어떻게 행동하는지"를 가르칩니다. 예를 들어, _"Club Members와 대화할 때 충성도 상태를 인정하세요."_ 원하는 만큼 많은 가이드라인을 가질 수 있습니다.
1. 에이전트 설명은 전반적인 맥락과 성격을 제공합니다. 예를 들어, _"당신은 Boogie Nights를 위한 유용한 호텔 예약 도우미입니다."_ 에이전트의 설명은 정적이고 제한적입니다.

이렇게 생각하세요: 용어집은 에이전트의 어휘를 구축하고, 가이드라인은 행동을 형성하며, 에이전트 설명은 전반적인 맥락, 역할, 성격 및 톤을 설정합니다.

## 용어집 vs 툴
용어집 용어와 툴 모두 에이전트가 도메인을 이해하도록 돕지만, 근본적으로 다른 목적을 수행합니다. 용어집은 정적 지식을 제공하고, 툴은 동적 데이터 액세스를 가능하게 합니다.

호텔 예약 시나리오를 고려하세요:

**용어집 용어:**
> * **Term:** Club Member
> * **Description:** 5회 이상 숙박한 손님
> * **Synonyms:** loyal guest, regular guest

**툴:**
`check_member_status(user_id)  # 현재 숙박 횟수와 혜택 반환`

용어집 용어는 Club Member가 무엇인지에 대한 일관된 정의를 제공하는 반면, 툴은 데이터베이스에서 특정 사용자의 실제 상태를 확인할 수 있습니다. 마찬가지로:

**용어집 용어:**
> * **Term:** Ocean View Room
> * **Description:** 대서양을 향한 15-20층의 프리미엄 객실
> * **Synonyms:** seaside room, beach view

**툴:**
`check_room_availability(room_type, dates)  # 현재 가용성과 요금 반환`

용어집은 에이전트가 Ocean View Room이 무엇인지 이해하도록 돕고, 툴은 특정 객실의 가용성과 가격에 대한 실시간 정보를 제공합니다.

정적 지식(용어집)과 동적 데이터 액세스(툴) 간의 이러한 분리는 일반 문의와 특정 데이터 기반 상호작용을 모두 처리할 수 있는 명확하고 유지 관리 가능한 에이전트 구현을 만드는 데 도움이 됩니다.
