# Retrievers

실용적인 이유로, Parlant는 두 가지 데이터 접근 모드를 구분합니다. 바로 tools와 **retrievers**입니다.

고객 대면 에이전트를 개발할 때, 데이터를 가져오는 데에는 실질적으로 두 가지 다른 사용 사례가 있습니다:
1. **Tools**: 사용자 요청과 같은 특정 이벤트에 대응하여 특정 서비스에서 데이터를 가져오는 경우
2. **Retrievers**: 현재 대화 상태에 대해 에이전트의 지식을 기반으로 하고, 방향을 설정하고, 정렬하기 위한 맥락적 정보를 가져오는 경우. 이는 전통적으로 RAG(Retrieval-Augmented Generation)라고 불립니다.

경험적으로, 에이전트가 일반적으로 "알고 있을 것으로 기대"되는 데이터에는 **retrievers**를 사용하고, 에이전트가 "로드"하거나 "무언가를 수행해야 하는" 데이터에는 tools를 사용하는 것이 좋습니다.

**Retrievers의 사용 사례는 다음과 같습니다:**
- 자주 묻는 질문에 대한 답변 가져오기
- 현재 대화 맥락을 기반으로 관련 문서나 정보 가져오기
- 에이전트의 응답을 개인화하기 위한 사용자별 데이터 가져오기 ([Variables](https://parlant.io/docs/concepts/customization/variables) 참조)

> **팁: 응답 지연 시간 트레이드오프**
>
> Retrievers는 현재 대화의 맥락 내에서 에이전트의 지식을 기반으로 하는 데에만 사용되기 때문에, 일반적으로 에이전트의 다른 작업(가이드라인 매칭, 툴 호출 등)과 병렬로 실행될 수 있습니다.
>
> 따라서 retrievers를 사용하면 가이드라인 매칭이나 툴 호출의 추가 지연 시간 없이 에이전트의 응답을 기반으로 할 수 있습니다.


## Retriever 생성
Retriever는 `p.RetrieverContext`를 받아 `p.RetrieverResult`를 반환하는 함수입니다. `p.RetrieverContext`에는 현재 대화 맥락이 포함되어 있고, `p.RetrieverResult`에는 retriever가 가져온 데이터가 포함됩니다.

```python
async def my_retriever(context: p.RetrieverContext) -> p.RetrieverResult:
  ...
```

#### 간단한 RAG 예제
다음은 고객의 마지막 메시지를 기반으로 DB에서 문서를 가져오는 간단한 retriever 예제입니다:

```python
async def answer_retriever(context: p.RetrieverContext) -> p.RetrieverResult:
    # 대화에서 마지막 메시지 가져오기
    if last_message := context.interaction.last_customer_message:
        # 임베더를 사용하여 메시지를 벡터로 변환
        message_vector = my_embedder.embed(last_message.content)
        # 메시지 벡터를 기반으로 데이터베이스에서 문서 가져오기
        documents = await fetch_documents_from_db(message_vector)

        return p.RetrieverResult(documents)

    return p.RetrieverResult(None)
```

또는 LLM을 사용하여 전체 상호작용 기록을 기반으로 쿼리를 생성할 수도 있습니다:

```python
async def answer_retriever(context: p.RetrieverContext) -> p.RetrieverResult:
    if context.interaction.messages:
        # 대화의 모든 메시지를 결합하여 깔끔한 맥락 생성
        conversation_text = "\n".join(str(msg) for msg in context.interaction.messages)

        # LLM을 사용하여 대화에서 사용자 쿼리 추출
        if query := await my_llm.extract_user_query_from_conversation(conversation_text):
          # 임베더를 사용하여 쿼리를 벡터로 변환
          message_vector = my_embedder.embed(query)
          # 쿼리 벡터를 기반으로 데이터베이스에서 문서 가져오기
          documents = await fetch_documents_from_db(message_vector)

          return p.RetrieverResult(documents)

    return p.RetrieverResult(None)
```

#### Retriever 연결하기

에이전트가 retriever를 실제로 사용하도록 하려면 다음과 같이 연결해야 합니다:

```python
await agent.attach_retriever(my_retriever)
```

디버깅 및 로깅 목적으로 유용한 retriever의 ID를 지정할 수도 있습니다:

```python
await agent.attach_retriever(my_retriever, id="my_retriever")
```


## Retriever 결과 수명

Retriever 결과의 수명은 현재 응답으로 제한됩니다. 즉, 대화 전체에 걸쳐 지속되지 않습니다. 이는 대화 맥락을 깨끗하고 집중적으로 유지하는 동시에 대화 전반에 걸쳐 평균 입력 토큰을 줄이는 데 도움이 됩니다.

## Retriever Context

Retriever context를 사용하면 더 정교한 retrievers를 구축하는 데 도움이 되는 여러 유용한 속성에 액세스할 수 있습니다:
- `server`: 현재 retriever 요청을 처리하고 있는 서버로, 서버별 리소스나 구성에 액세스하는 데 유용합니다.
- `container`: 현재 사용 중인 의존성 주입 컨테이너로, 컨테이너에 등록된 서비스와 리소스에 액세스할 수 있습니다.
- `logger`: 현재 사용 중인 로거로, retriever 실행 중 디버그 정보나 오류를 로깅하는 데 유용합니다.
- `trace_id`: 에이전트의 현재 응답에 대한 고유 식별자로, 추적 및 디버깅 목적으로 사용할 수 있습니다.
- `interaction`: 대화 기록 및 기타 관련 정보를 포함하는 현재 상호작용입니다.
- `agent`: 현재 상호작용을 처리하고 있는 에이전트입니다.
- `customer`: 현재 에이전트와 상호작용하고 있는 고객입니다.
- `variables`: 현재 상호작용에 대해 설정된 변수들입니다.
