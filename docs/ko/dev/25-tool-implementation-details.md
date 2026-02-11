# 25. 도구 구현 상세

이 문서는 OpenCode에 내장된 19개 도구의 구현 세부사항을 설명한다. 각 도구의 파라미터, 권한 모델, 핵심 로직을 다룬다.

**소스 디렉터리:** `packages/opencode/src/tool/`

## 도구 전체 목록

| # | 도구 | ID | 권한 | 설명 |
|---|------|----|------|------|
| 1 | Bash | `bash` | `bash` | 셸 명령 실행 |
| 2 | Read | `read` | `read` | 파일 읽기 |
| 3 | Write | `write` | `edit` | 파일 쓰기 |
| 4 | Edit | `edit` | `edit` | 파일 부분 편집 |
| 5 | Glob | `glob` | `glob` | 파일 패턴 검색 |
| 6 | Grep | `grep` | `grep` | 파일 내용 검색 |
| 7 | WebFetch | `webfetch` | `webfetch` | URL 내용 가져오기 |
| 8 | WebSearch | `websearch` | `websearch` | 웹 검색 |
| 9 | CodeSearch | `codesearch` | `codesearch` | 코드 검색 |
| 10 | Task | `task` | `task` | 하위 에이전트 생성 |
| 11 | Skill | `skill` | `skill` | 스킬 로딩 |
| 12 | TodoWrite | `todowrite` | `todowrite` | TODO 업데이트 |
| 13 | TodoRead | `todoread` | `todoread` | TODO 조회 |
| 14 | ApplyPatch | `applypatch` | `edit` | 통합 패치 적용 |
| 15 | Batch | `batch` | — | 병렬 도구 실행 |
| 16 | Question | `question` | — | 사용자에게 질문 |
| 17 | PlanEnter | `planenter` | — | 계획 모드 진입 |
| 18 | PlanExit | `planexit` | — | 계획 모드 종료 |
| 19 | InvalidTool | `invalid` | — | 잘못된 도구 호출 처리 |

## Bash: 셸 명령 실행

**파일:** `packages/opencode/src/tool/bash.ts`

```typescript
parameters: z.object({
  command: z.string(),                    // 실행할 명령어
  timeout: z.number().optional(),         // 타임아웃 (ms)
  workdir: z.string().optional(),         // 작업 디렉터리
  description: z.string(),               // 5~10단어 설명
})
```

**핵심 구현:**
- **Tree-sitter 파싱**: Bash 명령어를 AST로 파싱하여 파괴적 명령어(`rm`, `mv`, `chmod` 등) 감지
- **경로 검증**: 파괴적 명령어의 대상 경로가 프로젝트 범위 내인지 확인
- **프로세스 생성**: 자식 프로세스로 명령 실행, 타임아웃 시 프로세스 트리 종료
- **출력 제한**: stdout/stderr 합쳐서 30KB 메타데이터로 제한
- **권한**: `bash` 권한 + 외부 디렉터리 접근 시 `external_directory` 권한

## Read: 파일 읽기

**파일:** `packages/opencode/src/tool/read.ts`

```typescript
parameters: z.object({
  filePath: z.string(),                   // 파일 경로
  offset: z.coerce.number().optional(),   // 시작 줄 번호
  limit: z.coerce.number().optional(),    // 읽을 줄 수
})
```

**핵심 구현:**
- **파일 타입 감지**: 이미지/PDF → 첨부파일(attachment)로 반환, 바이너리 → 에러
- **줄 번호 형식**: `00001| 내용` 형태로 줄 번호를 붙여 반환
- **줄 제한**: 기본 2000줄, 최대 줄 길이 2000자 (초과 시 `...` 추가)
- **바이트 제한**: 최대 바이트 초과 시 자동 잘라냄
- **파일 없음 시**: 유사한 파일명을 3개까지 제안
- **수정 시간 기록**: `FileTime`에 읽기 시간 기록 (외부 수정 감지용)

## Edit: 파일 부분 편집

**파일:** `packages/opencode/src/tool/edit.ts`

```typescript
parameters: z.object({
  filePath: z.string(),
  oldString: z.string(),   // 교체할 텍스트
  newString: z.string(),   // 새 텍스트
  replaceAll: z.boolean().optional(),  // 전체 교체 (기본: false)
})
```

**핵심 구현 — 9단계 폴백 교체 전략:**

```
1. SimpleReplacer        — 정확한 문자열 매칭
2. LineTrimmedReplacer   — 줄별 공백 제거 후 매칭
3. BlockAnchorReplacer   — 앵커 기반 블록 매칭 (유사도 점수)
4. WhitespaceNormalizedReplacer — 공백 정규화 후 매칭
5. IndentationFlexibleReplacer — 들여쓰기 무시 매칭
6. EscapeNormalizedReplacer    — 이스케이프 시퀀스 정규화
7. TrimmedBoundaryReplacer     — 경계 공백 제거
8. ContextAwareReplacer        — 컨텍스트 기반 앵커 매칭
9. MultiOccurrenceReplacer     — 다중 발생 검색
```

- **Levenshtein 거리**: 유사도 점수 계산으로 가장 적합한 매칭 선택
- **LSP 진단**: 편집 후 LSP 진단을 실행하여 오류 감지
- `oldString === newString` 검증 → 에러

## Write: 파일 쓰기

**파일:** `packages/opencode/src/tool/write.ts`

```typescript
parameters: z.object({
  content: z.string(),
  filePath: z.string(),
})
```

**핵심 구현:**
- 파일이 존재하면 수정, 없으면 생성
- 부모 디렉터리 자동 생성 (`fs.mkdirSync({ recursive: true })`)
- `File.Event.Edited` (수정) 또는 `File.Event.Added` (생성) 이벤트 발행
- 프로젝트 전체에 대해 LSP 진단 실행 (최대 5개 파일의 에러 보고)

## Glob: 파일 패턴 검색

**파일:** `packages/opencode/src/tool/glob.ts`

```typescript
parameters: z.object({
  pattern: z.string(),               // 글로브 패턴
  path: z.string().optional(),       // 검색 디렉터리
})
```

**핵심 구현:**
- Ripgrep의 `files()` 제너레이터로 파일 열거
- 최대 100개 결과
- 경로 해석: 상대 경로 → `Instance.directory` 기준으로 절대 경로 변환

## Grep: 내용 검색

**파일:** `packages/opencode/src/tool/grep.ts`

```typescript
parameters: z.object({
  pattern: z.string(),                // 정규식 패턴
  path: z.string().optional(),        // 검색 디렉터리
  include: z.string().optional(),     // 파일 필터 (예: "*.ts")
})
```

**핵심 구현:**
- ripgrep 바이너리 직접 실행
- 최대 100개 매칭 결과
- 수정 시간 기준 정렬 (최신 파일 우선)
- 긴 줄 2000자로 잘라냄

## WebFetch: URL 내용 가져오기

**파일:** `packages/opencode/src/tool/webfetch.ts`

```typescript
parameters: z.object({
  url: z.string(),
  format: z.enum(["text", "markdown", "html"]).default("markdown"),
  timeout: z.number().optional(),     // 최대 120초
})
```

**핵심 구현:**
- **Cloudflare 감지**: 403 + `cf-mitigated` 헤더 시 User-Agent 변경 후 재시도
- **5MB 제한**: 응답 크기 초과 시 잘라냄
- **HTML → Markdown**: Turndown 라이브러리 사용
- **텍스트 추출**: HTMLRewriter로 `<script>`, `<style>`, `<embed>` 제외

## Task: 하위 에이전트 생성

**파일:** `packages/opencode/src/tool/task.ts`

```typescript
parameters: z.object({
  description: z.string(),           // 3~5단어 설명
  prompt: z.string(),                // 작업 지시사항
  subagent_type: z.string(),         // 에이전트 이름
  task_id: z.string().optional(),    // 기존 세션 재개용 ID
  command: z.string().optional(),    // 트리거 명령어
})
```

**핵심 구현:**
- 호출자의 권한에 따라 사용 가능한 에이전트 필터링
- 새 세션 생성 또는 `task_id`로 기존 세션 재개
- 하위 에이전트에서 `todowrite`, `todoread`, `task` 권한 거부
- 결과를 `<task_result>` XML 태그로 래핑
- 세션 ID 반환 (이후 재개 가능)

## ApplyPatch: 패치 적용

**파일:** `packages/opencode/src/tool/apply_patch.ts`

```typescript
parameters: z.object({
  patchText: z.string(),    // 패치 텍스트
})
```

**핵심 구현:**
- `Patch.parsePatch()` → `Patch.applyPatch()` 2단계
- Add / Update / Delete / Move 네 가지 작업 지원
- 부모 디렉터리 자동 생성
- 파일 이벤트 발행 (add/change/unlink)
- 변경된 파일에 LSP 진단 실행

## Batch: 병렬 도구 실행

**파일:** `packages/opencode/src/tool/batch.ts`

```typescript
parameters: z.object({
  tool_calls: z.array(z.object({
    tool: z.string(),
    parameters: z.record(z.any()),
  })),  // 최소 1개, 최대 25개
})
```

**핵심 구현:**
- 최대 25개 도구를 병렬(`Promise.all`) 실행
- `batch`와 `applypatch`는 호출 불가
- 각 도구 호출의 성공/실패를 개별 추적
- 파트(Part) 상태를 running → completed/error로 업데이트

## Question: 사용자에게 질문

**파일:** `packages/opencode/src/tool/question.ts`

```typescript
parameters: z.object({
  questions: z.array(Question.Info),  // 질문 배열
})
```

**핵심 구현:**
- `Question.ask()`로 UI에 다이얼로그 표시
- 사용자가 응답할 때까지 대기 (Promise)
- 복수 선택 지원
- 커스텀 응답(자유 텍스트) 지원

## 조건부 도구 활성화 요약

```typescript
// packages/opencode/src/tool/registry.ts — 조건 요약

// OpenCode 프로바이더 또는 OPENCODE_ENABLE_EXA 플래그
WebSearchTool, CodeSearchTool

// GPT-5 이상만 (나머지는 Edit/Write)
ApplyPatchTool

// OPENCODE_EXPERIMENTAL_LSP_TOOL 플래그
LspTool

// config.experimental.batch_tool = true
BatchTool

// 플래그 + CLI 클라이언트
PlanEnterTool, PlanExitTool

// app/cli/desktop 클라이언트
QuestionTool
```

## 참고 문서

- [07. 도구 시스템](./07-tool-system.md) — Tool.define() 인터페이스
- [09. 권한 시스템](./09-permission-system.md) — ctx.ask() 동작
- [19. 도구 추가하기](./19-adding-a-tool.md) — 새 도구 추가 가이드
- [29. 패치 시스템](./29-patch-system.md) — Patch 파싱/적용 상세
