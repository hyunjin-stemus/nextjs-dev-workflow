# 01 — 아키텍처 · 디렉토리 구조 · 컨벤션

## ⚠️ context7 보정

이 파일을 적용하기 전, SKILL.md Step 0 결과로 아래 항목을 확인한다:
- `@/*` alias 설정 방식 (create-next-app 기본 포함 여부)
- ESLint flat config가 현재 Next.js 버전에서 default인지 확인

---

## 목표 디렉토리 구조

```
project-root/
├── app/
│   ├── layout.tsx              # RootLayout (폰트, QueryProvider, Toaster)
│   ├── globals.css             # Tailwind + shadcn CSS variables
│   ├── providers.tsx           # QueryProvider ('use client')
│   ├── error.tsx               # 루트 error boundary (선택)
│   ├── (auth)/                 # 비로그인 라우트 그룹 (로그인 페이지 등)
│   │   └── layout.tsx
│   └── (authenticated)/        # 인증 필요 라우트 그룹
│       └── layout.tsx
├── components/
│   ├── ui/                     # shadcn primitive (lowercase 파일명)
│   ├── common/                 # 도메인 무관 재사용 컴포넌트 (PascalCase)
│   └── layout/                 # Header, NavBar, 레이아웃 셸 (PascalCase)
├── hooks/                      # 커스텀 훅 — flat 구조, useXxx.ts camelCase
├── lib/
│   ├── utils.ts                # cn() 헬퍼 (shadcn 표준)
│   ├── utils/                  # 순수 유틸 (date.ts, url.ts 등)
│   ├── api/
│   │   ├── base-url.ts         # API_BASE_URL 단일 export
│   │   ├── types.ts            # ApiResponse<T>, 공통 타입
│   │   └── client.ts           # fetch wrapper, ApiError
│   ├── query/
│   │   ├── query-client.ts     # getQueryClient() 싱글톤
│   │   └── query-keys.ts       # 중앙 query key 팩토리
│   ├── stores/                 # zustand 슬라이스
│   └── [domain]/               # 도메인별 모듈 (constants, types, index 배럴)
├── public/
│   └── fonts/                  # 로컬 폰트
└── tests/                      # vitest + playwright (소스 구조 미러링)
    ├── unit/
    └── e2e/
```

---

## 경로 Alias

`tsconfig.json`의 `paths`에 `@/*` → `./*` 매핑이 있어야 한다.
create-next-app --import-alias "@/*" 옵션으로 자동 설정되므로 별도 추가 불필요.

**사용 패턴**:
```ts
import { Button } from '@/components/ui/button'  // 항상 절대 경로
import { cn } from '@/lib/utils'
import { apiFetch } from '@/lib/api/client'
import { queryKeys } from '@/lib/query/query-keys'
```

**같은 도메인 하위 컴포넌트**는 상대경로 허용:
```ts
// components/learning/SessionCard.tsx 안에서
import { SessionDetail } from './SessionDetail'  // 상대경로 OK
```

---

## 네이밍 컨벤션

| 대상 | 컨벤션 | 예시 |
|---|---|---|
| 컴포넌트 파일 | PascalCase | `SessionCard.tsx` |
| shadcn ui 파일 | lowercase | `button.tsx`, `dialog.tsx` |
| 훅 파일 | camelCase | `useAttendance.ts` |
| lib 유틸 | camelCase | `date.ts`, `grade-bucket.ts` |
| 상수 | SCREAMING_SNAKE | `export const API_TIMEOUT = 10_000` |
| 타입/인터페이스 | PascalCase | `ApiResponse<T>`, `UserProfile` |
| zustand 슬라이스 | kebab-case 파일 | `ui-store.ts`, `user-store.ts` |

---

## 도메인 모듈 패턴 (`lib/[domain]/`)

서로 연관된 상수·타입·로직을 같은 폴더에 모으고 배럴 `index.ts`로 재노출한다.
변경 시 하나의 폴더만 건드리면 된다.

```
lib/dashboard/
  constants.ts   — PAGE_SIZE, REFRESH_INTERVAL 등
  types.ts       — DashboardSummary, CardData 등
  utils.ts       — 도메인 순수 계산 함수
  index.ts       — re-export 배럴
```

```ts
// lib/dashboard/index.ts
export * from './constants'
export * from './types'
export * from './utils'
```

컴포넌트에서:
```ts
import { PAGE_SIZE, type DashboardSummary } from '@/lib/dashboard'
```

---

## 그룹 라우트로 레이아웃 분리

URL 경로에 영향 없이 레이아웃을 달리 적용할 때 App Router 그룹 라우트를 사용한다.

```
app/
  (auth)/
    layout.tsx      ← 비로그인 레이아웃 (로그인 페이지용)
    login/page.tsx
  (authenticated)/
    layout.tsx      ← 인증된 사용자 레이아웃 (Header + Sidebar)
    dashboard/page.tsx
    settings/page.tsx
```

인증 처리는 이 스킬 범위 외 — `(authenticated)/layout.tsx`에 인증 체크 훅 자리를 비워두면 된다.

---

## 코딩 스타일 요약

- 들여쓰기: **2칸**
- 문자열: **작은따옴표** `'`
- `'use client'` 선언: 서버 컴포넌트가 기본이므로 클라이언트 기능이 필요한 파일에만 명시
- `interface` 우선 (단순 객체 형태), 유니언/교차타입은 `type`
- `export default`는 page/layout만, 나머지는 named export
- 매직 넘버 금지 — 상수(`constants.ts`)나 환경변수로 추출
