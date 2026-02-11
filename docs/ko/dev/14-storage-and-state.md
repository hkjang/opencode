# 14. 저장소와 상태

이 문서는 OpenCode의 저장소(Storage)와 상태 관리를 설명한다. 파일 기반 영속 저장, 인스턴스 컨텍스트, 스냅샷(Snapshot) 시스템을 다룬다.

**소스 파일:**
- `packages/opencode/src/storage/storage.ts` — 파일 기반 저장소
- `packages/opencode/src/project/instance.ts` — 인스턴스 관리
- `packages/opencode/src/snapshot/` — 스냅샷 시스템

## Storage: 파일 기반 영속 저장소

### 개요

Storage는 JSON 파일 기반의 키-값 저장소이다. 키는 문자열 배열이며, 파일 시스템 경로로 변환된다.

```
키: ["session", "project-1", "session-123"]
경로: {dataDir}/storage/session/project-1/session-123.json
```

### 주요 함수

```typescript
// packages/opencode/src/storage/storage.ts
export namespace Storage {
  // 읽기 (읽기 잠금)
  export async function read<T>(key: string[]): Promise<T>

  // 쓰기 (쓰기 잠금)
  export async function write<T>(key: string[], content: T): Promise<void>

  // 읽기-수정-쓰기 (쓰기 잠금)
  export async function update<T>(
    key: string[],
    fn: (current: T) => T
  ): Promise<T>

  // 삭제
  export async function remove(key: string[]): Promise<void>

  // 키 목록 조회 (접두어 기반)
  export async function list(prefix: string[]): Promise<string[][]>
}
```

### 동시성 제어

읽기/쓰기 잠금(Lock)을 사용하여 동시 접근을 안전하게 처리한다:
- `Lock.read()`: 여러 읽기 동시 가능, 쓰기 중에는 대기
- `Lock.write()`: 배타적 잠금, 다른 읽기/쓰기 모두 대기

### 데이터 계층 구조

```
storage/
├── session/
│   └── {projectID}/
│       └── {sessionID}.json        ← Session.Info
├── message/
│   └── {sessionID}/
│       └── {messageID}.json        ← MessageV2.Info
├── part/
│   └── {messageID}/
│       └── {partID}.json           ← MessageV2.Part
├── permission/
│   └── approved.json               ← PermissionNext 승인 룰셋
└── migration                        ← 마이그레이션 추적
```

### 마이그레이션

Storage는 내부 마이그레이션 시스템을 갖고 있다:

1. **레거시 프로젝트 저장소 이전**: 이전 형식의 세션/메시지를 새 구조로 변환
2. **세션 요약 diff 추출**: 세션 요약에서 diff 데이터를 분리

마이그레이션 상태는 `migration` 파일에 기록되어 중복 실행을 방지한다.

### 에러 처리

```typescript
export const NotFoundError = NamedError.create("StorageNotFound", z.object({
  key: z.array(z.string()),
}))
```

존재하지 않는 키에 접근하면 `NotFoundError`가 발생한다.

## Instance: 프로젝트 컨텍스트

### 개요

Instance는 프로젝트 디렉터리별 런타임 컨텍스트를 관리한다. 비동기 로컬 저장소(AsyncLocalStorage)를 활용하여 호출 스택 전체에서 현재 프로젝트 정보에 접근할 수 있다.

### 핵심 API

```typescript
// packages/opencode/src/project/instance.ts

// 프로젝트 속성
Instance.directory   // 현재 작업 디렉터리
Instance.worktree    // Git 워크트리 루트 (비-Git이면 "/")
Instance.project     // Project.Info 객체

// 컨텍스트 제공
await Instance.provide({ directory: "/path" }, async () => {
  // 이 콜백 내에서 Instance.directory 등 사용 가능
})

// 경로 확인
Instance.containsPath("/path/to/file")  // 프로젝트 범위 내인지 확인

// 디렉터리 범위 상태
const myState = Instance.state(
  async () => ({ /* 초기화 */ }),  // init
  async (s) => { /* 정리 */ }     // dispose
)

// 정리
await Instance.dispose()     // 현재 인스턴스 정리
await Instance.disposeAll()  // 모든 인스턴스 정리
```

### Instance.state() 동작 원리

```
Instance.state(init, dispose)
  │
  ├── 첫 호출: init() 실행 → 결과 캐싱 (디렉터리별)
  ├── 이후 호출: 캐싱된 값 반환
  └── dispose 시: dispose() 실행 → 캐시 제거
```

**전역 캐시:** 디렉터리별로 `Map`에 저장되어 동일 디렉터리에 대한 재초기화를 방지한다.

### 사용하는 모듈

거의 모든 핵심 모듈이 `Instance.state()`를 사용한다:

| 모듈 | 캐싱하는 상태 |
|------|-------------|
| Config | 병합된 설정, 디렉터리 목록 |
| Agent | 에이전트 목록 |
| Provider | 모델 목록, SDK 인스턴스 |
| ToolRegistry | 커스텀 도구 |
| Bus | 구독 맵 |
| PermissionNext | 대기 중 요청, 승인 룰셋 |
| MCP | 연결 상태, 클라이언트 |
| Plugin | 훅, SDK 클라이언트 |
| Storage | 저장소 디렉터리, 마이그레이션 상태 |

## Snapshot: 파일 시스템 스냅샷

### 개요

스냅샷 시스템은 도구 실행 전후의 파일 시스템 상태를 기록하여, 에이전트가 수행한 변경 사항을 추적하고 되돌릴 수 있게 한다.

### 동작 방식

```
1. step-start 이벤트
   └── 현재 파일 해시 기록 (Snapshot 시작)

2. 도구 실행 (파일 읽기/쓰기)

3. step-finish 이벤트
   └── 변경된 파일 감지 → PatchPart 생성
       └── diff 계산 (이전 vs 현재)
```

### PatchPart

```typescript
const PatchPart = z.object({
  type: z.literal("patch"),
  path: z.string(),      // 변경된 파일 경로
  hash: z.string(),      // 파일 해시
  diff: z.string(),      // unified diff
})
```

스냅샷 데이터는 `SnapshotPart`로 메시지에 포함되어, 세션의 전체 파일 변경 이력을 추적할 수 있다.

## 참고 문서

- [03. 핵심 패턴](./03-core-patterns.md) — Instance.state() 패턴 설명
- [05. 세션 생명주기](./05-session-lifecycle.md) — 스냅샷이 세션 처리에서 사용되는 방식
- [08. 이벤트 버스](./08-bus-event-system.md) — 인스턴스 해제 이벤트
