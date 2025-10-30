# 강제 및 설명 가능성

Parlant가 대화 모델을 일관되게 강제하고 에이전트의 상황 인식 및 의사결정 프로세스에 대한 가시성을 제공하는 방법을 살펴봅시다.

이 섹션에서 배울 내용:

1. 주의 추론 쿼리(Attentive Reasoning Queries, ARQs)가 대화 모델을 어떻게 강제하는지
2. ARQ 아티팩트를 사용하여 동작을 문제 해결하고 개선하는 방법

### 런타임 강제 이해하기

메시지 생성 중에 Parlant는 [주의 추론 쿼리](https://arxiv.org/abs/2503.03669#:~:text=We%20present%20Attentive%20Reasoning%20Queries%20%28ARQs%29%2C%20a%20novel,in%20Large%20Language%20Models%20through%20domain-specialized%20reasoning%20blueprints.)를 사용한 프롬프팅 방법을 통해 실시간 대화에서 지침이 일관되게 준수되는지 확인합니다. Parlant는 간단히 지침을 프롬프트에 추가하고 최선을 기원하는 것이 아니라, LLM이 지침을 준수할 수 있고 준수할 가능성을 극대화하기 위한 명시적 기법을 사용합니다.

주의 추론 쿼리(ARQs)는 본질적으로 LLM에 의사 결정이나 문제 해결 시 특정 사고 패턴을 따르도록 안내하는 프롬프트에 내장된 구조화된 추론 청사진입니다. AI 에이전트가 자연적으로 모든 중요한 요소를 고려하기를 바라는 대신, ARQ는 다양한 도메인(예: 고객 서비스)에 대한 추론 단계를 명시적으로 설명합니다. 전문화된 정신 체크리스트를 따르는 것과 같습니다.

ARQ가 행동 강제에 효과적인 이유는 그들이 그렇지 않으면 간과될 수 있는 중요한 고려사항에 주의를 강제하기 때문입니다. 모델은 미리 정해진 추론 단계(컨텍스트 평가, 솔루션 탐색, 비판, 의사결정 형성)를 거쳐야 하므로, 행동하기 전에 일관되게 중요한 제약 조건을 평가합니다.

![ARQs](https://arxiv.org/html/2503.03669v1/x1.png)

**그림:** ARQ의 일러스트레이션 ([연구 논문](https://arxiv.org/abs/2503.03669#:~:text=We%20present%20Attentive%20Reasoning%20Queries%20%28ARQs%29%2C%20a%20novel,in%20Large%20Language%20Models%20through%20domain-specialized%20reasoning%20blueprints.)에서 가져옴)

정확도와 지침 준수를 증가시키는 것 외에도, 이 프로세스는 부작용으로 원하는 행동과의 일치를 유지하는 데 도움이 되는 투명하고 감사 가능한 추론 경로를 만듭니다.

ARQ는 맥락과 위험 수준에 따라 조정할 수 있을 만큼 유연하며, 특정 도메인이나 규제 요구사항에 맞춘 추론 청사진을 사용합니다. 이 더 신중한 사고 프로세스에는 약간의 계산 오버헤드가 있지만, 신중하게 설계된 ARQ는 Chain-of-Thought 추론보다 정확도와 지연 시간 모두에서 우수합니다.

Parlant는 각 컴포넌트(예: 지침 매칭, 도구 호출 또는 메시지 작성)에 대해 다양한 ARQ 세트를 사용하며, 평가 중인 특정 엔티티(특정 지침, 도구 또는 대화 컨텍스트)에 맞춰 ARQ를 동적으로 특화합니다.

다음은 `GuidelineMatcher`의 로그에서의 예시입니다:

```json
{
  "guideline_id": "fl00LGUyZX",
  "condition": "the customer wants to return an item",
  "condition_application_rationale": "The customer explicitly stated that they need to return a sweater that doesn't fit, indicating a desire to return an item.",
  "condition_applies": true,
  "action": "get the order number and item name and them help them return it",
  "action_application_rationale": [
    {
      "action_segment": "Get the order number and item name",
      "rationale": "I've yet to get the order number and item name from the customer."
    },
    {
      "action_segment": "Help them return it",
      "rationale": "I've yet to offer to help the customer in returning the item."
    }
  ],
  "applies_score": 9
}
```

### 에이전트 행동 설명 및 문제 해결

Parlant의 메시지 생성은 상당한 품질 보증을 거칩니다. 위에서 언급했듯이, ARQ는 에이전트가 상황과 지침을 어떻게 해석했는지를 설명하는 데 도움이 되는 아티팩트를 생성합니다.

문제가 발생하면, 이러한 아티팩트를 검사하여 에이전트가 응답한 이유와 의도를 올바르게 해석했는지 더 잘 이해할 수 있습니다.

시간이 지나면서, 이 피드백 루프는 더 정확하고 효과적인 지침 세트를 구축하는 데 도움이 됩니다.

![Parlant의 설명 가능성](https://parlant.io/img/explainability.gif)
