# 04 — 컴포넌트 컨벤션 · Tailwind · shadcn 셋업

## ⚠️ context7 보정

이 파일을 적용하기 전, SKILL.md Step 0 결과로 확인:
- Tailwind 현재 메이저 (v4 vs v3): 설정 방식이 완전히 다름
  - v4: `tailwind.config` 없음, `postcss.config.mjs`에 `@tailwindcss/postcss`, `globals.css`에 `@import "tailwindcss"` + `@theme inline`
  - v3: `tailwind.config.ts` 사용, `globals.css`에 `@tailwind base/components/utilities`
- shadcn 최신 init 명령어 및 `components.json` 스키마

---

## 1. Tailwind CSS 설정 (v4 방식 — v3이면 아래 주석 참고)

**`postcss.config.mjs`** (v4):
```js
const config = {
  plugins: {
    '@tailwindcss/postcss': {},
  },
}
export default config
```

**`app/globals.css`** (v4 CSS-first 방식):
```css
@import "tailwindcss";
@import "tw-animate-css";        /* 필요 시 */

/* shadcn/ui CSS variables 임포트 (shadcn init 후 생성됨) */
@import "shadcn/tailwind.css";   /* shadcn 패키지 설치 시 */

/* 다크모드 variant */
@custom-variant dark (&:is(.dark *));

/* 디자인 토큰 — 프로젝트에 맞게 수정 */
@theme inline {
  --font-sans: var(--font-geist-sans, ui-sans-serif, system-ui);
  --font-mono: var(--font-geist-mono, ui-monospace);
  /* 브랜드 색상 추가 예시 */
  /* --color-brand: oklch(62% 0.17 165); */
}

@layer base {
  *, *::before, *::after {
    border-color: var(--color-border);
  }
  body {
    background-color: var(--color-background);
    color: var(--color-foreground);
  }
  button { cursor: pointer; }
}
```

> v3 사용 시: `tailwind.config.ts`에서 `content` 경로 설정 + `globals.css`는 `@tailwind base/components/utilities`.

---

## 2. `lib/utils.ts` — cn() 헬퍼

shadcn 표준. 조건부 클래스 병합 + Tailwind 충돌 해결.

```ts
import { clsx, type ClassValue } from 'clsx'
import { twMerge } from 'tailwind-merge'

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}
```

사용:
```ts
cn('flex items-center', isActive && 'bg-brand', className)
```

---

## 3. shadcn/ui 초기화

context7에서 최신 CLI 명령을 확인하고, 아래를 기준으로 적용한다:

```bash
# 초기화 (style: base-nova 또는 default, baseColor: neutral)
pnpm dlx shadcn@latest init

# 기본 컴포넌트 추가
pnpm dlx shadcn@latest add button dialog tooltip sonner
```

`components.json` 생성 확인:
```json
{
  "style": "default",
  "rsc": true,
  "tsx": true,
  "tailwind": {
    "css": "app/globals.css",
    "baseColor": "neutral",
    "cssVariables": true
  },
  "aliases": {
    "components": "@/components",
    "utils": "@/lib/utils",
    "ui": "@/components/ui",
    "lib": "@/lib",
    "hooks": "@/hooks"
  }
}
```

---

## 4. 컴포넌트 분류 원칙

```
components/
  ui/       — shadcn primitive (lowercase 파일명: button.tsx, dialog.tsx)
              직접 수정 가능한 소스 파일. 추상화 없이 사용.
  common/   — 도메인 무관 재사용 (ErrorModal, Pagination, ConfirmModal 등)
              PascalCase 파일명
  layout/   — 레이아웃 셸 (Header, NavBar, PageShell 등)
              PascalCase 파일명
  [domain]/ — 도메인별 (PostCard, UserAvatar 등)
              PascalCase 파일명, 도메인 하위 컴포넌트는 서브디렉토리 허용
```

**파일명 규칙**:
- `components/ui/` → `button.tsx` (lowercase, shadcn 컨벤션 유지)
- 나머지 → `PostCard.tsx`, `Header.tsx` (PascalCase)

---

## 5. CVA (class-variance-authority) 컴포넌트 패턴

여러 variant가 있는 UI 컴포넌트는 CVA로 관리한다.
`buttonVariants`처럼 variant 정의를 별도 export하면 다른 컴포넌트에서 재사용 가능.

```tsx
// components/ui/button.tsx (shadcn install 후 커스터마이징 예시)
import { cva, type VariantProps } from 'class-variance-authority'
import { cn } from '@/lib/utils'

const buttonVariants = cva(
  'inline-flex items-center justify-center rounded-md text-sm font-medium transition-colors focus-visible:outline-none disabled:pointer-events-none disabled:opacity-50',
  {
    variants: {
      variant: {
        default: 'bg-neutral-900 text-white hover:bg-neutral-800',
        outline: 'border border-neutral-200 bg-white hover:bg-neutral-50',
        ghost: 'hover:bg-neutral-100',
        destructive: 'bg-red-500 text-white hover:bg-red-600',
      },
      size: {
        default: 'h-10 px-4 py-2',
        sm: 'h-8 px-3 text-xs',
        lg: 'h-12 px-6',
        icon: 'h-10 w-10',
      },
    },
    defaultVariants: {
      variant: 'default',
      size: 'default',
    },
  }
)

interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {}

function Button({ className, variant, size, ...props }: ButtonProps) {
  return (
    <button
      className={cn(buttonVariants({ variant, size, className }))}
      {...props}
    />
  )
}

export { Button, buttonVariants }
```

---

## 6. 'use client' 선언 전략

- **기본**: 모든 컴포넌트는 서버 컴포넌트 (RSC)
- **`'use client'` 필요한 경우**:
  - `useState`, `useEffect`, `useRef`, `useCallback` 등 React 훅 사용
  - 이벤트 핸들러 직접 바인딩
  - 브라우저 API (`window`, `document`, `localStorage`)
  - `useQuery`, `useMutation` (react-query 클라이언트 훅)
- 규칙: `'use client'` 경계를 **가능한 한 말단 leaf 컴포넌트**로 내린다

```tsx
// Good — 부모는 RSC, 인터랙션 있는 자식만 'use client'
// app/posts/page.tsx (RSC)
import { PostList } from '@/components/posts/PostList'  // 'use client' 컴포넌트
export default function PostsPage() {
  return <PostList />
}

// components/posts/PostList.tsx
'use client'
import { usePosts } from '@/hooks/usePosts'
// ...
```

---

## 7. props 타입 패턴

```tsx
// interface 우선 (단순 객체)
interface CardProps {
  title: string
  description?: string
  onClick?: () => void
  children?: React.ReactNode
  className?: string  // 스타일 주입 허용 시 포함
}

// 조합 타입은 type 사용
type ButtonVariant = 'primary' | 'secondary' | 'ghost'
type ButtonSize = 'sm' | 'md' | 'lg'
```

---

## 8. 컴포넌트 작성 체크리스트

새 컴포넌트를 만들 때 확인:

- [ ] 서버/클라이언트 컴포넌트 여부 결정 → `'use client'` 필요하면 선언
- [ ] props 타입 명시 (`interface XxxProps`)
- [ ] `className` 주입 허용 여부 결정 (`cn()` 사용)
- [ ] variant 있으면 CVA 사용
- [ ] `export default` 금지 (named export만, page/layout 제외)
- [ ] 파일명: PascalCase (shadcn `ui/`는 lowercase)
