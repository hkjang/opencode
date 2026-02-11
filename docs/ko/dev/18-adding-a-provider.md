# 18. 프로바이더 추가하기

이 문서는 OpenCode에 새 LLM 프로바이더(Provider)를 추가하는 실전 가이드이다.

**핵심 파일:** `packages/opencode/src/provider/provider.ts`

## 개요

새 프로바이더를 추가하는 방법은 크게 세 가지이다:

1. **AI SDK 프로바이더 추가** — `@ai-sdk/*` 패키지가 있는 프로바이더
2. **OpenAI 호환 프로바이더** — OpenAI API 호환 서비스
3. **완전한 커스텀 프로바이더** — 특수한 인증이나 모델 처리가 필요한 경우

## 방법 1: AI SDK 프로바이더 추가

AI SDK에 이미 프로바이더 패키지가 존재하는 경우 가장 간단하다.

### 단계 1: 의존성 추가

```bash
cd packages/opencode
bun add @ai-sdk/newprovider
```

### 단계 2: BUNDLED_PROVIDERS에 등록

```typescript
// packages/opencode/src/provider/provider.ts

// 파일 상단의 BUNDLED_PROVIDERS 맵에 추가
const BUNDLED_PROVIDERS: Record<string, (options: any) => SDK> = {
  // 기존 프로바이더들...
  "@ai-sdk/anthropic": createAnthropic,
  "@ai-sdk/openai": createOpenAI,
  // ...

  // 새 프로바이더 추가
  "@ai-sdk/newprovider": createNewProvider,
}
```

**키:** npm 패키지명
**값:** AI SDK의 `create*` 팩토리 함수

### 단계 3: 커스텀 로더 추가 (선택)

특별한 초기화 로직이 필요한 경우 `CUSTOM_LOADERS`에 추가한다.

```typescript
const CUSTOM_LOADERS: Record<string, CustomLoader> = {
  // 기존 로더들...

  async "newprovider"() {
    return {
      autoload: false,  // 환경 변수 감지 시 자동 로드 여부
      options: {
        // 프로바이더별 옵션
        headers: {
          "X-Custom-Header": "value",
        },
      },
    }
  },
}
```

**CustomLoader 반환 타입:**

```typescript
{
  autoload: boolean              // true: API 키 없이도 자동 로드
  options?: Record<string, any>  // 프로바이더 SDK에 전달할 옵션
  getModel?: (                   // 커스텀 모델 로더 (선택)
    sdk: any,
    modelID: string,
    options?: Record<string, any>
  ) => Promise<LanguageModel>
}
```

### 단계 4: 모델 정보

모델 정보는 [models.dev](https://models.dev) API에서 자동으로 가져온다. 모델이 API에 등록되어 있지 않다면 Config에서 수동으로 추가할 수 있다.

```jsonc
// opencode.json
{
  "providers": {
    "newprovider": {
      "api": {
        "key": "{env:NEWPROVIDER_API_KEY}"
      },
      "npm": "@ai-sdk/newprovider",
      "models": {
        "new-model-v1": {
          "name": "New Model v1",
          "attachment": true,
          "cost": { "input": 3, "output": 15 },
          "limit": { "context": 200000, "output": 8192 }
        }
      }
    }
  }
}
```

## 방법 2: OpenAI 호환 프로바이더

OpenAI API와 호환되는 서비스는 설정만으로 추가할 수 있다.

```jsonc
// opencode.json
{
  "providers": {
    "my-local-llm": {
      "api": {
        "url": "http://localhost:8000/v1",
        "key": "any-value"
      },
      "npm": "@ai-sdk/openai-compatible",
      "name": "My Local LLM",
      "models": {
        "my-model": {
          "name": "My Custom Model",
          "cost": { "input": 0, "output": 0 },
          "limit": { "context": 128000, "output": 4096 }
        }
      }
    }
  }
}
```

## 방법 3: 커스텀 프로바이더 (고급)

복잡한 인증이나 모델 처리가 필요한 경우이다.

### 예시: AWS Bedrock 스타일

```typescript
const CUSTOM_LOADERS: Record<string, CustomLoader> = {
  async "my-custom-provider"(input) {
    const config = await Config.get()
    const auth = await Auth.get("my-custom-provider")

    return {
      autoload: true,
      options: {
        // 프로바이더 SDK 옵션
        region: config.provider?.["my-custom-provider"]?.options?.region ?? "us-east-1",
        // 인증 정보
        accessKeyId: auth?.accessKeyId,
        secretAccessKey: auth?.secretAccessKey,
      },
      // 커스텀 모델 로더 — 모델 ID 변환 등
      async getModel(sdk, modelID, options) {
        // 모델 ID를 프로바이더 형식으로 변환
        const transformedID = transformModelID(modelID)
        return sdk.languageModel(transformedID, options)
      },
    }
  },
}
```

## 프로바이더 커스텀 Fetch

모든 프로바이더의 HTTP 요청은 커스텀 fetch 래퍼를 통과한다. 이 래퍼는:

1. **타임아웃 처리**: AbortSignal을 합성하여 요청 타임아웃 적용
2. **헤더 주입**: User-Agent 및 커스텀 헤더
3. **호환성**: OpenAI 호환 프로바이더에서 `stream_options` 제거 등

프로바이더별 특수 처리가 필요하면 `CUSTOM_LOADERS`의 `options`에 커스텀 fetch를 포함할 수 있다.

## 검증 절차

1. **설정 확인**: `opencode debug config`로 프로바이더가 로드되었는지 확인
2. **모델 목록**: `opencode models`로 사용 가능한 모델 확인
3. **실제 호출**: TUI에서 모델을 선택하고 간단한 질문으로 테스트
4. **에러 처리**: 잘못된 API 키나 모델 ID로 에러 메시지 확인
5. **SDK 재생성**: API 변경이 있으면 `./script/generate.ts` 실행

## 체크리스트

- [ ] AI SDK 프로바이더 패키지 설치 (또는 OpenAI 호환 설정)
- [ ] `BUNDLED_PROVIDERS`에 등록 (내장 프로바이더의 경우)
- [ ] `CUSTOM_LOADERS`에 커스텀 로더 추가 (필요 시)
- [ ] 모델 정보 확인 (models.dev 또는 Config)
- [ ] 인증 흐름 확인 (API 키, OAuth 등)
- [ ] `opencode debug config`로 로드 확인
- [ ] 실제 LLM 호출 테스트
- [ ] SDK 재생성 (`./script/generate.ts`)

## 참고 문서

- [06. 프로바이더 시스템](./06-provider-system.md) — 프로바이더 시스템 아키텍처
- [10. 설정 시스템](./10-config-system.md) — 프로바이더 설정 구조
- [CONTRIBUTING.ko.md](../../../CONTRIBUTING.ko.md) — PR 규칙
