# 26. LSP와 포매팅

이 문서는 OpenCode의 Language Server Protocol(LSP) 통합과 자동 코드 포매팅 시스템을 설명한다.

**소스 파일:**
- `packages/opencode/src/lsp/index.ts` — LSP 조율
- `packages/opencode/src/lsp/client.ts` — LSP 클라이언트
- `packages/opencode/src/lsp/server.ts` — LSP 서버 정의 (30+ 서버)
- `packages/opencode/src/lsp/language.ts` — 언어 ID 매핑
- `packages/opencode/src/format/index.ts` — 포매터 관리
- `packages/opencode/src/format/formatter.ts` — 포매터 정의 (30+)

## LSP 통합

### 개요

OpenCode는 다수의 LSP 서버를 관리하여 코드 분석(진단, 정의 이동, 참조 찾기 등) 기능을 제공한다. 에이전트가 파일을 수정하면 LSP 서버가 진단(Diagnostics)을 제공하여 에이전트에 피드백한다.

### LSP API

```typescript
// packages/opencode/src/lsp/index.ts
export namespace LSP {
  // 초기화 (서버 자동 생성)
  export function init(): Promise<void>

  // 서버 상태
  export function status(): Promise<Status[]>

  // 파일 변경 알림
  export function touchFile(path: string, waitForDiagnostics?: boolean): Promise<void>

  // 진단 (에러/경고)
  export function diagnostics(): Diagnostic[]

  // 코드 인텔리전스
  export function hover(file: string, position: Position): Promise<Hover>
  export function definition(file: string, position: Position): Promise<Location[]>
  export function references(file: string, position: Position): Promise<Location[]>
  export function implementation(file: string, position: Position): Promise<Location[]>

  // 심볼 검색
  export function workspaceSymbol(query: string): Promise<Symbol[]>
  export function documentSymbol(file: string): Promise<DocumentSymbol[]>

  // 호출 계층
  export function prepareCallHierarchy(file: string, position: Position): Promise<CallHierarchyItem[]>
  export function incomingCalls(item: CallHierarchyItem): Promise<IncomingCall[]>
  export function outgoingCalls(item: CallHierarchyItem): Promise<OutgoingCall[]>

  // 진단 포맷팅
  export namespace Diagnostic {
    export function pretty(diagnostics: Diagnostic[]): string
  }
}
```

### 서버 생명주기

```
파일 열기/수정
  │
  ├── 확장자 → 언어 ID 매핑 (language.ts)
  │
  ├── 해당 언어의 LSP 서버 확인
  │     ├── 이미 실행 중 → 재사용
  │     ├── 생성 가능 → 자식 프로세스 생성
  │     └── 실패한 서버 → 재시도 안 함
  │
  ├── textDocument/didOpen 알림
  │
  └── 진단 수신 (150ms 디바운싱)
       └── LSP.Event.Updated 이벤트 발행
```

### 지연 생성(Lazy spawning)

LSP 서버는 해당 언어의 파일이 처음 접근될 때만 생성된다. 이를 통해 불필요한 리소스 사용을 방지한다.

### 내장 LSP 서버

`server.ts`에 30개 이상의 LSP 서버가 정의되어 있다:

| 언어 | 서버 | 설치 명령 |
|------|------|-----------|
| TypeScript/JavaScript | typescript-language-server | npm |
| Python | pylsp / pyright / ty | pip / npm |
| Rust | rust-analyzer | rustup |
| Go | gopls | go install |
| Java | jdtls | (수동) |
| C/C++ | clangd | (시스템) |
| Ruby | solargraph | gem |
| PHP | intelephense | npm |
| Swift | sourcekit-lsp | (Xcode) |
| Kotlin | kotlin-language-server | (수동) |
| Lua | lua-language-server | (수동) |
| Zig | zls | zig |
| Elixir | elixir-ls / next-ls | mix |
| ... | ... | ... |

### 커스텀 LSP 서버

```jsonc
// opencode.json
{
  "lsp": {
    "my-language": {
      "command": ["my-lsp-server", "--stdio"],
      "extensions": ["*.ml"],
      "init": {
        "customOption": true
      }
    }
  }
}
```

### 실험적 기능

- `OPENCODE_EXPERIMENTAL_LSP_TOOL`: LSP 도구를 에이전트에게 노출 (hover, definition 등)
- `OPENCODE_EXPERIMENTAL_LSP_TY`: Python에 Pyright 대신 Ty 사용

## 자동 포매팅

### 개요

파일이 수정될 때 해당 언어의 포매터를 자동으로 실행한다. `File.Event.Edited` 이벤트를 구독하여 백그라운드에서 포매팅한다.

### 포매터 목록 (30+)

| 포매터 | 언어 | 감지 기준 |
|--------|------|-----------|
| prettier | JS/TS/HTML/CSS/JSON/YAML/MD | package.json 의존성 또는 .prettierrc |
| biome | JS/TS/JSON | package.json 의존성 또는 biome.json |
| gofmt | Go | go 명령어 존재 |
| rustfmt | Rust | rustfmt 명령어 존재 |
| black | Python | black 명령어 존재 + pyproject.toml 확인 |
| ruff | Python | ruff 명령어 존재 |
| mix format | Elixir | mix 프로젝트 |
| swift-format | Swift | swift-format 명령어 존재 |
| dart format | Dart | dart 명령어 존재 |
| clang-format | C/C++/Java/Protobuf | .clang-format 존재 |
| stylua | Lua | stylua 명령어 존재 |
| zigfmt | Zig | zig 명령어 존재 |
| ktlint | Kotlin | ktlint 명령어 존재 |
| rubocop | Ruby | rubocop 명령어 존재 |
| terraform fmt | Terraform | terraform 명령어 존재 |
| shfmt | Shell | shfmt 명령어 존재 |
| laravel pint | PHP | Laravel 프로젝트 |
| ... | ... | ... |

### 포매터 구조

```typescript
// packages/opencode/src/format/formatter.ts
export const prettier = {
  name: "prettier",
  command: ["npx", "prettier", "--write"],
  extensions: ["*.js", "*.ts", "*.jsx", "*.tsx", "*.html", "*.css", "*.json", "*.yaml", "*.md"],
  async enabled() {
    // package.json에 prettier 의존성이 있거나
    // .prettierrc 파일이 존재하는지 확인
    return hasDependency("prettier") || hasConfigFile(".prettierrc")
  },
  environment: {
    // 포매터별 환경 변수
  },
}
```

### 포매터 감지

각 포매터는 `enabled()` 비동기 함수로 활성화 여부를 판단한다:

1. **바이너리 존재 확인**: `which()` 또는 `Bun.which()`
2. **설정 파일 확인**: `.prettierrc`, `biome.json`, `pyproject.toml` 등
3. **의존성 확인**: `package.json`의 dependencies/devDependencies

### 포매터 설정

```jsonc
// opencode.json
{
  "format": {
    "prettier": {
      "disabled": false,
      "command": ["npx", "prettier", "--write", "--single-quote"]
    },
    "black": {
      "disabled": true   // Black 포매터 비활성화
    }
  }
}
```

### 실행 흐름

```
File.Event.Edited (파일 수정 이벤트)
  │
  ├── 파일 확장자로 매칭되는 포매터 검색
  │
  ├── enabled() 확인
  │
  └── Bun.spawn(command + [filepath])
       └── 백그라운드 실행 (블로킹 없음)
       └── 에러 시 로깅만 (포매팅 실패는 무시)
```

### 이벤트

```typescript
export namespace Format {
  export const Status = z.object({
    name: z.string(),
    enabled: z.boolean(),
    extensions: z.array(z.string()),
  })

  export function status(): Promise<Status[]>  // 포매터 상태 조회
  export function init(): void                  // 이벤트 구독 초기화
}
```

## 참고 문서

- [07. 도구 시스템](./07-tool-system.md) — Edit/Write 도구가 LSP를 활용하는 방식
- [10. 설정 시스템](./10-config-system.md) — LSP/포매터 설정
- [11. 서버 API](./11-server-api.md) — `/lsp`, `/formatter` 엔드포인트
- [24. 파일 시스템과 검색](./24-file-and-search.md) — File.Event.Edited
