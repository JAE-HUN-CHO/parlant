# Snowflake Cortex 어댑터

Snowflake 호스팅 LLM을 사용하여 **채팅/구조화된 생성** 및 **임베딩**을 위한 [Snowflake Cortex REST API](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-rest-api)를 통합합니다. 어댑터는 다음과 직접 통신합니다:

- `POST /api/v2/cortex/inference:complete` - 채팅/JSON 출력
- `POST /api/v2/cortex/inference:embed` - 임베딩

## 요구사항

### 인증
[Snowflake REST API 인증](https://docs.snowflake.com/en/developer-guide/snowflake-rest-api/authentication)을 참조하세요. PAT를 권장합니다.

### 환경 변수
사용 가능한 모델 이름은 [Cortex 모델](https://docs.snowflake.com/en/user-guide/snowflake-cortex/llm)을 참조하세요.

```bash
export SNOWFLAKE_CORTEX_BASE_URL="https://<account>.snowflakecomputing.com"
export SNOWFLAKE_AUTH_TOKEN="<jwt-or-pat>"
export SNOWFLAKE_CORTEX_CHAT_MODEL="mistral-large2"
export SNOWFLAKE_CORTEX_EMBED_MODEL="e5-base-v2"
# 선택사항:
export SNOWFLAKE_CORTEX_MAX_TOKENS="8192"
```

## 사용 예시

```python
import parlant.sdk as p
from parlant.sdk import NLPServices

@p.tool
async def get_weather(context: p.ToolContext, city: str) -> p.ToolResult:
    # 날씨 API 로직을 여기에 작성하세요
    return p.ToolResult(f"Sunny, 72°F in {city}")

@p.tool
async def get_datetime(context: p.ToolContext) -> p.ToolResult:
    from datetime import datetime
    return p.ToolResult(datetime.now())

async def main():
    async with p.Server(nlp_service=NLPServices.snowflake) as server:
        agent = await server.create_agent(
            name="WeatherBot",
            description="Helpful weather assistant"
        )

        # 컨텍스트 변수를 사용하여 매 응답마다 에이전트의 컨텍스트가
        # 업데이트되도록 합니다 (업데이트 간격은 커스터마이징 가능).
        await agent.create_variable(name="current-datetime", tool=get_datetime)

        # 자연어로 에이전트 동작을 제어하고 가이드합니다
        await agent.create_guideline(
            condition="User asks about weather",
            action="Get current weather and provide a friendly response with suggestions",
            tools=[get_weather]
        )

        # 다른 (안정적으로 강제되는) 행동 모델링 요소를 추가하세요
        # ...

        # 🎉 http://localhost:8800에서 테스트 플레이그라운드 준비
        # 공식 React 위젯을 앱에 통합하거나,
        # 튜토리얼을 따라 자체 프론트엔드를 구축하세요!

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

## 설정 레퍼런스

| 변수                         | 필수 | 설명 |
|----------------------------------|----------|-------------|
| `SNOWFLAKE_CORTEX_BASE_URL`      | ✅       | 기본 계정 URL (예: `https://<account>.snowflakecomputing.com`).  |
| `SNOWFLAKE_AUTH_TOKEN`           | ✅       | `Authorization: Bearer` 헤더에 사용되는 OAuth / Keypair JWT / PAT.  |
| `SNOWFLAKE_CORTEX_CHAT_MODEL`    | ✅       | 채팅 모델 이름. |
| `SNOWFLAKE_CORTEX_EMBED_MODEL`   | ✅       | 임베딩 모델 이름. |
| `SNOWFLAKE_CORTEX_MAX_TOKENS`    | ❌       | 생성에 대한 로컬 상한선; 제공자 제한을 재정의하지 않습니다.  |


## 개인정보 보호 및 데이터 거주성 참고사항

어댑터를 사용하면 앱이 Snowflake 계정에서 Cortex를 직접 호출할 수 있으므로 LLM 작업을 위해 Snowflake 외부로 데이터를 이동할 필요성이 줄어듭니다. 지역 가용성 및 계정 설정에 대한 Snowflake의 REST 가이드를 검토하세요.
