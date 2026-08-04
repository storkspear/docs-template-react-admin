# Environment — 프레임워크 · 라이브러리 인벤토리

이 문서는 template-react-admin 이 쓰는 기술 스택을 버전과 함께 정리해요. 버전의 정본은 `package.json` 이고, 여기 값은 2026-08-01 기준이에요. 시각 요약판은 [`Environment UI`](./environment-ui.md) 에 있어요.

## 코어

| 이름 | 버전 | 용도 |
|---|---|---|
| React | ^19.2 | UI 라이브러리 — antd 6 가 React 19 를 네이티브 지원해요. `@ant-design/v5-patch-for-react-19` 는 이 레포에 없어요 |
| TypeScript | ~7.0 | 언어 — **strict 모드** (`tsconfig.app.json`). 7.x 는 네이티브 포트라 `tsc -b` 가 1초 안에 돌아요 |
| Vite | ^8.1 | 빌드 · dev 서버 (`npm run dev` → :5173) |

### `skipLibCheck: true` 를 끄지 마세요

`tsconfig.app.json` · `tsconfig.node.json` 양쪽에 켜져 있어요. 끄면 antd 6 의 하위 의존
(`@rc-component/select` · `picker` · `image`)에서 `.d.ts` 오류 4건이 나와요 — 우리 코드가 아니라
업스트림 타입 정의 문제라 우리 쪽에서 고칠 수 없어요.

## UI

| 이름 | 버전 | 용도 |
|---|---|---|
| Ant Design (antd) | ^6.5 | 컴포넌트 — `ConfigProvider` 토큰은 `src/lib/theme.ts` (아래 각주) |
| ag-grid-community / ag-grid-react | ^36.0 | 서버 페이지네이션 그리드 (Theming API) |
| lucide-react | ^1.25 | 아이콘 — `src/lib/icons.tsx` 가 antd 아이콘 이름으로 재-export |
| @xyflow/react | ^12.11 | ERD 콘솔(시스템 설계도) 그래프 — `SchemaPageLazy` 로 지연 로드 |
| dayjs | ^1.11 | 날짜 포맷 |
| 자체 차트 | — | `MiniBarChart` · `MiniPieChart` · `Sparkline` · `StackedBarChart` (외부 차트 라이브러리 없음) |

### antd 5 코드를 가져올 때 — antd 6 이관 기준표

파생 레포에서 antd 5 시절 코드를 가져올 때 이 표를 기준으로 보면 돼요. antd 5 → 6 에서
실제로 바뀐 지점을 **antd 6.5.3 의 타입 정의로 직접 확인한 결과**예요.

| antd 5 | antd 6 | 상태 |
|---|---|---|
| `Modal` `styles.content` | `styles.container` | **필수** — semantic 키에 `content` 가 없어요 |
| `Divider` `orientation="left\|right\|center"` | `titlePlacement` | **의미 변경 주의** — `orientation` 은 남아 있지만 이제 축(`horizontal\|vertical`)을 뜻해요. 값도 `start`/`end` 가 정본이고 `left`/`right` 는 RTL 보정을 거치는 별칭이에요 |
| `Divider` `type` | `orientation` | deprecated |
| `Space` `direction` | `orientation` | deprecated |
| `Tag` `bordered={false}` | `variant="filled"` | deprecated (아직 동작) |
| `Statistic` `valueStyle` | `styles.content` | deprecated (아직 동작) |
| `antd/es/table` 의 `ColumnsType` | `antd` 의 `TableColumnsType` | 내부 경로 import 회피 |
| `Alert` `message` | `title` | deprecated |
| `Modal` `maskClosable` | `mask={{ closable: true }}` | deprecated |
| `Divider` `orientationMargin` | `styles.content.margin` | deprecated |
| `Drawer` `width` · `height` | `size` | deprecated (아직 동작). 숫자·CSS 길이·`'100%'` 모두 그대로 받아요 |
| `Drawer` `styles.content` | `styles.section` | **필수** — `content` 는 deprecated 이고 DOM 에 적용되지 않아요 |

> deprecated prop 은 타입 검사로 안 걸리고 **렌더 시점 `console.error`** 로만 드러나요.
> `npm test`(화면 스모크)가 렌더 중 콘솔을 가로채 `deprecated` 문자열을 실패로 처리해요
> (`src/test/screens.smoke.test.tsx:93-102`). 다만 스모크가 **렌더한 화면만** 볼 수 있으니,
> 오류 상태 전용 컴포넌트처럼 mock 성공 경로에서 안 그려지는 곳은 여전히 수동 확인이 필요해요.

**바꾸지 않아도 되는 것** — antd 6 에서도 그대로 유효한 API 예요.

| API | 확인 결과 |
|---|---|
| `Descriptions.Item` children | deprecated 아님. `items` prop 과 **둘 다 유효**해요 |

- **`@ant-design/v5-patch-for-react-19` 는 반드시 제거**해야 해요. antd 6 는 React 19 를 네이티브 지원하고, 패치가 남아 있으면 충돌해요.
- `destroyOnHidden`(5.25 도입, `destroyOnClose` 의 후계)은 6 에서도 그대로예요 — `MediaModal` · `FileDetailDrawer` · `UserDetailDrawer` · `useReasonModal` · `ContentComposePage` · `RolesPage` 가 쓰고 있어요.
- `ConfigProvider` 를 10곳에서 쓰고 최대 3단까지 중첩돼요. 하위 `ConfigProvider` 에 테마를 안 넘기면 팝업 배경이 상위 무드를 못 따라가요.

### CSS 리셋

이 레포에는 Tailwind 가 없어요. preflight 리셋은 `src/index.css` 상단의
`@import 'antd/dist/reset.css'` 가 담당해요. `postcss` · `autoprefixer` 도 직접 의존이 아니에요
(`postcss` 는 vite 8 의 직접 의존이라 트리에는 남아요).

## 데이터

| 이름 | 버전 | 용도 |
|---|---|---|
| TanStack Query | ^5.62 | 서버 상태 — 목록은 `useAdminList` 훅 |
| react-router | ^8.3 | 라우팅 — `src/nav.tsx` 의 `NAV_ITEMS` 가 단일 소스. import 는 `react-router` 패키지에서 해요 (`react-router-dom` 은 이 레포에 없어요) |
| MSW | ^2.7 | mock — 전 화면이 백엔드 없이 동작, `VITE_USE_MOCK=false` 로 실서버 전환 |

## 개발 도구

| 이름 | 버전 | 용도 |
|---|---|---|
| oxlint | ^1.71 | 린트 |
| tsc -b | — | 타입 검사 — `npm run build` 게이트 |
| Vitest | ^4.1 | 화면 스모크 — `npm test`, `.githooks/pre-push` 게이트 |
| @testing-library/react · jsdom | ^16.3 · ^30.0 | 스모크 렌더 — `src/mocks/handlers.ts` 를 `msw/node` 로 재사용 |

## 백엔드 계약

짝 백엔드는 [`template-spring`](https://github.com/storkspear/template-spring) 의 `/api/admin/*` 이에요. 엔드포인트·에러 코드·RBAC 계약은 [`api-contract/admin-api.md`](../api-contract/admin-api.md) 가 정본이에요.

## 관련 문서

- [`Environment UI`](./environment-ui.md) — 이 인벤토리의 다이어그램 요약
- [루트 `README.md`](../../README.md) — 빠른 시작 · 화면↔엔드포인트 매핑
- [루트 `CLAUDE.md`](../../CLAUDE.md) — 단일 소스 표 · 컨벤션 5분 요약
