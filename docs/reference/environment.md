# Environment — 프레임워크 · 라이브러리 인벤토리

이 문서는 template-react-admin 이 쓰는 기술 스택을 버전과 함께 정리해요. 버전의 정본은 `package.json` 이고, 여기 값은 2026-07-29 기준이에요. 시각 요약판은 [`Environment UI`](./environment-ui.md) 에 있어요.

## 코어

| 이름 | 버전 | 용도 |
|---|---|---|
| React | ^19.2 | UI 라이브러리 (+ antd v5 React 19 패치) |
| TypeScript | ~6.0 | 언어 — **strict 모드** (`tsconfig.app.json`) |
| Vite | ^8.1 | 빌드 · dev 서버 (`npm run dev` → :5173) |

## UI

| 이름 | 버전 | 용도 |
|---|---|---|
| Ant Design (antd) | ^5.25 | 컴포넌트 — `ConfigProvider` 토큰은 `src/lib/theme.ts`. **5.25 는 `destroyOnHidden` 때문에 내릴 수 없는 하한** (아래 각주) |
| ag-grid-community / ag-grid-react | ^36.0 | 서버 페이지네이션 그리드 (Theming API) |
| Tailwind CSS | ^3.4 | 유틸리티 CSS (+ autoprefixer · postcss) |
| lucide-react | ^1.25 | 아이콘 — `src/lib/icons.tsx` 가 antd 아이콘 이름으로 재-export |
| @xyflow/react | ^12.11 | ERD 콘솔(시스템 설계도) 그래프 — `SchemaPageLazy` 로 지연 로드 |
| dayjs | ^1.11 | 날짜 포맷 |
| 자체 차트 | — | `MiniBarChart` · `MiniPieChart` · `Sparkline` · `StackedBarChart` (외부 차트 라이브러리 없음) |

### antd 5.25 하한

`Modal` · `Drawer` 의 `destroyOnHidden` 은 **antd 5.25 에서 들어온 API** 예요 (그 이전 이름 `destroyOnClose` 는 현재 `@deprecated`). 이 레포는 `MediaModal` · `FileDetailDrawer` · `UserDetailDrawer` · `useReasonModal` · `ContentComposePage` · `RolesPage` 에서 `destroyOnHidden` 을 쓰고 있어서, 5.25 아래로 내리면 **타입 에러 없이 조용히 무시**돼요 — 모달을 닫아도 영상이 계속 재생되고, 사유 입력창에 이전 텍스트가 남아요. **하한을 낮추지 마세요.**

## 데이터

| 이름 | 버전 | 용도 |
|---|---|---|
| TanStack Query | ^5.62 | 서버 상태 — 목록은 `useAdminList` 훅 |
| react-router-dom | ^7.18 | 라우팅 — `src/nav.tsx` 의 `NAV_ITEMS` 가 단일 소스 |
| MSW | ^2.7 | mock — 전 화면이 백엔드 없이 동작, `VITE_USE_MOCK=false` 로 실서버 전환 |

## 개발 도구

| 이름 | 버전 | 용도 |
|---|---|---|
| oxlint | ^1.71 | 린트 |
| tsc -b | — | 타입 검사 — `npm run build` 게이트 |

## 백엔드 계약

짝 백엔드는 [`template-spring`](https://github.com/storkspear/template-spring) 의 `/api/admin/*` 이에요. 엔드포인트·에러 코드·RBAC 계약은 [`api-contract/admin-api.md`](../api-contract/admin-api.md) 가 정본이에요.

## 관련 문서

- [`Environment UI`](./environment-ui.md) — 이 인벤토리의 다이어그램 요약
- [루트 `README.md`](../../README.md) — 빠른 시작 · 화면↔엔드포인트 매핑
- [루트 `CLAUDE.md`](../../CLAUDE.md) — 단일 소스 표 · 컨벤션 5분 요약
