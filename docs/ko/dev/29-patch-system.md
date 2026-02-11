# 29. 패치 시스템

이 문서는 OpenCode의 패치(Patch) 파싱 및 적용 시스템을 설명한다. `ApplyPatch` 도구가 사용하는 커스텀 패치 형식과 지능형 줄 매칭 알고리즘을 다룬다.

**소스 파일:** `packages/opencode/src/patch/index.ts`

## 개요

패치 시스템은 LLM이 생성한 파일 변경 지시를 파싱하고 적용한다. unified diff와 유사하지만 더 유연한 커스텀 형식을 사용하며, 4단계 줄 매칭 전략으로 공백 차이에 강인하게 동작한다.

## 패치 형식

```
*** Begin Patch
*** Add File: path/to/new-file.ts
+export function hello() {
+  return "world"
+}

*** Delete File: path/to/remove.ts

*** Update File: path/to/modify.ts
@@ optional context description @@
 context line (변경 없음)
-old line to remove
+new line to add
 another context line

*** Update File: path/to/move.ts
*** Move to: path/to/new-location.ts
@@ function body @@
-old code
+new code

*** End Patch
```

## 작업 타입

### Add (파일 추가)

```
*** Add File: src/new-module.ts
+import z from "zod"
+
+export const schema = z.object({
+  name: z.string(),
+})
```

모든 줄이 `+`로 시작한다. 부모 디렉터리가 없으면 자동 생성한다.

### Delete (파일 삭제)

```
*** Delete File: src/old-module.ts
```

파일을 완전히 삭제한다.

### Update (파일 수정)

```
*** Update File: src/module.ts
@@ validate function @@
 export function validate(input: string) {
-  return input.length > 0
+  if (!input) return false
+  return input.trim().length > 0
 }
```

- `@@` 줄: 컨텍스트 설명 (선택적, 매칭에 사용)
- ` ` (공백): 변경 없는 컨텍스트 줄
- `-`: 삭제할 줄
- `+`: 추가할 줄

### Move (파일 이동 + 수정)

```
*** Update File: src/old-name.ts
*** Move to: src/new-name.ts
@@ @@
 // 파일 내용 수정도 가능
```

## 핵심 함수

```typescript
// packages/opencode/src/patch/index.ts
export namespace Patch {
  // 패치 텍스트를 Hunk 배열로 파싱
  export function parsePatch(text: string): Hunk[]

  // Hunk 배열을 파일 시스템에 적용
  export async function applyPatch(hunks: Hunk[]): Promise<Result>

  // 청크에서 새 파일 내용 생성
  export function deriveNewContentsFromChunks(
    originalContent: string,
    chunks: UpdateFileChunk[]
  ): string

  // 셸 명령에서 패치 감지 (heredoc 추출)
  export function maybeParseApplyPatch(args: string): Hunk[] | null

  // 검증된 패치 파싱
  export function maybeParseApplyPatchVerified(args: string): Hunk[] | null
}
```

## Hunk 타입

```typescript
type Hunk =
  | {
      type: "add"
      path: string
      contents: string    // 새 파일 내용
    }
  | {
      type: "delete"
      path: string
    }
  | {
      type: "update"
      path: string
      move_path?: string  // 이동 대상 경로
      chunks: UpdateFileChunk[]
    }
```

## UpdateFileChunk

```typescript
type UpdateFileChunk = {
  old_lines: string[]           // 기존 줄 (컨텍스트 + 삭제)
  new_lines: string[]           // 새 줄 (컨텍스트 + 추가)
  change_context?: string       // @@ 컨텍스트 설명
  is_end_of_file?: boolean      // 파일 끝 앵커
}
```

## 4단계 줄 매칭 알고리즘

`deriveNewContentsFromChunks()`는 기존 파일에서 청크의 `old_lines`를 찾을 때 4단계 폴백 전략을 사용한다:

```
1단계: 정확한 매칭 (Exact)
  └── 줄 내용이 정확히 일치

2단계: 우측 공백 제거 (Rstrip)
  └── 줄 끝 공백을 제거한 후 비교

3단계: 양측 공백 제거 (Trim)
  └── 줄 양쪽 공백을 제거한 후 비교

4단계: 유니코드 정규화 (Unicode Normalized)
  └── NFKC 정규화 + 공백 제거 후 비교
```

이 전략으로 LLM이 들여쓰기나 공백을 약간 다르게 생성해도 올바른 위치에 패치를 적용할 수 있다.

## 역순 적용

같은 파일에 여러 청크를 적용할 때, **파일 끝에서부터 역순**으로 적용한다. 이를 통해 앞쪽 줄의 인덱스가 변경되어 뒤쪽 매칭에 영향을 주는 문제를 방지한다.

```
예: 10줄 파일에서 3번째와 7번째 줄을 수정할 때
  1. 먼저 7번째 줄 수정 (뒤쪽)
  2. 그 다음 3번째 줄 수정 (앞쪽)
→ 7번째 줄 수정이 3번째 줄의 위치에 영향을 주지 않음
```

## EOF 앵커

`is_end_of_file` 플래그가 설정된 청크는 파일 끝에 매칭한다:

```
*** Update File: src/module.ts
@@ @@
 // 마지막 함수
 }
+
+// 파일 끝에 추가할 새 코드
+export function newFeature() {}
```

## Heredoc 추출

Bash 도구에서 `cat <<'EOF'` 형태의 heredoc으로 패치를 적용하는 경우를 자동 감지한다:

```bash
cat <<'EOF' | apply_patch
*** Begin Patch
*** Update File: src/module.ts
...
*** End Patch
EOF
```

`maybeParseApplyPatch()`가 셸 명령에서 heredoc을 추출하고, 내용이 패치 형식인지 확인한다.

## 에러 타입

```typescript
// 패치 파싱 에러
export const ParseError = NamedError.create("PatchParse", z.object({
  message: z.string(),
}))

// 파일 I/O 에러
export const IoError = NamedError.create("PatchIo", z.object({
  path: z.string(),
  message: z.string(),
}))

// 줄 매칭 실패
export const ComputeReplacements = NamedError.create("PatchComputeReplacements", z.object({
  path: z.string(),
  chunk: z.any(),
}))
```

## ApplyPatch 도구와의 연동

```
LLM이 패치 텍스트 생성
  │
  ▼
ApplyPatchTool.execute(patchText)
  │
  ├── Patch.parsePatch(patchText) → Hunk[]
  │
  ├── Hunk별 적용:
  │     ├── add → 파일 생성 + File.Event.Added
  │     ├── delete → 파일 삭제 + File.Event.Unlinked
  │     └── update → deriveNewContentsFromChunks + File.Event.Edited
  │
  └── 변경된 파일에 LSP 진단 실행
```

## 참고 문서

- [25. 도구 구현 상세](./25-tool-implementation-details.md) — ApplyPatch, Edit 도구
- [07. 도구 시스템](./07-tool-system.md) — 도구 인터페이스
- [24. 파일 시스템과 검색](./24-file-and-search.md) — 파일 이벤트
