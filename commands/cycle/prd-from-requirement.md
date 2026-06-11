---
description: "PRD 생성 및 task 분할 — docs/REQUIREMENT.md → prd-writer → prd-validator → 사용자 승인 → .taskmaster/docs/prd.md → task-master"
---

# PRD 생성 및 task 분할

`docs/REQUIREMENT.md`를 읽어 PRD를 생성하고 검수를 거쳐 사용자 승인 후 `.taskmaster/docs/prd.md`를 작성한 뒤 task-master로 task를 분할한다.

## Steps

1. `docs/REQUIREMENT.md` 전체 내용을 읽는다.

2. **prd-writer 서브에이전트**를 호출해 PRD 초안을 생성한다.
   - 에이전트에 REQUIREMENT.md 전체 내용을 전달한다.
   - 에이전트가 반환한 PRD 초안 내용을 수령한다.

3. **prd-validator 서브에이전트**를 호출해 PRD 초안을 기술 검수한다.
   - 에이전트에 PRD 초안을 전달한다.
   - 검수 결과(문제점, 개선 제안)를 수령한다.
   - 검수 의견을 반영해 PRD 초안을 보완한다.

4. 보완된 PRD 전문을 사용자에게 보여주고 **검토 및 승인**을 요청한다.
   - 사용자 승인 없이 절대 다음 단계로 진행하지 않는다.
   - 수정 요청이 있으면 반영 후 다시 보여준다.

5. 사용자가 승인하면 `.taskmaster/docs/prd.md`에 PRD를 작성한다.
   - 디렉터리가 없으면 생성한다.
   - 아래 구조를 포함해야 한다:
     - `# Overview` — 프로젝트 목적과 배경
     - `# Goals` — 달성 목표 (불릿 리스트)
     - `# Technical Requirements` — 기술 스택, 제약 조건
     - `# Features` — 기능 단위 섹션 (task-master가 task로 분할할 단위)
     - `# Out of Scope` — 이번 사이클에서 제외할 내용
     - `# Success Metrics` — 완료 판단 기준

6. 아래 명령을 순서대로 실행한다.

   ```bash
   task-master parse-prd .taskmaster/docs/prd.md
   ```

   parse-prd 완료 후 생성된 task 목록 확인:

   ```bash
   task-master list
   ```

7. task가 생성되었으면 다음을 실행한다.

   ```bash
   task-master analyze-complexity --research
   task-master expand --all --research
   ```

8. **develop 브랜치 준비** — task 분할이 완료된 시점에 develop 브랜치를 생성하거나 체크아웃한다.

   ```bash
   git fetch origin
   # develop 브랜치가 이미 있으면 체크아웃, 없으면 main 기반으로 생성
   git checkout develop 2>/dev/null || git checkout -b develop main
   git push -u origin develop 2>/dev/null || true
   ```

   사용자에게 현재 브랜치가 `develop`임을 확인시킨다.

9. 완료 후 사용자에게 안내한다.
   - 생성된 task 수
   - 현재 브랜치: `develop` (모든 feature 브랜치는 이 브랜치 기반으로 생성됨)
   - 다음 작업 시작: `/task-start` 또는 `/task-start <id>` 호출
