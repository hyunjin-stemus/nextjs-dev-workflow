# 05 — 클라이언트 전역 상태 (zustand)

## ⚠️ context7 보정

이 파일을 적용하기 전, SKILL.md Step 0 결과로 확인:
- zustand 현재 메이저: `create()` API, `persist` 미들웨어 import 위치
- `useSyncExternalStore` 기반 접근 vs zustand — 현 버전 권장 패턴

---

## 언제 zustand를 쓰는가

**react-query가 커버하지 못하는 클라이언트 전용 UI 상태**에 사용한다.

| react-query | zustand |
|---|---|
| 서버 데이터 (API 응답) | UI 상태 (사이드바 open/close) |
| 캐시/무효화 | 사용자 선택 필터 |
| 로딩/에러 상태 | 토스트 큐 |
| 서버 동기화 | 페이지 간 공유 클라이언트 상태 |

> 대부분의 상태는 react-query로 충분하다. zustand는 "서버와 무관한 클라이언트 UI 상태"에만.

---

## 1. 기본 슬라이스 패턴

도메인별로 파일을 분리한다. 파일명은 kebab-case, `lib/stores/[domain]-store.ts`.

```ts
// lib/stores/ui-store.ts
import { create } from 'zustand'

interface UiState {
  sidebarOpen: boolean
  setSidebarOpen: (open: boolean) => void
  toggleSidebar: () => void
}

export const useUiStore = create<UiState>((set) => ({
  sidebarOpen: true,
  setSidebarOpen: (open) => set({ sidebarOpen: open }),
  toggleSidebar: () => set((s) => ({ sidebarOpen: !s.sidebarOpen })),
}))
```

---

## 2. 셀렉터 사용 — 불필요한 리렌더 방지

전체 store를 구독하지 않고 필요한 값만 선택한다.

```tsx
'use client'
import { useUiStore } from '@/lib/stores/ui-store'

// 좋은 예 — 필요한 값만 구독
function Sidebar() {
  const sidebarOpen = useUiStore((s) => s.sidebarOpen)
  return <aside className={sidebarOpen ? 'w-64' : 'w-0'}>...</aside>
}

// 나쁜 예 — 전체 store 구독 → 모든 상태 변경에 리렌더
function Sidebar() {
  const store = useUiStore()  // 지양
  return <aside className={store.sidebarOpen ? 'w-64' : 'w-0'}>...</aside>
}
```

---

## 3. persist 미들웨어 (localStorage/sessionStorage 유지)

사용자 설정 등 새로고침 후에도 유지해야 하는 상태.

```ts
// lib/stores/preferences-store.ts
import { create } from 'zustand'
import { persist } from 'zustand/middleware'

interface PreferencesState {
  theme: 'light' | 'dark' | 'system'
  setTheme: (theme: PreferencesState['theme']) => void
}

export const usePreferencesStore = create<PreferencesState>()(
  persist(
    (set) => ({
      theme: 'system',
      setTheme: (theme) => set({ theme }),
    }),
    {
      name: 'app-preferences',  // localStorage key
      // sessionStorage 사용 시:
      // storage: createJSONStorage(() => sessionStorage),
    }
  )
)
```

> persist 미들웨어 import 위치를 context7로 확인 — zustand 버전마다 다를 수 있다.

---

## 4. 복합 slices 패턴 (규모가 커질 때)

store가 많아지면 여러 슬라이스를 하나의 store에 합칠 수 있다.
단, 대부분의 경우 슬라이스별 독립 파일이 더 간단하다.

```ts
// lib/stores/app-store.ts (선택적 패턴)
import { create } from 'zustand'

interface UiSlice {
  sidebarOpen: boolean
  setSidebarOpen: (v: boolean) => void
}

interface FilterSlice {
  selectedCategory: string | null
  setCategory: (c: string | null) => void
}

type AppState = UiSlice & FilterSlice

export const useAppStore = create<AppState>((set) => ({
  // UI slice
  sidebarOpen: true,
  setSidebarOpen: (v) => set({ sidebarOpen: v }),
  // Filter slice
  selectedCategory: null,
  setCategory: (c) => set({ selectedCategory: c }),
}))
```

---

## 5. SSR 안전 사용

Next.js App Router + RSC 환경에서 zustand는 클라이언트 전용이다.

**`'use client'` 컴포넌트에서만 사용**:
```tsx
'use client'
import { useUiStore } from '@/lib/stores/ui-store'

export function NavBar() {
  const sidebarOpen = useUiStore((s) => s.sidebarOpen)
  // ...
}
```

서버 컴포넌트에서 zustand store에 직접 접근 금지.
서버에서 필요한 데이터는 react-query prefetch 또는 props로 내려보낸다.

---

## 6. 초기 구조 제안

프로젝트 초기에는 스토어를 최소로 유지한다.

```
lib/stores/
  ui-store.ts          # 레이아웃/UI 상태 (사이드바, 모달 등)
  # 필요 시 추가:
  # preferences-store.ts   # 사용자 설정 (persist 필요 시)
  # filter-store.ts        # 검색/필터 상태
```

> 상태가 한 페이지에서만 쓰인다면 `useState`로 관리.
> 여러 컴포넌트가 공유해야 할 때 zustand로 이동.
