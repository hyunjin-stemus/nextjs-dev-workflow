# 02 — 데이터 레이어: fetch wrapper + TanStack Query 3계층

## ⚠️ context7 보정

이 파일을 적용하기 전, SKILL.md Step 0 결과로 확인:
- TanStack Query 현재 메이저: `QueryClient`, `isServer`, `defaultShouldDehydrateQuery` import 위치
- `QueryClientProvider` / `HydrationBoundary` API 변경 여부

---

## 3계층 데이터 흐름 원칙

```
lib/api/*        →   hooks/use*           →   컴포넌트
fetch + 언래핑      'use client' + RQ 래퍼     훅만 소비
ApiError throw      queryKeys 팩토리 사용       API 세부사항 모름
```

컴포넌트는 "어디서 데이터가 오는지" 알 필요 없다. 훅만 호출한다.

---

## 1. `lib/api/base-url.ts`

```ts
export const API_BASE_URL =
  process.env.NEXT_PUBLIC_API_BASE_URL ?? 'http://localhost:3000/api/v1'
```

---

## 2. `lib/api/types.ts`

```ts
/** ISO 8601 UTC 문자열. 예: '2026-05-01T00:00:00.000Z' */
export type ISODateString = string

export interface ApiResponse<T> {
  success: boolean
  data: T | null
  message: string
  timestamp?: string
  path?: string
}
```

백엔드 `ApiResponse<T>` 래핑 구조를 여기서 정의한다.
각 도메인 타입은 `lib/api/[domain].ts` 또는 `lib/[domain]/types.ts`에 colocate.

---

## 3. `lib/api/client.ts`

`ApiResponse<T>`를 자동 언래핑해 `T`만 반환한다.
실패 시 `ApiError`를 throw — 호출부는 try/catch 없이 react-query에 위임하면 된다.

인증 토큰이 있으면 `Authorization: Bearer` 헤더를 자동 주입한다.
`getAuthToken()`은 이 레포에서는 auth-store에서 가져오지만, 프로젝트마다 구현이 다르므로
**확장점**으로 남겨두고 필요시 교체한다.

```ts
import type { ApiResponse } from './types'
import { API_BASE_URL } from './base-url'

// ─── 인증 토큰 확장점 ─────────────────────────────────────────────────────────
// 프로젝트 인증 방식에 맞게 구현. 예: zustand store / cookie / sessionStorage
function getAuthToken(): string | null {
  return null // TODO: 인증 구현 시 교체
}

// ─── ApiError ────────────────────────────────────────────────────────────────
export class ApiError extends Error {
  constructor(
    public status: number,
    message: string,
    public payload: ApiResponse<unknown> | null = null,
  ) {
    super(message)
    this.name = 'ApiError'
    Object.setPrototypeOf(this, ApiError.prototype)
  }
}

// ─── 헤더 빌드 ────────────────────────────────────────────────────────────────
function buildHeaders(init: RequestInit): Headers {
  const headers = new Headers(init.headers)
  if (
    !headers.has('Content-Type') &&
    init.body !== undefined &&
    !(init.body instanceof FormData)
  ) {
    headers.set('Content-Type', 'application/json')
  }
  const token = getAuthToken()
  if (token) headers.set('Authorization', `Bearer ${token}`)
  return headers
}

// ─── apiFetch<T> ─────────────────────────────────────────────────────────────
// ApiResponse<T>를 언래핑해 T만 반환. 실패 시 ApiError throw.
export async function apiFetch<T>(path: string, init: RequestInit = {}): Promise<T> {
  const res = await fetch(`${API_BASE_URL}${path}`, {
    ...init,
    headers: buildHeaders(init),
    credentials: 'include',
  })

  let body: ApiResponse<T> | null = null
  try {
    body = (await res.json()) as ApiResponse<T>
  } catch {
    body = null
  }

  if (!res.ok || !body || body.success === false) {
    throw new ApiError(
      res.status,
      body?.message ?? res.statusText,
      body as ApiResponse<unknown> | null,
    )
  }
  if (body.data == null) {
    throw new ApiError(res.status, `Unexpected null data from ${path}`, body as ApiResponse<unknown>)
  }
  return body.data as T
}

// ─── apiFetchVoid ─────────────────────────────────────────────────────────────
// 응답 본문 없는 API (DELETE, POST 204 등)
export async function apiFetchVoid(path: string, init: RequestInit = {}): Promise<void> {
  const res = await fetch(`${API_BASE_URL}${path}`, {
    ...init,
    headers: buildHeaders(init),
    credentials: 'include',
  })

  if (!res.ok) {
    let body: ApiResponse<unknown> | null = null
    try {
      body = (await res.json()) as ApiResponse<unknown>
    } catch {
      // JSON 없으면 무시
    }
    throw new ApiError(res.status, body?.message ?? res.statusText, body)
  }
}
```

도메인 API 모듈 예 (`lib/api/posts.ts`):
```ts
import { apiFetch, apiFetchVoid } from './client'
import type { Post, CreatePostDto } from '@/lib/posts/types'

export async function fetchPosts(page: number): Promise<Post[]> {
  return apiFetch<Post[]>(`/posts?page=${page}`)
}

export async function createPost(dto: CreatePostDto): Promise<Post> {
  return apiFetch<Post>('/posts', {
    method: 'POST',
    body: JSON.stringify(dto),
  })
}

export async function deletePost(id: string): Promise<void> {
  return apiFetchVoid(`/posts/${id}`, { method: 'DELETE' })
}
```

---

## 4. `lib/query/query-client.ts`

서버는 매 요청마다 새 인스턴스, 브라우저는 싱글톤.
RSC dehydrate 시 pending 쿼리도 포함해 hydration 플리커를 방지한다.

```ts
import { QueryClient, isServer, defaultShouldDehydrateQuery } from '@tanstack/react-query'

function makeQueryClient() {
  return new QueryClient({
    defaultOptions: {
      queries: {
        staleTime: 60_000,
        refetchOnWindowFocus: false,
        retry: 1,
      },
      dehydrate: {
        shouldDehydrateQuery: (q) =>
          defaultShouldDehydrateQuery(q) || q.state.status === 'pending',
      },
    },
  })
}

let browserQueryClient: QueryClient | undefined

export function getQueryClient() {
  if (isServer) return makeQueryClient()
  if (!browserQueryClient) browserQueryClient = makeQueryClient()
  return browserQueryClient
}
```

---

## 5. `lib/query/query-keys.ts`

모든 query key를 **한 곳에** 팩토리 함수로 정의한다.
`as const` 튜플을 반환하므로 타입 추론이 정확하고, 
namespace 기반 무효화(`queryKeys.postsBase`)도 가능하다.

```ts
export const queryKeys = {
  // 예시 — 프로젝트 도메인에 맞게 추가
  postsBase: ['posts'] as const,
  posts: (page: number) => ['posts', page] as const,
  postDetail: (id: string) => ['posts', 'detail', id] as const,
  userProfile: () => ['user', 'profile'] as const,
} as const
```

---

## 6. `app/providers.tsx`

```ts
'use client'

import { useState, type ReactNode } from 'react'
import { QueryClientProvider } from '@tanstack/react-query'
import { getQueryClient } from '@/lib/query/query-client'

export function QueryProvider({ children }: { children: ReactNode }) {
  const [client] = useState(() => getQueryClient())
  return (
    <QueryClientProvider client={client}>
      {children}
    </QueryClientProvider>
  )
}
```

---

## 7. `app/layout.tsx` (배선)

```tsx
import type { ReactNode } from 'react'
import type { Metadata } from 'next'
import { QueryProvider } from './providers'
import { Toaster } from '@/components/ui/sonner'
import './globals.css'

export const metadata: Metadata = {
  title: 'My App',
  description: 'App description',
}

export default function RootLayout({ children }: { children: ReactNode }) {
  return (
    <html lang='ko'>
      <body>
        <QueryProvider>
          {children}
          <Toaster position='top-right' richColors />
        </QueryProvider>
      </body>
    </html>
  )
}
```

---

## 8. 훅 패턴 (`hooks/useXxx.ts`)

한 훅 = 한 query 또는 한 mutation. 컴포넌트가 fetch 세부사항을 알 필요 없다.

```ts
// hooks/usePosts.ts
'use client'

import { useQuery } from '@tanstack/react-query'
import { fetchPosts } from '@/lib/api/posts'
import { queryKeys } from '@/lib/query/query-keys'

export function usePosts(page: number) {
  return useQuery({
    queryKey: queryKeys.posts(page),
    queryFn: () => fetchPosts(page),
    staleTime: 30_000,
  })
}
```

```ts
// hooks/useCreatePost.ts
'use client'

import { useMutation, useQueryClient } from '@tanstack/react-query'
import { createPost } from '@/lib/api/posts'
import { queryKeys } from '@/lib/query/query-keys'

export function useCreatePost() {
  const qc = useQueryClient()
  return useMutation({
    mutationFn: createPost,
    onSuccess: () => {
      // 목록 전체 무효화
      qc.invalidateQueries({ queryKey: queryKeys.postsBase })
    },
  })
}
```

컴포넌트에서:
```tsx
'use client'

import { usePosts } from '@/hooks/usePosts'
import { useErrorModal } from '@/hooks/useErrorModal'
import { ErrorModal } from '@/components/common/ErrorModal'
import { mapApiError } from '@/lib/api/error-messages'

export function PostList({ page }: { page: number }) {
  const { data: posts, error } = usePosts(page)
  const { isOpen, message, showError, close } = useErrorModal()

  // react-query error는 error boundary로 올라가거나 여기서 처리
  if (error) {
    const msg = mapApiError(error, {
      notFound: '게시글을 찾을 수 없습니다.',
      fallback: '게시글을 불러오지 못했습니다.',
    })
    showError(msg)
  }

  return (
    <>
      {posts?.map((p) => <div key={p.id}>{p.title}</div>)}
      <ErrorModal open={isOpen} message={message} onClose={close} />
    </>
  )
}
```
