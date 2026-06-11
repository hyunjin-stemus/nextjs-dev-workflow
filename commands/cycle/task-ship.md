---
description: "task 배포 — 커밋 → PR 생성 → nextjs-react-reviewer 리뷰 → 병합 → done"
---

# task 배포 — 커밋 → PR → 리뷰 → 병합 → done

커밋부터 PR 생성, nextjs-react-reviewer 리뷰, 병합, task done 처리까지 진행한다.

## Arguments

`$ARGUMENTS` — task ID (예: `2.1`).

## Prerequisites

`/task-implement $ARGUMENTS`가 완료되어 사용자 컨펌이 받아진 상태여야 한다.

## Steps

1. **커밋 전 최종 검증** — 아래 순서대로 실행하고, 모두 통과한 경우에만 커밋으로 진행한다. 실패 시 반드시 수정 후 재실행한다.

   ```bash
   # 1) 린트
   pnpm lint

   # 2) 타입체크
   pnpm tsc --noEmit

   # 3) Playwright E2E 테스트
   pnpm playwright test tests/e2e/

   # 4) 프로덕션 빌드
   pnpm build
   ```

2. **커밋** — 변경된 파일을 스테이징하고 커밋한다.

   ```bash
   git add <변경 파일들>
   git commit -m "feat: <task 제목 요약> (task <id>)"
   ```

   커밋 메시지는 conventional commits 형식을 따른다 (`feat`, `fix`, `refactor`, `test`, `docs`, `style`).

3. **푸시**:

   ```bash
   git push -u origin feature/<task-id>-<slug>
   ```

4. **PR 생성** — base 브랜치를 `develop`으로 지정하고, 1단계에서 통과한 항목은 `[x]`로 체크한다:

   ```bash
   gh pr create \
     --base develop \
     --title "<task 제목>" \
     --body "$(cat <<'EOF'
   ## Summary
   - <변경 내용 불릿 요약>

   ## Task
   Task Master task ID: <id>

   ## Test plan
   - [x] 린트 통과 (`pnpm lint`)
   - [x] 타입체크 통과 (`pnpm tsc --noEmit`)
   - [x] Playwright E2E 테스트 통과
   - [x] 빌드 성공 (`pnpm build`)

   🤖 Generated with [Claude Code](https://claude.com/claude-code)
   EOF
   )"
   ```

5. **PR 리뷰** — `nextjs-react-reviewer` 서브에이전트를 호출한다.
   - PR URL 또는 PR 번호를 에이전트에 전달한다.
   - 리뷰 완료 후 아래 형식으로 **모든 리뷰 항목을 빠짐없이 정리**해서 사용자에게 보여준다:

     **[PR 코드 리뷰 결과 전체 목록]**

     | # | 파일 | 심각도 | 내용 | 반영 시 고려사항 |
     |---|------|--------|------|-----------------|
     | 1 | path/to/file.tsx | 🔴 critical | 지적 내용 | 반영 시 발생 가능한 문제점, side effect, SSR/CSR 영향, 추가 커밋 필요 여부 등 상세 기술 |
     | 2 | ... | 🟡 warning | ... | ... |

     심각도 기준: 🔴 critical (버그/보안/타입오류) / 🟡 warning (컨벤션/성능) / 🔵 suggestion (개선 제안)

   - 각 항목에 대해 **반영 시 발생 가능한 문제점과 고려사항**을 구체적으로 기술한다:
     - 다른 컴포넌트/파일로의 연쇄 영향
     - SSR/CSR 동작 변경 가능성
     - React Query 캐시, hydration 관련 주의사항
     - 병합 전 반드시 반영해야 하는 항목 여부

   - **컨펌 대기**: 위 목록을 보여준 후 반드시 사용자에게 "어떤 항목을 반영할까요?" 확인을 받는다.
     컨펌 없이 코드를 수정하지 않는다.

   - 리뷰 결과를 PR 댓글로 게시한다:

     ```bash
     gh pr comment <PR번호> --body "$(cat <<'EOF'
     ## 🤖 Next.js Code Review

     <리뷰 결과 전문>
     EOF
     )"
     ```

6. **PR 피드백 반영** — 컨펌된 항목만 수정한 후 추가 커밋을 생성하고 푸시한다. 컨펌되지 않은 항목은 건드리지 않는다.

7. **병합 컨펌 대기** — 사용자에게 병합 여부를 확인한다. **사용자 명시적 컨펌 없이 병합하지 않는다.**

8. **병합** (사용자 컨펌 후):

   ```bash
   gh pr merge <PR번호> --squash
   ```

9. **로컬 develop 브랜치 동기화**:

   ```bash
   git checkout develop
   git pull origin develop
   ```

10. **task done 처리**:

    ```bash
    task-master set-status --id=$ARGUMENTS --status=done
    ```

11. **task-master update-subtask** — 구현 중 발견한 중요 사항을 subtask에 기록한다.

    ```bash
    task-master update-subtask --id=$ARGUMENTS --prompt="구현 완료. 주요 결정 및 참고 사항: ..."
    ```

12. 사용자에게 안내: 다음 task를 진행하려면 `/task-start`를 호출하거나, 모든 task가 완료되면 `/cycle-finish`를 호출한다.
