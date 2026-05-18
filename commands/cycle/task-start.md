---
description: "task 시작 — task 선택 + plan mode 진입 + 사용자 컨펌"
---

# task 시작 — plan 작성 및 컨펌

작업할 task를 선택하고 plan mode에서 구현 계획을 수립한 뒤 사용자 컨펌을 받는다.

## Arguments

`$ARGUMENTS` — task ID (예: `2.1`). 생략 시 `task-master next`로 자동 선택.

## Steps

1. task 정보를 조회한다.

   - `$ARGUMENTS`가 있으면:
     ```bash
     task-master show $ARGUMENTS
     ```
   - 없으면:
     ```bash
     task-master next
     task-master show <조회된 ID>
     ```

2. task 상태를 `in-progress`로 변경한다.

   ```bash
   task-master set-status --id=<id> --status=in-progress
   ```

3. **plan mode를 진입**하여 구현 계획을 작성한다. 아래 항목을 모두 포함한다.
   - 구현 접근법 (어떤 컴포넌트/페이지/훅을 만들 것인지)
   - 변경 대상 파일 목록
   - TDD 테스트 전략 — Playwright E2E 시나리오를 먼저 정의한다 (어떤 사용자 행동을 검증할 것인지)
   - ui-markup-specialist / nextjs-app-developer 에이전트 활용 계획
   - 예상 위험 요소
   - 타입체크/린트 사전 확인 사항

4. `ExitPlanMode`로 사용자 컨펌을 받는다. 컨펌 없이 구현 단계로 진행하지 않는다.

5. 컨펌 후 사용자에게 안내: 구현을 시작하려면 `/task-implement <id>`를 호출하거나 직접 구현 진행을 요청한다.
