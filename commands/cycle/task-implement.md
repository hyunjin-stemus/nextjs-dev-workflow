---
description: "task 구현 — 브랜치 생성 → TDD(Playwright) → 구현 → Playwright 자동화 테스트 → Playwright MCP 직접 검증 → 타입체크 → 린트 → 코드리뷰 → 사용자 컨펌"
---

# task 구현 — TDD → 코드 → Playwright 자동화 → MCP 직접 검증 → 타입체크 → 린트 → 코드리뷰

브랜치 생성부터 코드 리뷰 피드백 반영 후 사용자 컨펌까지 진행한다.

## Arguments

`$ARGUMENTS` — task ID (예: `2.1`).

## Prerequisites

`/task-start $ARGUMENTS`가 완료되어 plan 컨펌이 받아진 상태여야 한다.

## Steps

1. **브랜치 생성** — develop 브랜치를 최신화한 뒤 feature 브랜치를 생성한다.

   ```bash
   git checkout develop
   git pull origin develop
   git checkout -b feature/<task-id>-<slug>
   ```

   예: `feature/2.1-add-attendance-page`

2. **TDD: Playwright E2E 테스트 먼저 작성** — 구현 전에 요구사항을 검증하는 Playwright 테스트를 먼저 작성한다.
   - plan에서 정의한 사용자 시나리오를 `tests/e2e/<기능명>.spec.ts`에 작성한다.
   - dev 서버를 실행하고 테스트가 실제로 실패하는지 확인한다.

   ```bash
   pnpm dev &
   # 서버 기동 대기 후
   pnpm playwright test tests/e2e/<기능명>.spec.ts
   ```

   테스트가 실패(구현 미완성이므로 당연히 실패)하는 것을 확인한 후 구현 단계로 이동한다.

3. **구현** — plan에 따라 코드를 작성한다. 아래 에이전트를 활용한다.

   **ui-markup-specialist 에이전트** — 정적 마크업, Tailwind 스타일링, 레이아웃 구현

   이 에이전트를 호출할 때 반드시 아래 지침을 함께 전달한다:

   > **Figma 픽셀 정밀도 필수**: 구현 전 `mcp__plugin_figma_figma__get_design_context`로 파일키 `53QxHxnGIcJ2nJAG712BO3`, 해당 노드의 수치(width, height, padding, gap, border-radius, font-size, color)를 조회하고 Tailwind 클래스에 1:1 매핑한다. 어림짐작 구현 금지.
   >
   > **반응형 필수**: 모든 UI는 3개 브레이크포인트를 동시에 구현한다.
   > - **모바일 (기본, < 768px)**: 단일 컬럼, `px-4`, 터치 친화적 크기, 하단 BottomNav 고려 (`pb-[72px]`)
   > - **태블릿 (`md:`, ≥ 768px)**: 사이드바 레이아웃, `md:px-[120px]`, Figma 1440px 기준 여백 적용
   > - **데스크탑 (`lg:`, ≥ 1024px)**: 태블릿과 동일한 레이아웃 유지, 필요 시 max-width 제한
   >
   > **구현 후 검증**: Playwright MCP(`mcp__plugin_playwright_playwright__browser_take_screenshot`)로 1440×900 뷰포트 스크린샷을 찍고, `mcp__plugin_figma_figma__get_screenshot`으로 Figma 스크린샷과 비교한다. 차이가 있으면 수정 후 재비교. 스크린샷 비교 없이 "완료"를 보고하지 않는다.

   **nextjs-app-developer 에이전트** — App Router 구조, 라우팅, 레이아웃, 데이터 페칭, 성능 최적화

   이 에이전트를 호출할 때 반드시 아래 지침을 함께 전달한다:

   > **반응형 필수**: 레이아웃/라우팅 구조를 설계할 때 모바일/태블릿/데스크탑 3개 브레이크포인트를 고려한다. 모바일 BottomNav(`fixed bottom-0`, `z-50`)와 사이드바(`hidden md:flex`)의 공존 패턴을 유지한다.

4. **Playwright 자동화 테스트 재실행** — 구현 후 작성해둔 E2E 테스트가 통과하는지 확인한다.

   ```bash
   pnpm playwright test tests/e2e/<기능명>.spec.ts
   ```

   실패 시 수정 후 재실행. 모든 시나리오가 통과할 때까지 반복한다.
   Playwright 테스트는 모든 task에서 필수이며 생략하지 않는다.

5. **Playwright MCP로 직접 브라우저 검증** — 자동화 테스트 통과 후, Playwright MCP 도구(`mcp__plugin_playwright_playwright__*`)를 사용해 브라우저에서 직접 요구사항 충족 여부를 확인한다.

   dev 서버(http://localhost:3002)에 접속해 아래 항목을 직접 검증한다:
   - **골든 패스** — task의 핵심 사용자 시나리오를 처음부터 끝까지 인터랙션으로 수행
   - **UI 완성도** — 레이아웃, 스타일, 반응형 동작이 요구사항과 일치하는지 스크린샷으로 확인
   - **엣지 케이스** — 빈 상태, 에러 상태, 경계값 등 주요 예외 시나리오 직접 확인
   - **기존 기능 회귀** — 변경으로 인해 인접 페이지나 기능이 깨지지 않았는지 확인

   발견된 문제는 즉시 수정 후 재검증한다. 문제가 없으면 다음 단계로 진행한다.

6. **타입체크**:

   ```bash
   pnpm tsc --noEmit
   ```

   에러 발생 시 **반드시 수정** 후 다음 단계로. 무시하고 진행 금지.

7. **린트**:

   ```bash
   pnpm lint
   ```

   에러 발생 시 **반드시 수정** 후 다음 단계로. 무시하고 진행 금지.

8. **코드 리뷰** — `pr-review-toolkit` 에이전트를 병렬로 호출한다.

   아래 에이전트를 동시에 실행한다:
   - `pr-review-toolkit:code-reviewer` — 코드 품질, 스타일, Next.js 컨벤션 준수
   - `pr-review-toolkit:silent-failure-hunter` — 에러 처리, 폴백 동작
   - `pr-review-toolkit:type-design-analyzer` — 타입 설계 품질

   각 에이전트에 변경된 파일 목록과 task 컨텍스트를 전달한다.

   리뷰 완료 후 아래 형식으로 **모든 리뷰 항목을 빠짐없이 정리**해서 사용자에게 보여준다:

   **[코드 리뷰 결과 전체 목록]**

   | # | 파일 | 심각도 | 내용 | 반영 시 고려사항 |
   |---|------|--------|------|-----------------|
   | 1 | path/to/file.tsx | 🔴 critical | 지적 내용 | 반영 시 발생 가능한 문제점, side effect, 타입 변경 연쇄 영향, 테스트 수정 필요 여부 등 상세 기술 |
   | 2 | ... | 🟡 warning | ... | ... |

   심각도 기준: 🔴 critical (버그/보안/타입오류) / 🟡 warning (컨벤션/성능) / 🔵 suggestion (개선 제안)

   각 항목에 대해 **반영 시 발생 가능한 문제점과 고려사항**을 구체적으로 기술한다:
   - 다른 컴포넌트/파일로의 연쇄 영향
   - 타입 변경이 필요한 경우 영향 범위
   - SSR/CSR 동작 변경 가능성
   - React Query 캐시, hydration 관련 주의사항
   - 기존 기능 회귀 위험성

   **컨펌 대기**: 위 목록을 보여준 후 반드시 사용자에게 "어떤 항목을 반영할까요?" 확인을 받는다.
   컨펌 없이 코드를 수정하지 않는다.

9. **피드백 반영** — 컨펌된 항목만 수정한다. 컨펌되지 않은 항목은 건드리지 않는다.

10. **사용자 컨펌 대기** — 수정 내용을 요약하고 커밋 진행 여부를 묻는다. 사용자 명시적 확인 없이 커밋하지 않는다.

11. 컨펌 후 사용자에게 안내: PR 생성과 병합을 위해 `/task-ship <id>`를 호출한다.
