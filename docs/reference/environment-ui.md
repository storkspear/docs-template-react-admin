# Environment UI — 기술 스택 다이어그램

이 문서는 template-react-admin 의 기술 스택을 **한 장의 다이어그램**으로 요약해요. 버전의 정본은 [`environment.md`](./environment.md) 와 `package.json` 이에요.

%%TECH_STACK_DIAGRAM%%

## 읽는 법

- **코어 → UI → 데이터** 층이 프론트엔드 전부예요 — 상태관리는 TanStack Query(서버) + useState(로컬)로 단순하게 유지해요.
- MSW mock 덕분에 백엔드 없이 전 화면이 동작해요. `.env` 의 `VITE_USE_MOCK=false` 로 실서버 전환.
- 맨 아래 화살표는 짝 백엔드 [`template-spring`](https://github.com/storkspear/template-spring) 의 `/api/admin/*` 계약이에요 — RBAC 4티어(viewer/support/admin/master) 권한 체계를 포함해요.

> 💡 이 다이어그램은 docs 뷰어에서만 렌더링돼요. GitHub raw 에서는 `%%TECH_STACK_DIAGRAM%%` 플레이스홀더만 보여요.

## 관련 문서

- [`Environment`](./environment.md) — 버전표 인벤토리 (정본)
- [`admin-api.md`](../api-contract/admin-api.md) — 백엔드 계약 전체
- [`screens.md`](../guide/screens.md) — 화면별 가이드
