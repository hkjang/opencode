# 21. 인증과 세션 공유

이 문서는 OpenCode의 인증(Auth) 모듈과 세션 공유(Share) 시스템을 설명한다.

**소스 파일:**
- `packages/opencode/src/auth/index.ts` — 인증 관리
- `packages/opencode/src/share/share.ts` — 레거시 공유
- `packages/opencode/src/share/share-next.ts` — 최신 공유 시스템

## 인증 시스템 (Auth)

### 개요

Auth 모듈은 LLM 프로바이더(Provider)와 외부 서비스의 인증 정보를 안전하게 관리한다. 세 가지 인증 방식을 지원하며, 파일 시스템에 제한된 권한(`0o600`)으로 저장한다.

### 인증 타입

```typescript
// packages/opencode/src/auth/index.ts

// OAuth 토큰 기반 인증
export const Oauth = z.object({
  type: z.literal("oauth"),
  access_token: z.string(),
  refresh_token: z.string().optional(),
  expires_at: z.number().optional(),
})

// API 키 기반 인증
export const Api = z.object({
  type: z.literal("api"),
  key: z.string(),
})

// Well-known 서비스 인증
export const WellKnown = z.object({
  type: z.literal("wellknown"),
  // 서비스별 자격 증명
})

// 판별 유니온
export const Info = z.discriminatedUnion("type", [Oauth, Api, WellKnown])
```

### 주요 함수

```typescript
export namespace Auth {
  // 특정 프로바이더의 인증 정보 조회
  export async function get(providerID: string): Promise<Info | undefined>

  // 모든 인증 정보 조회
  export async function all(): Promise<Record<string, Info>>

  // 인증 정보 저장
  export async function set(key: string, info: Info): Promise<void>

  // 인증 정보 삭제
  export async function remove(key: string): Promise<void>
}
```

### 저장 위치

인증 데이터는 `Global.Path.data/auth.json`에 저장된다. 파일 권한은 `0o600`(소유자만 읽기/쓰기)으로 설정되어 다른 사용자가 접근할 수 없다.

```
~/.local/share/opencode/auth.json  (Linux)
~/Library/Application Support/opencode/auth.json  (macOS)
%LOCALAPPDATA%/opencode/auth.json  (Windows)
```

### CLI 인증 관리

```bash
opencode auth login anthropic    # Anthropic API 키 입력
opencode auth login openai       # OpenAI API 키 입력
opencode auth logout anthropic   # 인증 삭제
opencode auth status             # 모든 프로바이더 인증 상태
```

### OAuth 더미 키

OAuth 기반 프로바이더(Copilot, Codex 등)는 API 키 대신 OAuth 토큰을 사용한다. `OAUTH_DUMMY_KEY` 상수는 이런 프로바이더가 키 기반 검증을 통과할 수 있게 한다.

## 세션 공유 (Share)

### 개요

세션 공유 시스템은 대화 세션을 외부 URL로 공유할 수 있게 한다. 레거시(`Share`)와 최신(`ShareNext`) 두 구현이 존재한다.

### 비활성화

```bash
export OPENCODE_DISABLE_SHARE=1  # 공유 기능 비활성화
```

### ShareNext (최신 구현)

```typescript
// packages/opencode/src/share/share-next.ts
export namespace ShareNext {
  // 공유 API URL
  export function url(): string

  // 세션 공유 생성 (공유 ID 반환)
  export async function create(sessionID: string): Promise<string>

  // 공유 삭제
  export async function remove(sessionID: string): Promise<void>

  // 초기화 — Bus 이벤트 구독
  export function init(): void
}
```

### 실시간 동기화

`init()`이 호출되면 다음 Bus 이벤트를 구독하여 세션 변경을 실시간으로 공유 서버에 전송한다:

```
Session.Event.Updated       → 세션 메타데이터 동기화
MessageV2.Event.Updated     → 메시지 동기화
MessageV2.Event.PartUpdated → 메시지 파트 동기화
Session.Event.Diff          → 파일 변경 diff 동기화
```

### 디바운싱

동기화 요청은 1초 간격으로 디바운싱(debouncing)되어 빈번한 업데이트로 인한 API 부하를 방지한다.

### 공유 API URL

| 환경 | URL |
|------|-----|
| 개발 | `https://api.dev.opencode.ai` |
| 프로덕션 | `https://api.opencode.ai` |

환경 변수 `OPENCODE_SHARE_URL`로 커스텀 URL을 지정할 수 있다.

### Storage 연동

ShareNext는 Storage에 공유 메타데이터를 영속 저장하여, 서버 재시작 후에도 공유 상태를 유지한다.

## 참고 문서

- [06. 프로바이더 시스템](./06-provider-system.md) — 프로바이더 인증
- [10. 설정 시스템](./10-config-system.md) — 인증 설정
- [13. MCP 통합](./13-mcp-integration.md) — MCP OAuth 인증
- [27. 인프라 모듈](./27-infrastructure-modules.md) — Global 경로
