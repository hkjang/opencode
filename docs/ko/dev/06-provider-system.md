# 06. 프로바이더 시스템

이 문서는 OpenCode의 프로바이더(Provider) 시스템을 설명한다. 20개 이상의 LLM 서비스를 통합하는 추상화 계층이다.

**소스 파일:** `packages/opencode/src/provider/provider.ts`

## 개요

프로바이더 시스템은 다양한 LLM 서비스(Anthropic, OpenAI, Google 등)를 통일된 인터페이스로 추상화한다. AI SDK(Vercel)의 프로바이더 어댑터를 활용하며, 모델 선택, 인증, 비용 추적, 커스텀 설정을 관리한다.

## 핵심 타입

### Provider.Model

```typescript
export const Model = z.object({
  id: z.string(),              // 모델 ID (예: "claude-sonnet-4-20250514")
  name: z.string(),            // 표시 이름
  providerID: z.string(),      // 프로바이더 ID (예: "anthropic")
  // 가격 정보
  cost: z.object({
    input: z.number(),         // 입력 토큰당 비용 (USD)
    output: z.number(),        // 출력 토큰당 비용
    cache_read: z.number().optional(),
    cache_write: z.number().optional(),
  }).optional(),
  // 모델 능력
  limit: z.object({
    context: z.number(),       // 최대 컨텍스트 길이
    output: z.number(),        // 최대 출력 길이
  }),
  capabilities: z.object({
    temperature: z.boolean(),
    topP: z.boolean(),
    topK: z.boolean(),
  }).optional(),
  // 변형
  variants: z.record(...).optional(),
})
```

### Provider.Info

```typescript
export const Info = z.object({
  id: z.string(),
  name: z.string(),
  models: z.array(Model),
  source: z.enum(["env", "config", "custom", "api"]),
})
```

## 번들 프로바이더

OpenCode에 기본 내장된 프로바이더 목록:

| 프로바이더 | SDK 패키지 | 인증 방식 |
|-----------|-----------|-----------|
| anthropic | @ai-sdk/anthropic | API 키 |
| openai | @ai-sdk/openai | API 키 |
| google | @ai-sdk/google-vertex | API 키 / OAuth |
| aws | @ai-sdk/amazon-bedrock | AWS 자격증명 |
| azure | @ai-sdk/azure | API 키 |
| mistral | @ai-sdk/mistral | API 키 |
| groq | @ai-sdk/groq | API 키 |
| deepseek | @ai-sdk/deepseek | API 키 |
| xai | @ai-sdk/xai | API 키 |
| cohere | @ai-sdk/cohere | API 키 |
| deepinfra | @ai-sdk/deepinfra | API 키 |
| fireworks | @ai-sdk/fireworks | API 키 |
| together | @ai-sdk/togetherai | API 키 |
| sambanova | @ai-sdk/sambanova | API 키 |
| cerebras | @ai-sdk/cerebras | API 키 |
| openrouter | @openrouter/ai-sdk-provider | API 키 |
| copilot | (커스텀) | GitHub Copilot |
| codex | (커스텀) | GitHub Codex |
| ollama | ollama-ai-provider | 로컬 |
| litellm | (OpenAI 호환) | API 키 |
| opencode | (내부) | OpenCode 인증 |

## 주요 함수

### Provider.list()

```typescript
export function list(): Provider.Info[]
```

사용 가능한 모든 프로바이더와 모델 목록을 반환한다. 프로바이더 소스별로 수집한다:

1. **env**: 환경 변수로 감지된 프로바이더 (예: `ANTHROPIC_API_KEY`)
2. **config**: `opencode.json`에 설정된 프로바이더
3. **custom**: 사용자 커스텀 프로바이더
4. **api**: OpenCode API에서 제공하는 프로바이더

### Provider.getModel()

```typescript
export function getModel(
  providerID: string,
  modelID: string
): Provider.Model
```

프로바이더 ID와 모델 ID로 모델을 검색한다. 정확한 매칭이 실패하면 퍼지 매칭(Fuzzy matching)으로 유사한 모델을 제안한다.

### Provider.getLanguage()

```typescript
export function getLanguage(model: Provider.Model): LanguageModel
```

AI SDK의 `LanguageModel` 인스턴스를 반환한다. 이 인스턴스로 `streamText()`나 `generateText()`를 호출할 수 있다.

내부적으로:
1. 프로바이더별 커스텀 로더(Loader)로 SDK 인스턴스 생성
2. 모델 옵션 및 인증 정보 주입
3. 커스텀 fetch 래퍼로 타임아웃, 헤더 처리
4. 결과를 해시(Hash) 키로 캐싱

### Provider.defaultModel()

```typescript
export function defaultModel(): { providerID: string, modelID: string }
```

Config에서 설정된 기본 모델을 반환한다.

### Provider.getSmallModel()

```typescript
export function getSmallModel(providerID: string): Provider.Model
```

경량 모델을 반환한다 (휴리스틱 기반). 제목 생성, 요약 등 간단한 작업에 사용된다.

### Provider.parseModel()

```typescript
export function parseModel(modelString: string): { providerID: string, modelID: string }
```

`"anthropic/claude-sonnet-4-20250514"` 형식의 문자열을 파싱한다.

## 커스텀 Fetch 래퍼

모든 LLM 요청은 커스텀 fetch 래퍼를 통과한다:

```typescript
// 내부 구현
async function customFetch(url, options) {
  // 1. 타임아웃 처리 (AbortSignal 합성)
  // 2. OpenAI 호환 프로바이더: 요청 본문에서 stream_options 제거
  // 3. 헤더 주입 (User-Agent, 커스텀 헤더)
  // 4. 에러 처리 및 재시도 지원
  return fetch(url, options)
}
```

## 모델 변형(Variant)

하나의 모델이 여러 설정 변형을 가질 수 있다:

```jsonc
// opencode.json
{
  "providers": {
    "anthropic": {
      "models": {
        "claude-sonnet-4-20250514": {
          "variants": {
            "creative": {
              "temperature": 0.9,
              "topP": 0.95
            },
            "precise": {
              "temperature": 0.1,
              "topP": 0.1
            }
          }
        }
      }
    }
  }
}
```

## 모델 정렬

`Provider.sort()`는 모델을 다음 우선순위로 정렬한다:

1. 사용자가 최근 사용한 모델
2. 프로바이더별 기본 모델
3. 모델 능력 (출력 토큰 제한 등)

## AWS Bedrock 특수 처리

AWS Bedrock은 리전(Region)과 모델 접두어에 복잡한 로직이 필요하다:

- 리전별 모델 가용성 확인
- 크로스 리전 추론 프로필 지원
- 모델 ID 접두어 매핑 (`anthropic.` → Anthropic 모델 등)

## 참고 문서

- [04. 에이전트 시스템](./04-agent-system.md) — 에이전트별 모델 선택
- [05. 세션 생명주기](./05-session-lifecycle.md) — LLM 호출 흐름
- [18. 프로바이더 추가하기](./18-adding-a-provider.md) — 새 프로바이더 추가 가이드
