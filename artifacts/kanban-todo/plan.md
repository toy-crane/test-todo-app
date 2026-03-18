# Kanban Todo 구현 계획

## Architecture Decisions

| 결정 사항 | 선택 | 사유 |
|-----------|------|------|
| 드래그&드롭 라이브러리 | `@atlaskit/pragmatic-drag-and-drop` | Atlassian(Trello/Jira)에서 실제 사용 중. 경량, 프레임워크 비종속적, 접근성 지원 |
| 상태 관리 | React Context + useReducer | 로컬 전용 앱으로 외부 라이브러리 불필요. reducer로 액션 기반 상태 변경 관리 |
| 다크모드 | Tailwind `dark` class 전환 | `<html>` 에 `dark` 클래스 토글. shadcn semantic token 자동 대응 |
| 데이터 영속화 | localStorage + JSON 스키마 버저닝 | useEffect로 상태 변경 시 자동 저장. 스키마 버전 포함하여 마이그레이션 대비 |
| 에러 알림 | sonner (toast) | shadcn 공식 토스트 솔루션. JSON 가져오기 에러 표시에 사용 |

## Required Skills

| 스킬 | 용도 |
|------|------|
| `vercel-react-best-practices` | React/Next.js 성능 최적화 규칙 |
| `web-design-guidelines` | Web Interface Guidelines 준수 |
| `shadcn` | shadcn/ui 컴포넌트 사용 규칙 |

## UI Components

### 설치 필요

| 컴포넌트 | 설치 명령 |
|----------|-----------|
| Progress | `bunx --bun shadcn@latest add progress` |
| Sonner (Toast) | `bunx --bun shadcn@latest add sonner` |
| Alert | `bunx --bun shadcn@latest add alert` |

### 커스텀 컴포넌트

| 컴포넌트 | 역할 |
|----------|------|
| `KanbanBoard` | 3칼럼 칸반 보드 레이아웃 + Context Provider |
| `KanbanColumn` | 칼럼 헤더 + 카드 리스트 + 드롭 영역 |
| `KanbanCard` | 카드 표시 (제목, 우선순위 뱃지, 태그, 서브태스크 진행률, 마감일) + 드래그 소스 |
| `CardEditDialog` | 카드 편집 모달 (Dialog 기반, 모든 필드 편집) |
| `AddCardForm` | 인라인 카드 추가 폼 |
| `SubtaskList` | 서브태스크 체크리스트 + 진행률 표시 |
| `SearchFilterBar` | 검색 입력 + 우선순위/태그 Select 필터 + 필터 칩 |
| `BoardHeader` | 앱 제목 + 다크모드 토글 + Export/Import 버튼 |

## 실행 프로토콜

- 각 task 시작 전, **참조 규칙**에 나열된 파일을 반드시 읽고 규칙을 준수하며 구현한다

## Tasks

### Task 0: 프로젝트 설정 및 의존성 설치

- **시나리오**: (선행 작업 — 모든 시나리오 구현에 필요한 컴포넌트 및 라이브러리 설치)
- **참조 규칙**: `.claude/skills/shadcn/SKILL.md`
- **구현 대상**:
  - shadcn 컴포넌트 설치: `progress`, `sonner`, `alert`
  - 외부 라이브러리 설치: `@atlaskit/pragmatic-drag-and-drop`
  - 설치 후 컴포넌트 파일 검증
- **수용 기준**:
  - [ ] `bunx --bun shadcn@latest add progress sonner alert` 실행 → `components/ui/progress.tsx`, `components/ui/alert.tsx` 생성됨
  - [ ] `bun add @atlaskit/pragmatic-drag-and-drop` 실행 → `package.json`에 의존성 추가됨
  - [ ] `bun run build` 에러 없음
- **커밋**: `chore: add progress, sonner, alert components and pragmatic-drag-and-drop`

---

### Task 1: Spec 테스트 생성

- **시나리오**: KANBAN-001 ~ KANBAN-020 (전체)
- **참조 규칙**: `artifacts/spec.yaml`, `artifacts/kanban-todo/wireframe.html`, `CLAUDE.md` (spec 테스트 작성 규칙)
- **구현 대상**:
  - `__tests__/kanban-board.spec.test.tsx` — spec.yaml의 모든 시나리오를 수용 기준 테스트로 작성
  - 요소 선택은 `getByRole`, `getByLabelText`, `getByText` 등 안정적 패턴 사용
  - wireframe의 컴포넌트 타입에 맞는 인터랙션 패턴 사용 (Dialog, Select, Switch, Checkbox, Progress)
- **수용 기준**:
  - [ ] `bun run test __tests__/kanban-board.spec.test.tsx` 실행 → 테스트 20개가 모두 FAIL (Red 상태)
  - [ ] 각 테스트가 spec.yaml의 시나리오 ID를 `describe`/`it` 이름에 포함
  - [ ] 드래그&드롭 테스트(KANBAN-007, 008)는 BoardContext의 MOVE_CARD 디스패치를 직접 호출하여 상태 변경을 검증한다 (pragmatic-drag-and-drop은 브라우저 D&D API에 의존하므로 jsdom에서 시뮬레이션 불가)
- **커밋**: `test: add spec tests for all kanban board scenarios (KANBAN-001~020)`

---

### Task 2: 데이터 모델 및 상태 관리

- **시나리오**: KANBAN-016, KANBAN-017 (localStorage 저장/복원의 기반)
- **참조 규칙**: `.claude/skills/vercel-react-best-practices/rules/client-localstorage-schema.md`, `.claude/skills/vercel-react-best-practices/rules/rerender-functional-setstate.md`, `.claude/skills/vercel-react-best-practices/rules/rerender-derived-state-no-effect.md`
- **구현 대상**:
  - `lib/types.ts` — Card, Column, BoardState 타입 정의
  - `lib/board-reducer.ts` — 보드 상태 reducer (ADD_CARD, UPDATE_CARD, DELETE_CARD, MOVE_CARD, ADD_SUBTASK, TOGGLE_SUBTASK, DELETE_SUBTASK)
  - `lib/board-context.tsx` — BoardContext + BoardProvider (useReducer + localStorage 연동)
  - `lib/storage.ts` — localStorage 읽기/쓰기 유틸리티 (스키마 버전 포함)
- **수용 기준**:
  - [ ] reducer 단위 테스트 (`__tests__/board-reducer.test.tsx`) — 각 액션에 대한 상태 변경 검증
  - [ ] localStorage 유틸리티 테스트 — 저장/로드/유효하지 않은 데이터 처리 검증
  - [ ] `bun run test __tests__/board-reducer.test.tsx` → 모든 테스트 PASS
- **커밋**: `feat: add board data model, reducer, and localStorage persistence`

---

### Task 3: 칸반 보드 기본 레이아웃

- **시나리오**: KANBAN-001, KANBAN-002
- **참조 규칙**: `.claude/skills/shadcn/rules/styling.md`, `.claude/skills/shadcn/rules/composition.md`, `.claude/skills/shadcn/rules/icons.md`, `.claude/skills/vercel-react-best-practices/rules/rerender-no-inline-components.md`, `.claude/skills/vercel-react-best-practices/rules/rendering-conditional-render.md`
- **구현 대상**:
  - `components/kanban/board.tsx` — KanbanBoard: 3칼럼 그리드 레이아웃 + BoardProvider 래핑
  - `components/kanban/column.tsx` — KanbanColumn: 칼럼 헤더(이름 + 카드 수) + 카드 리스트 + 빈 상태("No cards yet") + "Add card" 버튼
  - `components/kanban/card.tsx` — KanbanCard: 제목, 우선순위 Badge, 태그 Badge, 서브태스크 진행률(Progress), 마감일
  - `components/kanban/add-card-form.tsx` — AddCardForm: 인라인 제목 입력 + Cancel/Add 버튼
  - `app/page.tsx` — KanbanBoard 렌더링
- **수용 기준**:
  - [ ] 앱 로드 시 "To Do", "In Progress", "Done" 3개 칼럼 헤더 표시됨 (KANBAN-001)
  - [ ] 저장된 데이터 없음 → 각 칼럼에 카드 0개, 빈 상태 표시
  - [ ] "To Do" 칼럼에서 "우유 사기" 입력 후 추가 → 카드 생성됨 (KANBAN-002)
  - [ ] 빈 제목으로 추가 시도 → 카드 생성되지 않음
  - [ ] `bun run test` → KANBAN-001, KANBAN-002 spec 테스트 PASS
- **커밋**: `feat: add kanban board layout with columns and card creation`

---

### Task 4: 카드 편집 모달

- **시나리오**: KANBAN-003, KANBAN-004, KANBAN-005, KANBAN-006, KANBAN-009, KANBAN-010
- **참조 규칙**: `.claude/skills/shadcn/rules/composition.md` (Dialog 구성), `.claude/skills/shadcn/rules/forms.md` (Field, FieldGroup, validation), `.claude/skills/shadcn/rules/styling.md`, `.claude/skills/shadcn/rules/icons.md`, `.claude/skills/vercel-react-best-practices/rules/rerender-functional-setstate.md`
- **구현 대상**:
  - `components/kanban/card-edit-dialog.tsx` — CardEditDialog (Dialog 기반 모달)
    - 필드: 제목(Input), 설명(Textarea), 우선순위(Select), 마감일(Input type text), 태그(입력+뱃지+삭제)
    - 서브태스크: 체크리스트(Checkbox + 삭제 버튼) + 추가 입력 + Progress 바
    - 삭제 버튼(휴지통 아이콘) → 카드 삭제
    - Save/Cancel 버튼
  - `components/kanban/subtask-list.tsx` — SubtaskList: 서브태스크 CRUD + 진행률
- **수용 기준**:
  - [ ] 카드 클릭 → 편집 Dialog에 제목, 설명, 우선순위, 태그, 마감일 필드 표시 (KANBAN-003)
  - [ ] 제목 변경 후 Save → 카드 제목 반영됨 (KANBAN-004)
  - [ ] 우선순위 변경 후 Save → 카드에 반영됨 (KANBAN-004)
  - [ ] 제목 변경 후 Cancel → 원본 유지됨 (KANBAN-005)
  - [ ] 삭제 버튼 클릭 → 카드 삭제됨 (KANBAN-006)
  - [ ] 서브태스크 추가 → 체크리스트에 표시 (KANBAN-009)
  - [ ] 서브태스크 완료 토글 → 진행률 업데이트 (KANBAN-009)
  - [ ] 서브태스크 삭제 → 체크리스트에서 제거, 진행률 업데이트 (KANBAN-010)
  - [ ] `bun run test` → KANBAN-003~006, 009, 010 spec 테스트 PASS
- **커밋**: `feat: add card edit dialog with subtask management`

---

### Task 5: 카드 드래그&드롭 이동

- **시나리오**: KANBAN-007, KANBAN-008
- **참조 규칙**: `.claude/skills/vercel-react-best-practices/rules/bundle-dynamic-imports.md`, `.claude/skills/shadcn/rules/styling.md`, `.claude/skills/shadcn/rules/icons.md`
- **구현 대상**:
  - `components/kanban/card.tsx` — 드래그 소스 설정 (`draggable` from pragmatic-drag-and-drop)
  - `components/kanban/column.tsx` — 드롭 영역 설정 (`dropTargetForElements`)
  - 드래그 중: 원래 위치 점선 플레이스홀더, 드롭 대상 칼럼 강조
  - 드롭 완료: MOVE_CARD 액션 디스패치
- **수용 기준**:
  - [ ] "To Do"의 카드를 "In Progress"로 드래그&드롭 → "To Do"에서 사라지고 "In Progress"에 표시 (KANBAN-007)
  - [ ] "In Progress"의 카드를 "Done"으로 드래그&드롭 → 이동 반영 (KANBAN-008)
  - [ ] `bun run test` → KANBAN-007, KANBAN-008 spec 테스트 PASS
- **커밋**: `feat: add drag and drop card movement between columns`

---

### Task 6: 검색·필터

- **시나리오**: KANBAN-011, KANBAN-012, KANBAN-013, KANBAN-014
- **참조 규칙**: `.claude/skills/shadcn/rules/styling.md`, `.claude/skills/shadcn/rules/composition.md`, `.claude/skills/shadcn/rules/icons.md`, `.claude/skills/vercel-react-best-practices/rules/rerender-derived-state-no-effect.md`, `.claude/skills/vercel-react-best-practices/rules/rerender-derived-state.md`
- **구현 대상**:
  - `components/kanban/search-filter-bar.tsx` — SearchFilterBar
    - 검색 Input (실시간 필터링, ✕로 초기화)
    - 우선순위 Select 드롭다운 (High/Medium/Low/All)
    - 태그 Select 드롭다운 (보드 내 존재하는 태그 목록)
    - Active 필터 칩 표시 + 개별 ✕ 제거 + "Clear all"
  - 필터 로직: 보드 컨텍스트에 필터 상태 추가, 파생 상태로 필터링된 카드 목록 계산
- **수용 기준**:
  - [ ] "우유" 검색 → "우유 사기" 카드만 표시, "회의록 작성" 숨겨짐 (KANBAN-011)
  - [ ] 검색어 삭제 → 모든 카드 표시 (KANBAN-011)
  - [ ] 우선순위 "High" 선택 → High 카드만 표시 (KANBAN-012)
  - [ ] 태그 "업무" 선택 → "업무" 태그 카드만 표시 (KANBAN-013)
  - [ ] 필터 해제 → 모든 카드 표시 (KANBAN-014)
  - [ ] `bun run test` → KANBAN-011~014 spec 테스트 PASS
- **커밋**: `feat: add search and priority/tag filters with filter chips`

---

### Task 7: 다크모드

- **시나리오**: KANBAN-015
- **참조 규칙**: `.claude/skills/shadcn/rules/styling.md` (semantic token, no manual dark: override), `.claude/skills/shadcn/rules/icons.md` (sun/moon 아이콘), `.claude/skills/vercel-react-best-practices/rules/rendering-hydration-no-flicker.md`
- **구현 대상**:
  - `components/kanban/board-header.tsx` — 다크모드 Switch 토글 (Sun/Moon 아이콘)
  - `<html>` 요소에 `dark` 클래스 토글
  - localStorage에 테마 설정 저장
- **수용 기준**:
  - [ ] 라이트 모드에서 토글 클릭 → `<html>`에 `dark` 클래스 추가, 다크 모드 스타일 적용 (KANBAN-015)
  - [ ] 다크 모드에서 토글 클릭 → 라이트 모드 복귀 (KANBAN-015)
  - [ ] `bun run test` → KANBAN-015 spec 테스트 PASS
- **커밋**: `feat: add dark mode toggle with localStorage persistence`

---

### Task 8: localStorage 영속화 통합 검증

- **시나리오**: KANBAN-016, KANBAN-017
- **참조 규칙**: `.claude/skills/vercel-react-best-practices/rules/client-localstorage-schema.md`
- **구현 대상**:
  - Task 2에서 reducer + storage 유틸리티를 단위 테스트했으나, 실제 UI 컴포넌트와의 통합은 Task 3~5에서 이루어짐
  - 이 task는 추가 코드 구현 없이, KANBAN-016/017 spec 테스트가 PASS하는지 검증하는 통합 확인 단계
  - 만약 spec 테스트가 실패하면 BoardProvider의 localStorage 연동 로직을 디버깅·수정
- **수용 기준**:
  - [ ] 카드 "우유 사기" 추가 → localStorage에 저장됨 → 새로고침(remount) 후 카드 표시 (KANBAN-016)
  - [ ] 카드를 "Done"으로 이동 → localStorage에 저장됨 → 새로고침(remount) 후 "Done"에 표시 (KANBAN-017)
  - [ ] `bun run test` → KANBAN-016, KANBAN-017 spec 테스트 PASS
- **커밋**: `fix: ensure localStorage persistence works end-to-end` (수정 필요 시) 또는 커밋 없음 (이미 PASS인 경우)

---

### Task 9: JSON 내보내기/가져오기

- **시나리오**: KANBAN-018, KANBAN-019, KANBAN-020
- **참조 규칙**: `.claude/skills/shadcn/rules/composition.md` (Alert, Toast), `.claude/skills/shadcn/rules/styling.md`, `.claude/skills/shadcn/rules/icons.md`
- **구현 대상**:
  - `components/kanban/board-header.tsx` (기존 파일 수정) — Export 버튼 (JSON 파일 다운로드), Import 버튼 (파일 선택 트리거) 추가
  - `components/kanban/import-dialog.tsx` — ImportDialog: 파일 드롭/선택 영역 + Alert 컴포넌트로 경고 박스("Importing will replace all existing data.")
  - `lib/json-io.ts` — exportBoardToJson (Blob + download), importBoardFromJson (파일 파싱 + 검증)
  - 유효한 JSON → 기존 데이터 교체, 잘못된 JSON → sonner toast로 에러 표시
- **수용 기준**:
  - [ ] Export 클릭 → JSON 파일 다운로드 트리거됨 (KANBAN-018)
  - [ ] 유효한 JSON 가져오기 → 기존 데이터 교체, 가져온 카드 표시 (KANBAN-019)
  - [ ] 잘못된 JSON 가져오기 → 에러 토스트 표시, 기존 데이터 유지 (KANBAN-020)
  - [ ] `bun run test` → KANBAN-018~020 spec 테스트 PASS
- **커밋**: `feat: add JSON export/import with error handling via toast`

---

## 미결정 사항

- 없음 (모든 항목 결정 완료)
