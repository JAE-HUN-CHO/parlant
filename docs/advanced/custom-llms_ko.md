# 커스텀 NLP 모델

[엔진 확장](https://parlant.io/docs/advanced/engine-extensions)의 기본을 이해한 후, Parlant에 다른 NLP 모델을 통합할 수 있습니다.

> **커스텀 모델에 대한 참고사항**
>
> Parlant는 기본 제공 LLM으로 작동하도록 최적화되었으므로, 다른 모델을 사용하려면 추가 구성 및 테스트가 필요할 수 있습니다.
>
> 특히, Parlant는 작동 중에 복잡한 JSON 출력 스키마를 사용합니다. 이는 복잡한 출력을 처리할 수 있는 강력한 모델이 필요하거나, 또는 더 큰 모델을 교사로 사용하여 Parlant 데이터에 특별히 미세 조정된 더 작은 모델(SLM)을 사용해야 함을 의미합니다.
>
> 더 작은 모델을 사용하면 프로덕션 환경에서 비용, 지연 시간을 줄일 수 있고, 때로는 정확도도 향상시킬 수 있는 좋은 방법입니다.

## `NLPService` 이해하기

지원되는 기본 제공 공급자에서 다른 모델을 사용하거나 완전히 다른 공급자를 사용하려는 경우, 커스텀 `NLPService` 구현을 만들어 수행할 수 있습니다.

`NLPService`는 3가지 주요 컴포넌트를 가집니다:
1. **스키마틱 생성기(Schematic Generators)**: 프롬프트를 기반으로 구조화된 컨텐츠를 생성하는 데 사용됩니다.
2. **임베더(Embedders)**: 의미 있는 검색을 위해 텍스트의 벡터 표현을 생성하는 데 사용됩니다.
3. **모더레이션 서비스(Moderation Service)**: 대화에서 해롭거나 부적절한 사용자 입력을 필터링하는 데 사용됩니다.

> **참조 예제**
>
> 공식 [`OpenAIService`](https://github.com/emcie-co/parlant/blob/main/src/parlant/adapters/nlp/openai_service.py)에서 `NLPService`의 프로덕션 준비 완료된 참조 구현을 확인할 수 있습니다.

### 스키마틱 생성(Schematic Generation)

Parlant 엔진 전체에서, `SchematicGenerator[T]` 객체에 대한 참조를 찾을 수 있습니다. 이들은 제공된 프롬프트의 지침을 사용하여 [Pydantic](https://docs.pydantic.dev/latest/) 모델을 생성하는 객체입니다. 백그라운드에서, 그들은 항상 JSON 스키마를 생성하는 LLM을 사용하여 차례로 Pydantic 모델로 변환됩니다.

Parlant의 모든 LLM 요청은 실제로 이러한 스키마틱 생성기를 사용하여 이루어지므로, 사용하는 모델이 일관되게 유효한 JSON 스키마를 생성할 수 있어야 합니다. 이것이 Parlant에서 모델에 대한 유일한 요구사항입니다.

이제 커스텀 NLP 서비스에서 구현해야 할 몇 가지 중요한 인터페이스를 살펴봅시다.

#### 토크나이저 추정

`EstimatingTokenizer` 인터페이스는 프롬프트의 토큰 수를 추정하는 데 사용됩니다. 이는 LLM API를 사용할 때 비용과 속도 제한을 관리하는 데 중요합니다. 임베딩 모델에서도 사용되며, Parlant는 입력 텍스트를 모델의 컨텍스트 윈도우에 맞게 더 작은 부분으로 분할해야 할 때 사용됩니다.

"추정"이라고 불리는 이유는 모든 모델 API가 정확한 토큰 수를 제공하지는 않기 때문입니다.

```python
class EstimatingTokenizer(ABC):
    """프롬프트의 토큰 수를 추정하기 위한 인터페이스."""

    @abstractmethod
    async def estimate_token_count(self, prompt: str) -> int:
        """주어진 프롬프트의 토큰 수를 추정합니다."""
        ...
```

예를 들어, `OpenAI`를 사용하면, `tiktoken` 라이브러리를 사용하여 GPT 모델의 정확한 토큰 수를 얻거나 다른 인기 있는 모델의 추정 토큰 수를 얻도록 이를 구현할 수 있습니다.

#### 스키마틱 생성기

이제 프롬프트를 기반으로 구조화된 컨텐츠를 생성하는 데 사용되는 `SchematicGenerator[T]` 인터페이스 자체를 살펴봅시다.

`SchematicGenerator[T]`의 각 생성 결과에는 생성된 객체뿐만 아니라 생성 프로세스에 대한 추가 메타데이터도 포함되어 있습니다. 다음과 같습니다:

```python
@dataclass(frozen=True)
class SchematicGenerationResult(Generic[T]):
    content: T  # 생성된 스키마틱 컨텐츠 (Pydantic 모델 인스턴스)
    info: GenerationInfo  # 생성 프로세스에 대한 메타데이터


@dataclass(frozen=True)
class GenerationInfo:
    schema_name: str  # 생성된 컨텐츠에 사용된 Pydantic 스키마의 이름
    model: str  # 생성에 사용된 모델의 이름
    duration: float  # 생성에 소요된 시간 (초 단위)
    usage: UsageInfo  # 토큰 사용량 정보


@dataclass(frozen=True)
class UsageInfo:
    input_tokens: int
    output_tokens: int
    extra: Optional[Mapping[str, int]] = None  # 캐시된 입력 토큰과 같은 메트릭을 포함할 수 있습니다
```

이제 커스텀 모델에 대해 구현해야 할 `SchematicGenerator[T]` 인터페이스 자체를 살펴봅시다:

```python
class SchematicGenerator(ABC, Generic[T]):
    """프롬프트를 기반으로 구조화된 컨텐츠를 생성하기 위한 인터페이스."""

    @abstractmethod
    async def generate(
        self,
        # 생성을 위한 지침이 포함된 프롬프트 (또는 PromptBuilder).
        prompt: str | PromptBuilder,
        # 힌트는 온도, top P, logit bias와 같은 생성을 위한 추가 컨텍스트나 매개변수를 제공하는 좋은 방법입니다.
        hints: Mapping[str, Any] = {},
    ) -> SchematicGenerationResult[T]:
        """제공된 프롬프트와 힌트를 기반으로 컨텐츠를 생성합니다."""
        # 자신의 모델을 사용하여 컨텐츠를 생성하려면 이 메서드를 구현합니다.
        ...

    @property
    @abstractmethod
    def id(self) -> str:
        """생성기의 고유 식별자를 반환합니다."""
        # 일반적으로, 이는 LLM API에서 사용된 모델 이름 또는 ID입니다.
        ...

    @property
    @abstractmethod
    def max_tokens(self) -> int:
        """기본 모델의 컨텍스트 윈도우의 최대 토큰 수를 반환합니다."""
        # 모델이 처리할 수 있는 최대 토큰 수를 반환합니다.
        ...

    @property
    @abstractmethod
    def tokenizer(self) -> EstimatingTokenizer:
        """기본 모델의 토큰화기를 근사하는 토큰화기를 반환합니다."""
        # 이 토큰화기는 이 모델의 프롬프트에 대해 토큰 수를 추정할 수 있어야 합니다.
        ...

    @cached_property
    def schema(self) -> type[T]:
        """생성된 컨텐츠의 스키마 유형을 반환합니다.

        이는 파생 클래스에 유용하며, 타입 매개변수를 알 필요 없이
        현재 인스턴스의 구체적인 스키마 유형에 접근할 수 있습니다.
        """
        # 이 메서드를 구현할 필요가 없습니다 - 상속된 편의 메서드입니다.
        orig_class = getattr(self, "__orig_class__")
        generic_args = get_args(orig_class)
        return cast(type[T], generic_args[0])
```

> **참조 예제**
>
> 공식 [`OpenAIService`](https://github.com/emcie-co/parlant/blob/main/src/parlant/adapters/nlp/openai_service.py)에서 `SchematicGenerator[T]`의 프로덕션 준비 완료된 참조 구현을 확인할 수 있습니다.

### 임베딩(Embedding)

Parlant가 구조화된 컨텐츠를 생성하는 것 외에도, 임베더를 사용하여 텍스트의 벡터 표현을 생성합니다. 이들은 응답 수명 주기 전체에서 해당하는 경우 의미 있는 검색에 사용됩니다.

#### 임베딩 결과

모든 임베딩 작업은 임베더가 생성한 벡터를 포함하는 `EmbeddingResult`를 반환합니다:

```python
@dataclass(frozen=True)
class EmbeddingResult:
    vectors: Sequence[Sequence[float]]
```

#### 임베더

이제 `Embedder` 인터페이스와 이를 구현하는 방법을 살펴봅시다:

```python
class Embedder(ABC):
    @abstractmethod
    async def embed(
        self,
        texts: list[str],
        hints: Mapping[str, Any] = {},
    ) -> EmbeddingResult:
        # 주어진 텍스트에 대해 임베딩을 생성합니다.
        ...

    @property
    @abstractmethod
    def id(self) -> str:
        # 임베더의 고유 식별자를 반환합니다 - 보통 모델 이름 또는 ID입니다.
        ...

    @property
    @abstractmethod
    def max_tokens(self) -> int:
        # 모델의 컨텍스트 윈도우의 최대 토큰 수를 반환합니다.
        ...

    @property
    @abstractmethod
    def tokenizer(self) -> EstimatingTokenizer:
        # 모델의 프롬프트에 대해 토큰 수를 근사하는 토큰화기를 반환합니다.
        ...

    @property
    @abstractmethod
    def dimensions(self) -> int:
        # 임베딩 공간의 차원성을 반환합니다.
        ...
```

> **참조 예제**
>
> 공식 [`OpenAIService`](https://github.com/emcie-co/parlant/blob/main/src/parlant/adapters/nlp/openai_service.py)에서 `Embedder`의 프로덕션 준비 완료된 참조 구현을 확인할 수 있습니다.

### 모더레이션 서비스

Parlant는 해롭거나 부적절한 사용자 입력을 필터링하기 위한 포괄적인 컨텐츠 모더레이션 시스템을 포함합니다. 모더레이션 서비스는 스키마틱 생성기 및 임베더와 함께 `NLPService`의 3가지 주요 컴포넌트입니다.

#### Parlant의 모더레이션 이해하기

Parlant의 모더레이션 시스템은 AI 에이전트에 도달하기 전에 다양한 유형의 해로운 컨텐츠를 감지하고 플래그할 수 있는 컨텐츠 필터링 기능을 제공합니다. 엔진은 모든 표준 모더레이션 공급자와 통합할 수 있으며 다양한 수준의 엄격함으로 구성할 수 있습니다.

#### 모더레이션 인터페이스

모든 모더레이션 서비스는 `ModerationService` 추상 기본 클래스를 구현합니다:

```python
@dataclass(frozen=True)
class CustomerModerationContext:
    """모더레이션 확인을 위한 컨텍스트"""
    session: Session    # 확인 중인 메시지의 세션 컨텍스트
    message: str    # 확인할 메시지의 컨텐츠

@dataclass(frozen=True)
class ModerationCheck:
    """모더레이션 확인의 결과."""
    flagged: bool  # 컨텐츠가 부적절한 것으로 플래그되었는지 여부
    tags: list[str]  # 플래그된 특정 카테고리

class ModerationService(ABC):
    """컨텐츠 모더레이션 서비스를 위한 추상 기본 클래스."""

    @abstractmethod
    async def moderate_customer(self, context: CustomerModerationContext) -> ModerationCheck:
        """정책 위반에 대한 컨텐츠를 확인하고 모더레이션 결과를 반환합니다."""
        ...
```

#### 모더레이션 태그

Parlant는 일반적인 컨텐츠 정책 카테고리에 매핑되는 표준화된 모더레이션 태그를 사용합니다:

```python
ModerationTag: TypeAlias = Literal[
    "jailbreak",      # 프롬프트 주입 시도
    "harassment",     # 괴롭힘 또는 따돌림 컨텐츠
    "hate",          # 혐오 발언 또는 차별
    "illicit",       # 불법 활동 또는 물질
    "self-harm",     # 자해 또는 자살 컨텐츠
    "sexual",        # 성인용 또는 성적 컨텐츠
    "violence",      # 폭력 또는 그래픽 컨텐츠
]
```

#### 커스텀 모더레이션 서비스 구현

자신만의 모더레이션 서비스를 만드는 방법은 다음과 같습니다:

```python
import httpx
import parlant.sdk as p

class MyModerationService(p.ModerationService):
    def __init__(self, api_key: str, logger: p.Logger):
        self._api_key = api_key
        self._logger = logger
        self._client = httpx.AsyncClient()

    async def moderate_customer(self, context: p.CustomerModerationContext) -> p.ModerationCheck:
        """여기에 모더레이션 로직을 구현합니다."""
        try:
            # 예: 모더레이션 API 호출
            response = await self._client.post(
                "https://api.your-moderation-service.com/moderate",
                json={"text": context.message},
                headers={"Authorization": f"Bearer {self._api_key}"}
            )
            response.raise_for_status()

            result = response.json()

            # 서비스의 응답을 Parlant의 형식으로 매핑
            flagged = result.get("flagged", False)
            categories = result.get("categories", [])

            # 서비스의 카테고리를 Parlant의 표준화된 태그로 변환
            tags = []
            category_mapping = {
                "toxic": "harassment",
                "hate_speech": "hate",
                "violence": "violence",
                "sexual_content": "sexual",
                "self_harm": "self-harm",
                "illegal": "illicit",
                "prompt_injection": "jailbreak",
            }

            for category in categories:
                if category in category_mapping:
                    tags.append(category_mapping[category])

            return p.ModerationCheck(
                flagged=flagged,
                tags=tags,
            )

        except Exception as e:
            self._logger.error(f"Moderation check failed: {e}")
            # Fail closed: 플래그되지 않은 것을 반환하여 컨텐츠 통과 허용
            # 또는 Fail open: 플래그된 것을 반환하여 컨텐츠 차단
            return p.ModerationCheck(flagged=False, tags=[])
```

## 프롬프트 커스터마이징

자신의 `SchematicGenerator[T]`를 구현할 때, 실제로 사용하는 프롬프트를 커스터마이징할 수도 있습니다.

이는 `PromptBuilder` 클래스를 통해 달성됩니다. 이는 Parlant 엔진 전체에서 일관된 규칙과 형식을 사용하여 LLM을 위한 프롬프트를 구축하는 데 사용되는 동일한 클래스이며, 프롬프트 템플릿에 접근하고 수정할 수 있습니다.

이를 통해 할 수 있는 멋진 일 중 하나는 최종 프롬프트를 구축하기 직전에 특정 프롬프트 섹션을 편집하는 것입니다.

`CannedResponseGenerator`의 초안 작성 프롬프트를 재정의하는 방법의 예를 살펴봅시다.

```python
class MySchematicGenerator(p.SchematicGenerator[p.T]):
    async def generate(
        self,
        prompt: str | p.PromptBuilder,
        hints: Mapping[str, Any] = {},
    ) -> p.SchematicGenerationResult[T]:
        def edit_draft_instructions(section: p.PromptSection) -> p.PromptSection:
            # 동적으로 전달된 섹션의 속성을 검사할 수 있습니다
            # 수정된 템플릿에서 사용할 수 있는 것을 확인합니다.
            section.props

            section.template = f"""
            여기에 커스텀 지침을 작성합니다 ...
            필요한 경우 동적 props를 전달합니다: {section.props}
            """

            return section

        prompt.edit_section(
            name="canned-response-generator-draft-general-instructions",
            editor_func=edit_draft_instructions,
        )

        # 수정된 프롬프트로 부모 클래스의 generate 메서드 호출
        return await super().generate(prompt, hints)
```

Parlant 내 어디에서나 사용되는 모든 섹션을 수정할 수 있습니다. Parlant 코드베이스에서 `PromptBuilder.add_section()` 참조를 찾아 이러한 섹션을 찾을 수 있습니다.

## `NLPService` 구현

이제 주요 인터페이스를 이해했으므로, 자신의 `NLPService`를 구현할 수 있습니다. 이것이 쉬운 부분입니다.

다음과 같습니다:

```python
class MyNLPService(p.NLPService):
    def __init__(self, logger: p.Logger):
        self.logger = logger

    async def get_schematic_generator(self, t: type[p.T]) -> p.SchematicGenerator[p.T]:
        # 주어진 타입에 대해 커스텀 스키마틱 생성기를 반환합니다.
        return MySchematicGenerator[p.T](
            logger=self.logger,  # 로거를 사용한다고 가정
        )

    async def get_embedder(self) -> p.Embedder:
        return MyEmbedder(
            logger=self.logger,  # 로거를 사용한다고 가정
        )

    async def get_moderation_service(self) -> p.ModerationService:
        # 커스텀 모더레이션 서비스 구현을 반환합니다.
        # 모더레이션이 필요 없으면 NoModeration()을 반환합니다.
        return MyModerationService(logger=self.logger)
```

## 커스텀 `NLPService` 주입

자신의 커스텀 `NLPService`를 구현했으면, Parlant 서버에 쉽게 등록할 수 있습니다.

또한 의존성 주입 컨테이너에 대한 참조를 얻을 수 있으며, 필요에 따라 시스템 로거 및 기타 서비스에 접근할 수 있습니다.

```python
def load_custom_nlp_service(container: p.Container) -> p.NLPService:
    return MyNLPService(
        logger=container[p.Logger]
    )
```

그런 다음 Parlant 서버를 시작할 때, 로더 함수를 `nlp_service` 매개변수에 전달합니다:

```python
async with p.Server(
    nlp_service=load_custom_nlp_service,
) as server:
    # 여기에 코드를 작성합니다
```
