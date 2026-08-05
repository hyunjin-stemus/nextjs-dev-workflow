# nextjs-dev-workflow

Next.js 프론트엔드 개발을 위한 Claude Code 플러그인입니다.  
REQUIREMENT.md 기반 Task Master AI 워크플로우를 슬래시 커맨드로 제공합니다.

## 스킬

| 이름 | 설명 |
|---|---|
| `nextjs-react-scaffold` | 빈 폴더에서 Next.js + React 신규 프로젝트를 처음부터 스캐폴딩 (레이어 분리, TanStack Query, 공통 에러 처리, Zustand, shadcn/ui 컨벤션 등 프로덕션 레디 보일러플레이트) |

## 커맨드

| 커맨드 | 설명 |
|---|---|
| `/nextjs-dev-workflow:cycle:prd-from-requirement` | REQUIREMENT.md → PRD 작성/검수/승인 → task 분할 |
| `/nextjs-dev-workflow:cycle:task-start [id]` | task 선택 → plan mode → 사용자 컨펌 |
| `/nextjs-dev-workflow:cycle:task-implement [id]` | 브랜치 → TDD → 구현 → Playwright → 타입체크 → 린트 → 코드리뷰 |
| `/nextjs-dev-workflow:cycle:task-ship [id]` | 빌드검증 → 커밋 → PR → 리뷰 → 병합 → done |
| `/nextjs-dev-workflow:cycle:cycle-finish` | 전체 task done 확인 → 회고 문서화 |

## 설치

```bash
/plugin install github:hyunjin-stemus/nextjs-dev-workflow
```
