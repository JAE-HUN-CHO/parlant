# 설치

![Parlant Logo](https://parlant.io/logo/logo-full.svg)

**Parlant**는 LLM 에이전트를 위한 오픈소스 **Agentic Behavior Modeling Engine**으로, 개발자가 통제, 명확성 및 자신감을 가지고 고객 참여형 비즈니스 정렬 대화형 에이전트를 빠르게 만들 수 있도록 돕기 위해 구축되었습니다.

비즈니스가 요구하는 대로 정확하게 동작하는 고객 대면 에이전트를 구축하는 데 필요한 모든 구조를 제공합니다:

- **[여정](https://parlant.io/docs/concepts/customization/journeys)**:
  명확한 고객 여정와 각 단계에서 에이전트가 어떻게 응답해야 하는지 정의합니다.

- **[행동 가이드라인](https://parlant.io/docs/concepts/customization/guidelines)**:
  에이전트 동작을 쉽게 만들 수 있습니다. Parlant가 맥락적으로 관련 요소를 매칭합니다.

- **[도구 사용](https://parlant.io/docs/concepts/customization/tools)**:
  외부 API, 데이터 페처 또는 백엔드 서비스를 특정 상호작용 이벤트에 연결합니다.

- **[도메인 적응](https://parlant.io/docs/concepts/customization/glossary)**:
  에이전트에게 도메인별 용어를 가르치고 개인화된 응답을 만듭니다.

- **[정형 응답](https://parlant.io/docs/concepts/customization/canned-responses)**:
  응답 템플릿을 사용하여 환각을 제거하고 스타일 일관성을 보장합니다.

- **[설명 가능성](https://parlant.io/docs/advanced/explainability)**:
  각 가이드라인이 왜 그리고 언제 매칭되고 준수되었는지 이해합니다.

## 설치
Parlant는 [GitHub](https://github.com/emcie-co/parlant)와 [PyPI](https://pypi.org/project/parlant/) 모두에서 사용 가능하며 여러 플랫폼(Windows, Mac 및 Linux)에서 작동합니다.

Parlant가 제대로 실행되려면 [Python 3.10](https://www.python.org/downloads/release/python-3105/) 이상이 필요합니다.

```bash
pip install parlant
```

모험심이 있고 새로운 기능을 시도하고 싶다면, GitHub에서 직접 최신 개발 버전을 설치할 수도 있습니다.

```bash
pip install git+https://github.com/emcie-co/parlant@develop
```

## 첫 번째 에이전트 만들기

설치 후, 다음 코드를 사용하여 초기 샘플 에이전트를 시작할 수 있습니다. 나중에 동작을 구체화할 것입니다.

```python
# main.py

import asyncio
import parlant.sdk as p

async def main():
  async with p.Server() as server:
    agent = await server.create_agent(
        name="Otto Carmen",
        description="You work at a car dealership",
    )

asyncio.run(main())
```

Parlant가 `async`와 `await`를 사용한 비동기 프로그래밍 패러다임을 따른다는 것을 알 수 있습니다. 이것은 Python의 강력한 기능으로, 한 번에 많은 작업을 처리할 수 있는 코드를 작성할 수 있게 하여 프로덕션에서 에이전트가 더 많은 동시 요청을 처리할 수 있도록 합니다.

비동기 프로그래밍이 처음이라면 빠른 소개를 위해 [공식 Python 문서](https://docs.python.org/3/library/asyncio.html)를 확인하세요.

Parlant는 OpenAI를 기본 NLP 제공업체로 사용하므로 환경에 `OPENAI_API_KEY`가 설정되어 있는지 확인해야 합니다.

그런 다음 프로그램을 실행하세요!
```bash
export OPENAI_API_KEY="<YOUR_API_KEY>"
python main.py
```

Parlant는 기본적으로 여러 LLM 제공업체를 지원하며, `p.NLPServices` 클래스를 통해 액세스할 수 있습니다. `p.NLPService` 인터페이스를 구현하여 자체 제공업체를 추가할 수도 있습니다. 이를 수행하는 방법은 [Custom NLP Models](https://parlant.io/docs/advanced/custom-llms) 섹션에서 배울 수 있습니다.

내장 제공업체 중 하나를 사용하려면 서버를 생성할 때 지정할 수 있습니다. 예를 들어:

```python
async with p.Server(nlp_service=p.NLPServices.cerebras) as server:
  ...
```

일부 제공업체의 경우 추가 "extra" 패키지를 설치해야 할 수 있습니다. 예를 들어 Cerebras NLP 서비스를 사용하려면:

```bash
pip install parlant[cerebras]
```

그렇긴 하지만, Parlant는 [OpenAI](https://openai.com) 및 [Anthropic](https://www.anthropic.com) 모델과 가장 잘 작동하는 것으로 관찰됩니다. 이러한 모델은 유효한 JSON 스키마로 고품질 완성을 생성하는 데 매우 일관성이 있기 때문입니다. 따라서 처음 시작하는 경우 이 중 하나를 사용하는 것을 권장합니다.

## 에이전트 테스트

설치를 테스트하려면 [http://localhost:8800](http://localhost:8800)으로 이동하여 에이전트와 새 세션을 시작하세요.

![Post installation demo](https://parlant.io/img/post-installation-demo.gif)

## 첫 번째 가이드라인 만들기

가이드라인은 Parlant의 행동 모델의 핵심입니다. 특정 사용자 입력이나 조건에 에이전트가 어떻게 응답해야 하는지 정의할 수 있습니다. Parlant는 가이드라인 맥락을 영리하게 관리하므로 맥락 과부하나 기타 규모 문제에 대해 걱정하지 않고 필요한 만큼 많은 가이드라인을 추가할 수 있습니다.

```python
# main.py

import asyncio
import parlant.sdk as p

async def main():
  async with p.Server() as server:
    agent = await server.create_agent(
        name="Otto Carmen",
        description="You work at a car dealership",
    )

    ##############################
    ##    Add the following:    ##
    ##############################
    await agent.create_guideline(
        # This is when the guideline will be triggered
        condition="the customer greets you",
        # This is what the guideline instructs the agent to do
        action="offer a refreshing drink",
    )

asyncio.run(main())
```

이제 프로그램을 다시 실행하세요:
```bash
python main.py
```

[http://localhost:8800](http://localhost:8800)을 새로고침하고 새 세션을 시작한 다음 에이전트에게 인사하세요. 음료를 제안받을 것으로 예상할 수 있습니다!

## 공식 React 위젯 사용

프론트엔드 프로젝트가 React로 구축된 경우, 시작하는 가장 빠르고 쉬운 방법은 공식 Parlant React 위젯을 사용하여 서버와 통합하는 것입니다.

다음은 시작하기 위한 기본 코드 예제입니다:

```jsx
import React from 'react';
import ParlantChatbox from 'parlant-chat-react';

function App() {
  return (
    <div>
      <h1>My Application</h1>
      <ParlantChatbox
        server="PARLANT_SERVER_URL"
        agentId="AGENT_ID"
      />
    </div>
  );
}

export default App;
```

더 많은 문서와 사용자 정의를 보려면 **GitHub 저장소**를 참조하세요: https://github.com/emcie-co/parlant-chat-react.

```bash
npm install parlant-chat-react
```

## Client SDK 설치

Parlant 서버와 상호작용하는 사용자 정의 프론트엔드 앱을 만들려면 네이티브 클라이언트 SDK를 설치하는 것을 권장합니다. 현재 Python 및 TypeScript(JavaScript에서도 작동)를 지원합니다.

#### Python
```bash
pip install parlant-client
```

#### TypeScript/JavaScript
```bash
npm install parlant-client
```

사용자 정의 프론트엔드 통합에 대한 튜토리얼은 여기에서 검토할 수 있습니다: [Custom Frontend Integration](https://parlant.io/docs/production/custom-frontend).

다른 언어의 경우—곧 제공될 예정입니다! 그 동안 [REST API](https://parlant.io/docs/api/create-agent)를 직접 사용할 수 있습니다.
