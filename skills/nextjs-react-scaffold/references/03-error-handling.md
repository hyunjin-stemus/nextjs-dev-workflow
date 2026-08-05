# 03 — 공통 예외처리 스택

## ⚠️ context7 보정

이 파일을 적용하기 전, SKILL.md Step 0 결과로 확인:
- shadcn `Dialog` 컴포넌트 API — `open`, `onOpenChange`, `DialogContent` props 변경 여부
- Next.js App Router `error.tsx` 인터페이스 변경 여부

---

## 예외처리 스택 전체 흐름

```
apiFetch 실패
  ↓
ApiError throw  (status, message, payload)
  ↓
mapApiError(err, labels)  →  상태코드별 한글 메시지로 변환
  ↓
useErrorModal.showError(msg)  →  ErrorModal Dialog 표시
                              OR
  ↓
react-query가 error를 boundary로 전파
  ↓
app/[segment]/error.tsx  →  라우트 단위 회복 UI
  ↓
captureError(err)  →  모니터링 확장점 (Sentry 등 연결)
```

---

## 1. `lib/api/error-messages.ts`

`ApiError` 상태코드를 사람이 읽을 수 있는 한글 메시지로 변환한다.
호출마다 어떤 상황인지 `labels`로 커스터마이징한다.

```ts
import { ApiError } from './client'

export interface ServiceErrorLabels {
  unauthorized?: string    // 401
  forbidden?: string       // 403
  notFound?: string        // 404
  server?: string          // 5xx
  fallback: string         // 그 외 모든 경우 (필수)
}

const DEFAULT_SERVER =
  '서버에 일시적인 오류가 발생했습니다. 잠시 후 다시 시도해 주세요.'

export function mapApiError(err: unknown, labels: ServiceErrorLabels): string {
  if (err instanceof ApiError) {
    if (err.status === 401 && labels.unauthorized) return labels.unauthorized
    if (err.status === 403 && labels.forbidden) return labels.forbidden
    if (err.status === 404 && labels.notFound) return labels.notFound
    if (err.status >= 500) return labels.server ?? DEFAULT_SERVER
  }
  return labels.fallback
}
```

사용 예:
```ts
const message = mapApiError(error, {
  forbidden: '접근 권한이 없습니다.',
  notFound: '데이터를 찾을 수 없습니다.',
  fallback: '오류가 발생했습니다. 다시 시도해 주세요.',
})
showError(message)
```

---

## 2. `hooks/useErrorModal.ts`

모달 open/close 상태를 캡슐화하는 얇은 훅. 컴포넌트가 useState로 직접 관리하지 않아도 된다.

```ts
'use client'

import { useState, useCallback } from 'react'

export function useErrorModal() {
  const [isOpen, setIsOpen] = useState(false)
  const [message, setMessage] = useState('')

  const showError = useCallback((msg: string) => {
    setMessage(msg)
    setIsOpen(true)
  }, [])

  const close = useCallback(() => {
    setIsOpen(false)
  }, [])

  return { isOpen, message, showError, close }
}
```

---

## 3. `components/common/ErrorModal.tsx`

shadcn Dialog 기반. 프로젝트 디자인 토큰에 맞게 스타일을 조정한다.

```tsx
'use client'

import {
  Dialog,
  DialogContent,
  DialogDescription,
  DialogTitle,
} from '@/components/ui/dialog'

export interface ErrorModalProps {
  open: boolean
  title?: string
  message: string
  onClose: () => void
}

export function ErrorModal({
  open,
  title = '오류',
  message,
  onClose,
}: ErrorModalProps) {
  return (
    <Dialog open={open} onOpenChange={(v) => { if (!v) onClose() }}>
      <DialogContent className='max-w-sm'>
        <DialogTitle>{title}</DialogTitle>
        <DialogDescription>{message}</DialogDescription>
        <button
          type='button'
          onClick={onClose}
          className='mt-4 w-full rounded-lg bg-neutral-800 py-3 text-white font-semibold hover:opacity-90 transition-opacity'
        >
          확인
        </button>
      </DialogContent>
    </Dialog>
  )
}
```

---

## 4. 모니터링 확장점 — `lib/monitoring/error-tracker.ts`

Sentry 등 외부 모니터링 연동 자리. 아직 연동 전이면 개발 환경에서는 console에 출력하고,
이후 Sentry/DataDog 등을 달면 이 파일만 교체하면 된다.

```ts
interface CaptureOptions {
  tags?: Record<string, string>
  extra?: Record<string, unknown>
}

export function captureError(err: unknown, options?: CaptureOptions): void {
  if (process.env.NODE_ENV === 'development') {
    console.error('[captureError]', err, options)
    return
  }
  // TODO: Sentry 연동 시 교체
  // Sentry.captureException(err, { tags: options?.tags, extra: options?.extra })
}

export function captureMessage(msg: string, options?: CaptureOptions): void {
  if (process.env.NODE_ENV === 'development') {
    console.warn('[captureMessage]', msg, options)
    return
  }
  // TODO: Sentry.captureMessage(msg, { tags: options?.tags, extra: options?.extra })
}
```

---

## 5. `app/error.tsx` — 루트 error boundary

react-query throwOnError 또는 서버 컴포넌트 에러가 올라오면 여기서 잡는다.
라우트 segment 단위로 `[segment]/error.tsx`를 만들어 granular하게 처리할 수도 있다.

```tsx
'use client'

import { useEffect } from 'react'
import { captureError } from '@/lib/monitoring/error-tracker'

interface ErrorProps {
  error: Error & { digest?: string }
  reset: () => void
}

export default function GlobalError({ error, reset }: ErrorProps) {
  useEffect(() => {
    captureError(error, { tags: { layer: 'error-boundary/root' } })
  }, [error])

  return (
    <div className='flex min-h-screen flex-col items-center justify-center gap-4'>
      <h2 className='text-lg font-semibold'>문제가 발생했습니다</h2>
      <p className='text-sm text-neutral-500'>{error.message}</p>
      <button
        type='button'
        onClick={reset}
        className='rounded-lg bg-neutral-800 px-6 py-2 text-white text-sm'
      >
        다시 시도
      </button>
    </div>
  )
}
```

---

## 사용 패턴 요약

```
1. apiFetch → ApiError 자동 throw
2. 컴포넌트에서 error 처리 필요 시:
     mapApiError(error, labels) → showError(msg)
     → ErrorModal 표시
3. 중요 에러는 captureError(error)로 모니터링 전송
4. 회복 불가 에러는 error.tsx에서 reset 버튼 제공
```

모든 API 에러는 반드시 `ApiError` 형태로 throw한다.
`new Error('...')` 직접 throw는 `mapApiError`에서 처리 불가.
