# 24. 파일 시스템과 검색

이 문서는 OpenCode의 파일 관리, 검색, 감시 시스템을 설명한다.

**소스 파일:**
- `packages/opencode/src/file/index.ts` — 파일 연산 및 상태
- `packages/opencode/src/file/ripgrep.ts` — Ripgrep 바이너리 관리 및 검색
- `packages/opencode/src/file/ignore.ts` — 무시 패턴
- `packages/opencode/src/file/time.ts` — 파일 수정 시간 추적
- `packages/opencode/src/file/watcher.ts` — 파일 시스템 감시

## Ripgrep: 고속 파일 검색

### 개요

OpenCode는 내부적으로 [ripgrep](https://github.com/BurntSushi/ripgrep)(rg) 바이너리를 사용하여 파일 검색과 열거를 수행한다. 플랫폼별 바이너리를 자동으로 다운로드하고 관리한다.

### 주요 함수

```typescript
// packages/opencode/src/file/ripgrep.ts
export namespace Ripgrep {
  // ripgrep 바이너리 경로 (없으면 자동 다운로드)
  export function filepath(): Promise<string>

  // 파일 열거 — 비동기 제너레이터
  export async function* files(input: {
    cwd: string
    glob?: string[]       // 글로브 패턴
    hidden?: boolean      // 숨김 파일 포함
    follow?: boolean      // 심링크 따라가기
    maxDepth?: number     // 최대 탐색 깊이
    signal?: AbortSignal  // 취소 시그널
  }): AsyncGenerator<string>

  // 디렉터리 트리 구조
  export function tree(input: {
    cwd: string
    limit?: number        // 최대 파일 수
    signal?: AbortSignal
  }): Promise<string>

  // JSON 형식 내용 검색
  export function search(input: {
    cwd: string
    pattern: string       // 정규식 패턴
    glob?: string[]       // 파일 필터
    limit?: number        // 최대 결과 수
    follow?: boolean
  }): Promise<Match[]>
}
```

### 바이너리 관리

- **버전:** ripgrep v14.1.1
- **캐시 위치:** `Global.Path.cache/ripgrep/`
- **플랫폼 지원:** darwin (arm64/x64), linux (arm64/x64), win32 (x64)
- 최초 실행 시 GitHub에서 자동 다운로드

### 검색 결과 타입

```typescript
export const Match = z.object({
  path: z.string(),
  lines: z.string(),
  line_number: z.number(),
  // ...
})
```

## File: 파일 연산

### 파일 읽기

```typescript
// packages/opencode/src/file/index.ts
export namespace File {
  // 파일 내용 읽기 (이미지/바이너리 자동 감지)
  export async function read(filepath: string): Promise<Content>

  // 디렉터리 목록
  export async function list(dir?: string): Promise<Node[]>

  // 퍼지 검색
  export async function search(
    query: string,
    limit?: number,
    dirs?: string[],
    type?: "file" | "directory"
  ): Promise<Node[]>

  // Git 상태 (변경된 파일 목록)
  export async function status(): Promise<Info[]>

  // 파일 캐시 초기화
  export function init(): Promise<void>
}
```

### Content 타입

```typescript
export type Content = {
  type: "text" | "binary"
  content: string        // 텍스트 또는 base64
  diff?: string          // Git diff (있는 경우)
  patch?: string         // 패치 형식
  encoding?: string      // 인코딩
  mimeType?: string      // MIME 타입 (이미지 등)
}
```

### 바이너리 파일 감지

300개 이상의 파일 확장자를 바이너리로 분류한다: `.png`, `.jpg`, `.pdf`, `.exe`, `.zip`, `.wasm` 등.

이미지 파일은 base64로 인코딩하여 LLM에 첨부(attachment)로 전달할 수 있다.

### 이벤트

```typescript
export const Event = {
  Edited: BusEvent.define("file.edited", z.object({
    filepath: z.string(),
  })),
}
```

`File.Event.Edited`는 도구가 파일을 수정할 때 발행되며, Format 모듈이 구독하여 자동 포매팅을 수행한다.

## FileIgnore: 무시 패턴

### 기본 무시 패턴

```typescript
// packages/opencode/src/file/ignore.ts
export namespace FileIgnore {
  export function match(filepath: string, options?: {
    extra?: string[]      // 추가 무시 패턴
    whitelist?: string[]  // 화이트리스트 패턴
  }): boolean
}
```

**기본 무시 디렉터리** (34개):

```
node_modules, .git, dist, build, .next, .nuxt, __pycache__,
.venv, venv, target, .gradle, .idea, .vscode, .DS_Store,
coverage, .cache, tmp, temp, .terraform, vendor, ...
```

**기본 무시 파일 패턴:**
```
*.swp, *.log, coverage/*, ...
```

## FileTime: 수정 시간 추적

### 개요

세션별로 파일의 읽기/쓰기 시간을 추적하여, 외부에서 파일이 수정되었을 때 경고한다.

```typescript
// packages/opencode/src/file/time.ts
export namespace FileTime {
  // 파일 수정 시간 검증 (외부 수정 감지)
  export function assert(sessionID: string, filepath: string): void

  // 파일 쓰기 잠금 (동일 파일 동시 쓰기 방지)
  export function withLock<T>(
    filepath: string,
    fn: () => Promise<T>
  ): Promise<T>
}
```

### 동시 쓰기 보호

`withLock()`은 같은 파일에 대한 동시 쓰기를 직렬화한다. 여러 도구가 동시에 같은 파일을 수정하려 할 때 데이터 손실을 방지한다.

## FileWatcher: 파일 감시

### 개요

파일 시스템 변경을 실시간으로 감지하여 이벤트를 발행한다. Parcel의 네이티브 감시자를 사용한다.

```typescript
// packages/opencode/src/file/watcher.ts
export namespace FileWatcher {
  export const Event = {
    Updated: BusEvent.define("file.updated", z.object({
      type: z.enum(["add", "change", "unlink"]),
      path: z.string(),
    })),
  }
}
```

**플랫폼별 백엔드:**
- Windows: `@parcel/watcher-win32-x64`
- macOS: `@parcel/watcher-darwin-*` (FSEvents)
- Linux: `@parcel/watcher-linux-*` (inotify)

**실험적 기능:** `OPENCODE_EXPERIMENTAL_FILEWATCHER` 플래그로 활성화.

## 참고 문서

- [07. 도구 시스템](./07-tool-system.md) — Glob, Grep, Read, Write, Edit 도구
- [14. 저장소와 상태](./14-storage-and-state.md) — 스냅샷에서 파일 변경 추적
- [26. LSP와 포매팅](./26-lsp-and-formatting.md) — 파일 수정 후 포매팅
