---
name: nextjs-react-scaffold
description: |
  Use when the user wants to scaffold a brand-new Next.js + React project from scratch — empty folder, greenfield start, or "help me set up the initial structure." This skill creates a production-ready boilerplate in one shot: directory layout with proper layer separation (API, hooks, components, stores), TanStack Query data flow, shared error handling, Zustand patterns, and shadcn/ui conventions.

  Trigger on: new Next.js app setup, initial project structure, frontend boilerplate for a new full-stack project, "기본 틀", "초기 세팅", "새 프로젝트", "빈 폴더에 세팅".

  Do NOT trigger for: adding features, pages, or components to an existing project; deployment or CI/CD setup; dark mode or theme changes; individual component creation; modifying an existing Next.js codebase.

  The key signal is the user is starting fresh and wants the whole architecture decided upfront, not piecemeal additions.
---

# Next.js + React 프로젝트 스캐폴딩

## 이 스킬이 만드는 것

- 모듈형 디렉토리 구조 (`lib/api`, `lib/query`, `lib/stores`, `components/ui|common|layout`)
- 타입 안전 fetch wrapper + `ApiResponse<T>` 자동 언래핑
- TanStack Query 3계층 데이터 흐름 (`lib/api` → `hooks/use*` → 컴포넌트)
- 공통 예외처리 스택 (`ApiError` → `mapApiError` → `useErrorModal` + `ErrorModal`)
- zustand 스토어 패턴 (`lib/stores/`)
- shadcn/ui 설치 + CVA + `cn()` 컴포넌트 컨벤션

## Step 0 — context7로 최신 스택 조회 (필수, 첫 단계)

셋업 직전 **context7 MCP**로 현재 최신 설정 방법을 확인한다.
조회 결과로 아래 reference 스니펫을 보정한 뒤 실제 파일에 적용한다.

```
조회 대상:
1. Next.js App Router 현재 메이저 — create-next-app CLI 옵션, next.config 파일명
2. Tailwind CSS 현재 메이저 — v4인지 v3인지에 따라 설정 방식이 완전히 다름
3. shadcn/ui 최신 CLI — pnpm dlx shadcn@latest init 옵션
4. TanStack Query 현재 메이저 — QueryClient/Provider API
5. zustand 현재 메이저 — create() API, persist 미들웨어 사용법
```

## Step 1 — 사전 확인

```bash
pnpm --version   # 패키지 매니저 확인 (없으면 npm i -g pnpm)
node --version   # Node 18+ 권장
```

## Step 2 — 프로젝트 생성

```bash
pnpm create next-app <project-name> \
  --typescript \
  --tailwind \
  --eslint \
  --app \
  --import-alias "@/*" \
  --no-src-dir
```

> context7 조회 결과에서 최신 플래그 명을 확인해 적용한다.

## Step 3 — 기본 보일러플레이트 정리

```bash
# 불필요 파일 제거
rm -f app/page.tsx app/globals.css app/favicon.ico
# Next.js가 재생성하므로 내용만 교체
```

## Step 4 — 의존성 설치

context7에서 확인한 최신 버전으로 설치한다.

```bash
# 핵심 런타임
pnpm add @tanstack/react-query zustand sonner class-variance-authority clsx tailwind-merge

# shadcn 초기화 (최신 CLI 사용)
pnpm dlx shadcn@latest init
# 기본 컴포넌트 추가
pnpm dlx shadcn@latest add button dialog tooltip sonner
```

## Step 5 — 구조 및 파일 생성

아래 순서대로 reference 파일을 읽어 각 레이어를 만든다.

| 순서 | 파일 | 만드는 것 |
|---|---|---|
| 1 | `references/01-architecture.md` | 디렉토리 구조 + 경로 alias + 네이밍 컨벤션 |
| 2 | `references/02-data-layer.md` | `lib/api/types.ts`, `lib/api/client.ts`, `lib/api/base-url.ts`, `lib/query/` 전체 + `app/providers.tsx` |
| 3 | `references/03-error-handling.md` | `lib/api/error-messages.ts`, `hooks/useErrorModal.ts`, `components/common/ErrorModal.tsx`, `lib/monitoring/error-tracker.ts`, `app/error.tsx` |
| 4 | `references/04-components.md` | `app/globals.css` Tailwind 설정, `lib/utils.ts`, 컴포넌트 컨벤션 문서 |
| 5 | `references/05-state.md` | `lib/stores/ui-store.ts` zustand 슬라이스 |

각 reference 상단의 **"⚠️ context7 보정"** 항목을 Step 0 결과로 먼저 업데이트한 뒤 파일을 생성한다.

## Step 5.5 — 예시 도메인 모듈 생성 (필수)

3계층 데이터 흐름이 실제로 어떻게 연결되는지 보여주는 **예시 도메인 모듈**을 반드시 만든다.
이게 없으면 팀원이 "API 추가할 때 어디에 뭘 만들면 되는지" 알기 어렵다.

프로젝트 성격에 맞는 도메인 이름(예: `posts`, `users`, `products`)을 하나 골라 아래 파일을 생성:

```
lib/[domain]/
  types.ts        — 도메인 타입 (예: Post, CreatePostDto)
  constants.ts    — 페이지 사이즈 등 상수
  index.ts        — 배럴 re-export

lib/api/[domain].ts   — apiFetch 래핑 함수 (fetchXxx, createXxx, deleteXxx)

hooks/
  use[Domain]s.ts       — useQuery 래퍼 (목록 조회)
  use[Domain]Detail.ts  — useQuery 래퍼 (단건 조회)
  useCreate[Domain].ts  — useMutation 래퍼 (생성)
```

예시 컴포넌트는 만들지 않아도 된다. 훅이 어떻게 API 함수를 감싸고 queryKeys를 사용하는지만 보여주면 충분하다.

## Step 6 — RootLayout 배선

`app/layout.tsx`에 `QueryProvider`, Toaster, 폰트를 배선한다. 자세한 패턴은 `02-data-layer.md` 참조.

## Step 7 — 검증

```bash
pnpm tsc --noEmit   # 타입 에러 없어야 함
pnpm lint           # ESLint 통과
pnpm build          # 빌드 성공
```

3개 모두 통과하면 스캐폴딩 완료.

---

## 핵심 아키텍처 원칙

> reference 파일을 읽기 전에 이 원칙을 이해하면 "왜 이렇게 만드는지" 맥락이 명확해진다.

**3계층 데이터 흐름**
```
lib/api/*       →  hooks/use*          →  컴포넌트
(fetch + 언래핑)   ('use client', RQ래퍼)   (훅만 소비, API 모름)
```

**예외처리 스택**
```
ApiError  →  mapApiError(err, labels)  →  useErrorModal/ErrorModal
          →  route error.tsx boundary
          →  captureError() (모니터링 확장점)
```

**컴포넌트 분류**
```
components/
  ui/       — shadcn primitive (lowercase 파일명)
  common/   — 도메인 무관 재사용 (PascalCase)
  layout/   — Header, NavBar, 레이아웃 셸
  [domain]/ — 도메인별 (PascalCase)
```

**lib 도메인 모듈**
```
lib/[domain]/
  constants.ts
  types.ts
  [logic files]
  index.ts   ← 배럴 re-export
```
