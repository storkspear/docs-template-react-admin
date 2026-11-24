# 파생 레포 체크리스트 (심화판)

"Use this template"으로 저장소를 만든 뒤 설치, 브랜딩, 메뉴 구성, 서버 연결 순서로 설정해요. 이미 운영 중인 앱에 화면을 추가한다면 4절부터 읽어도 돼요.

---

<a id="0-첫-셋업--factory-cli"></a>

## 0. 첫 셋업 — factory CLI

```bash
git clone git@github.com:<org>/<your-admin>.git && cd <your-admin>
./factory install              # ~/.local/bin/<repo명> symlink 등록 (이후 <repo명> 으로 어디서든 호출)
<repo명> local init             # .env 백업·복원 또는 예제 복사 + npm install
<repo명> local start            # 백엔드(GET /api/admin/health) 프로브 → 있으면 실서버, 없으면 mock 자동 기동
```

`local init`은 기존 `.env`가 있으면 보존하고 저장소 밖 백업을 갱신해요. 파일이 없으면 백업에서 복원하거나 `.env.example`을 복사한 뒤 의존성을 설치해요. `gh`와 `origin`이 있고 `.env`에 `GHCR_TOKEN`이 설정돼 있으면 해당 저장소의 GitHub Secret도 등록해요. 로컬 실행에 이 토큰이 필요한 것은 아니에요.

`local`은 이 레포 기준 기본 env라 `<repo명> start`, `<repo명> test`로 줄여 써도 동일해요. `local start`가 백엔드를 못 찾으면 "백엔드 미연결 → mock 데이터로 실행합니다" 안내 후 mock으로 폴백하니, **처음엔 백엔드 없이도 전 화면을 바로 시연**할 수 있어요.

`dev start`는 `.env`의 `VITE_PROXY_TARGET_DEV`를 프로브 대상으로 써요(미설정 시 에러 안내). `prod start`는 로컬 dev 서버로 안 붙고 정적 배포 안내만 출력해요 — prod는 `npm run build` 결과물을 호스팅에 올리는 방식이에요.

<a id="devprod-배포--repo-dev-deploy--repo-prod-deploy-mac-mini-nginx--cloudflare-tunnel"></a>

### dev/prod 배포 — `<repo> dev deploy` / `<repo> prod deploy` (Mac mini nginx + Cloudflare Tunnel)

홈서버 배포와 정리 절차는 [배포 가이드](./deployment.md)에 모았어요. `dev`는 `.env`, `prod`는 `.env.prod`를 읽어요. SSH·nginx 권한과 Cloudflare 설정을 준비한 뒤 해당 환경의 명령을 실행해주세요.

---

<a id="1-브랜딩--appconfigts--moodsts--palettets--favicon"></a>

## 1. 브랜딩 — appConfig.ts / moods.ts / palette.ts / favicon

앱 이름과 소개는 `appConfig.ts`, UI 테마는 `moods.ts`, 테마와 무관한 보조색은 `palette.ts`에서 관리해요. 컴포넌트에 값을 따로 적지 않고 이 설정을 참조해요. 테마는 Obsidian·Blurple·Ember·Glacier 중 선택하거나 추가할 수 있어요. 정적 파일인 `index.html`과 favicon은 별도로 수정해야 해요.

### `src/appConfig.ts`

```ts
export const appConfig = {
  name: 'Admin',                 // 사이드바 워드마크 + 저작권 표기
  tagline: '운영 콘솔',           // 로그인 모바일 헤더 등 한 줄 소개
  documentTitle: 'Admin · 운영 콘솔',  // 브라우저 탭 제목
  login: {
    title: '앱 공장 운영 콘솔',
    description: '여러 앱의 사용자·매출·활동을 한 곳에서. …',
    subtitle: '운영 콘솔에 접속하려면 로그인하세요.',
  },
  supportEmail: 'support@example.com',
} as const
```

`main.tsx`가 부팅 시 `document.title = appConfig.documentTitle`로 설정해요. 다만 **`index.html`의 `<title>` 태그는 별도 정적 값**이라(JS가 로드되기 전 잠깐 보이는 제목), 브랜드를 바꿀 때는 `index.html`의 `<title>`도 같이 맞춰주는 게 좋아요.

### `src/lib/palette.ts`

```ts
export const brand = {
  50: '#eef2ff', 100: '#e0e7ff', /* … */ 600: '#4f46e5', /* … */ 800: '#3730a3',
} as const
export const PRIMARY = brand[600]   // 무드와 독립적인 보조색
```

`brand` 스케일은 이제 **차트 기본색·종이(본문) 액센트 같은 무드 무관 보조색**에만 쓰여요 — antd `ConfigProvider` 테마(`lib/theme.ts`)·ag-grid 테마(`lib/agGrid.ts`)·로그인 그라디언트·`ComponentsPage` 스와치는 `moods.ts`의 무드에서 파생돼요(2026-07 디자인 리프레시). 컴포넌트에서는 색상 값을 직접 적는 대신 `brand[N]`, `semantic.*`, `alpha(hex, a)`를 사용해요.

<a id="publicfaviconsvg-예외--정적-파일이라-별도-수정"></a>

### `public/favicon.svg` (예외 — 정적 파일이라 별도 수정)

```svg
<stop stop-color="#4f46e5" />   <!-- brand[600] 과 같은 값으로 직접 교체 -->
<stop offset="1" stop-color="#4338ca" />  <!-- brand[700] -->
```

정적 SVG라 JS 변수를 못 써요. 브랜드 색(무드 primary 또는 `brand` 스케일)을 바꿨으면 **이 파일의 hex 두 개도 같이 손으로 맞춰야** 브랜드가 완전히 일치해요(현재 값은 `brand[600]`/`brand[700]`과 동일).

**체크**: `npm run dev` 후 로그인 화면 그라디언트·사이드바·`/components` 페이지의 색 스와치가 새 브랜드로 보이는지 확인.

---

<a id="2-메뉴-구성--navtsx"></a>

## 2. 메뉴 구성 — nav.tsx

`src/nav.tsx`의 `NAV_ITEMS`가 라우트·사이드바 메뉴·활성 표시(선택된 메뉴 강조)의 **단일 소스**예요. `App.tsx`와 `AppLayout`이 이 배열을 읽어 라우트와 메뉴를 구성해요. 접근을 제한할 화면은 `src/lib/rbac.ts`의 `NAV_PERM`에도 권한을 연결하고, 버튼과 서버 API의 권한 검사도 확인해주세요. 메뉴 등록만으로 서버 권한이 추가되지는 않아요.

```tsx
export const NAV_ITEMS: NavItem[] = [
  // group = 사이드바 그룹 헤더('공통'/'앱별'/'템플릿'), scope = 노출 조건('global'=항상, 'app'=앱 선택 시만)
  { path: '/', label: '대시보드', icon: <DashboardOutlined />, element: <DashboardPage />, scope: 'global', group: '공통' },
  { path: '/apps', label: '서비스 현황', icon: <AppstoreOutlined />, element: <AppsPage />, scope: 'global', group: '공통' },
  { path: '/apps/:slug', element: <AppDetailPage /> },  // label 없음 = 메뉴엔 안 뜨고 라우트만
  { path: '/users', label: '사용자', icon: <TeamOutlined />, element: <UsersPage />, scope: 'app', group: '앱별' },
  // … 이 앱에 필요 없는 화면은 통째로 줄 삭제
]
```

**이 앱에 필요 없는 화면 제거**: 예를 들어 발송(`/send`)·역할·권한(`/roles`)이 필요 없다면 `NAV_ITEMS`에서 해당 줄을 지우고 `src/pages/SendPage.tsx`/`RolesPage.tsx` 파일도 함께 삭제하세요(빌드에 안 잡히는 미사용 파일을 남겨두면 나중에 헷갈려요).

**상세/파라미터 라우트 추가**: `AppDetailPage`처럼 사이드바에 안 띄우고 라우트만 있는 화면은 `label`을 생략하면 돼요. `App.tsx`가 `NAV_ITEMS`를 순회하며 `label` 유무와 무관하게 전부 `<Route>`로 등록하고, `AppLayout`의 `MENU_ITEMS`(=`NAV_ITEMS.filter(n => n.label)`)만 사이드바에 표시해요.

---

<a id="3-mock--실서버-전환"></a>

## 3. Mock → 실서버 전환

개발 실행은 `factory`의 자동 선택과 Vite 직접 실행 중 하나를 사용해요. 발송·영수증 등 [Mock 전용 기능](../api-contract/admin-api.md#mock--실서버-차이)은 실행 모드만 바꾼다고 실서버에서 제공되지는 않아요.

**factory로 실행할 때**는 `.env`에 프로브 대상을 설정해요.

```dotenv
VITE_PROXY_TARGET=http://localhost:8081
VITE_PROXY_TARGET_DEV=https://api-dev.example.com
```

`./factory local start`는 로컬 주소의 `GET /api/admin/health`를 확인해 실서버·Mock 모드를 결정해요. `./factory dev start`는 `VITE_PROXY_TARGET_DEV`를 확인하고, `--mock`을 붙이면 Mock으로 실행해요. 이 경로에서는 `.env`의 `VITE_USE_MOCK` 값을 그대로 따르지 않아요. 헤더의 MOCK 배지로 실행 모드를 확인할 수 있어요.

**Vite를 직접 실행할 때**는 `.env.local` 등에 값을 설정하고 `npm run dev`를 실행해요.

```dotenv
VITE_PROXY_TARGET=http://localhost:8081
VITE_USE_MOCK=false
# 프록시 없이 절대 URL을 호출할 때만 설정해요. 서버의 CORS 설정도 필요해요.
# VITE_API_BASE=https://api.example.com
```

배포된 정적 파일의 설정은 [배포 가이드](./deployment.md)를 따라 바꿔요. Vite 설정은 빌드할 때 반영되므로 다시 빌드하고 배포해야 해요.

전환 시 확인할 것:

- [ ] `src/lib/types.ts`의 DTO가 실제 백엔드 응답 필드와 1:1인지 (`docs/api-contract/admin-api.md` 대조)
- [ ] `src/mocks/fixtures.ts`의 슬러그(`tradelog`/`gymlog`/`moodlog`)는 데모용 — 실서버 전환 후엔 무시되니 그대로 둬도 무방하지만, 로컬 mock 데모용 문구는 이 앱 도메인에 맞게 바꿔도 좋아요
- [ ] 로그인 자격증명이 실서버 admin 계정으로 바뀌었는지 (mock 전용 데모 계정 `master@example.com`/`password` — 이메일 프리픽스 `viewer@`/`support@`/`admin@`/`master@`로 역할이 결정되는 규칙 포함 — 안내 문구는 `USE_MOCK`일 때만 노출되니 신경 안 써도 됨 — `LoginPage.tsx` 참고)
- [ ] `npm run build` · `npm test` 통과

---

## 4. 화면 추가 패턴

<a id="그리드형목록-화면--가장-흔한-패턴"></a>

### 그리드형(목록) 화면 — 가장 흔한 패턴

`GridPage`, `AdminDataGrid`, `useAdminList` 조합으로 PC 페이지네이션과 모바일 무한스크롤을 구성해요. 아래는 구조를 보여주는 예제예요. `Xxx`, `getXxx`, `xxxColumns`와 필터 입력은 해당 화면에 맞게 정의해야 해요. 앱을 선택하기 전에는 조회를 비활성화해요.

```tsx
// src/pages/XxxPage.tsx
import { useState } from 'react'
import { Empty, Form } from 'antd'
import { useAdminList } from '../lib/useAdminList'
import { useAppContext } from '../lib/appContext'
import GridPage from '../components/GridPage'
import AdminDataGrid from '../components/AdminDataGrid'
import QueryErrorState from '../components/QueryErrorState'
import { getXxx } from '../api/client'   // client.ts 에 엔드포인트 함수 추가 필요
import type { Xxx } from '../lib/types'  // types.ts 에 DTO 추가 필요

export default function XxxPage() {
  const [form] = Form.useForm()
  const { slug } = useAppContext()
  const [filters, setFilters] = useState<{ query?: string }>({})

  const list = useAdminList<Xxx>({
    key: ['xxx', slug, filters],
    fetchPage: (page, size) => getXxx(slug as string, { ...filters, page, size }),
    enabled: !!slug,
  })

  if (!slug) return <Empty description="상단에서 앱을 선택하세요" />

  return (
    <GridPage
      title="Xxx"
      filters={<Form form={form} layout="inline" onFinish={(v) => { setFilters(v); list.resetPage() }}>{/* Form.Item… */}</Form>}
      onSearch={() => form.submit()}
      onReset={() => { form.resetFields(); setFilters({}); list.resetPage() }}
    >
      {list.isError ? (
        <QueryErrorState error={list.error} title="목록 오류" onRetry={list.refetch} />
      ) : (
        <AdminDataGrid<Xxx>
          columns={xxxColumns}          // 이 화면에 맞는 ColDef<Xxx>[]를 정의해요
          rowData={list.rows}
          loading={list.loading}
          getRowId={(p) => String(p.data.id)}
          {...list.gridProps}
        />
      )}
    </GridPage>
  )
}
```

- 컬럼 정의(`ColDef[]`)는 `userColumns.tsx`처럼 `src/lib/xxxColumns.tsx`로 분리하면 여러 화면(목록 최상위 + 상세 탭)에서 재사용하기 좋아요.
- 앱 슬러그를 고르는 셀렉트가 필요하면 **항상 `useAppOptions()`**를 쓰세요 — 앱 목록이 하드코딩돼 있으면 새 앱이 추가될 때마다 여러 화면을 고쳐야 해요.

<a id="폼형-화면--card--form"></a>

### 폼형 화면 — `Card` + `Form`

목록이 아니라 설정/발송 같은 단일 폼 화면은 `SettingsPage.tsx`/`SendPage.tsx`를 레퍼런스로 삼으세요.

```tsx
export default function XxxPage() {
  const { message } = App.useApp()   // 정적 message import 지양 — main.tsx 의 <AntdApp> 컨텍스트 사용
  const [form] = Form.useForm()

  return (
    <div style={{ flex: 1, minHeight: 0, overflowY: 'auto' }}>
      <Typography.Title level={3} style={{ marginTop: 0 }}>Xxx</Typography.Title>
      <Card size="small" style={{ maxWidth: 720 }}>
        <Form layout="vertical" form={form} onFinish={(v) => { /* … */ message.success('저장했어요.') }}>
          {/* Form.Item … */}
          <Button type="primary" htmlType="submit">저장</Button>
        </Form>
      </Card>
    </div>
  )
}
```

### 화면 추가 후 마무리

1. `src/nav.tsx`의 `NAV_ITEMS`에 라우트를 추가하고 필요한 `NAV_PERM` 권한을 연결해요: `{ path: '/xxx', label: '…', icon: <…/>, element: <XxxPage /> }`
2. 새 엔드포인트가 필요하면 `src/lib/types.ts`(DTO)·`src/api/client.ts`(함수)·`src/mocks/handlers.ts`+`fixtures.ts`(mock) 세 곳 동기화 — 실서버 계약은 `docs/api-contract/admin-api.md`와 대조
3. `npm run build` · `npm test` 통과 확인

---

## 5. 최종 체크리스트

- [ ] `appConfig.ts` — 앱 이름·로그인 문구 교체, `index.html`의 `<title>`도 맞춤
- [ ] `moods.ts` — 브랜드 무드 확정(4종 중 선택 or 커스텀 추가) + `public/favicon.svg`의 hex 2곳 수동 동기화 (차트·종이 보조색은 `palette.ts`)
- [ ] `nav.tsx` — 이 앱에 필요한 메뉴만 남기고, 안 쓰는 페이지 파일도 삭제
- [ ] `src/mocks/` — 데모 데이터를 이 앱 도메인으로 교체 (또는 실서버 전환으로 무시)
- [ ] `src/api/client.ts` / `src/lib/types.ts` — 백엔드 계약과 1:1 확인 (`docs/api-contract/admin-api.md`)
- [ ] `.env` — `VITE_PROXY_TARGET`(/`_DEV`) 설정, `<repo명> local start`로 자동 mock↔실서버 전환 확인
- [ ] `npm run build` · `npm test` 통과

---

## 관련 문서

- [`../api-contract/README.md`](../api-contract/README.md) — 백엔드 계약 구성
- [`../api-contract/admin-api.md`](../api-contract/admin-api.md) — 엔드포인트 상세
- [배포 가이드](./deployment.md) — 홈서버 배포·검증·정리
- [`./screens.md`](./screens.md) — 기존 화면들이 각각 무엇을 하는지 (화면 가이드)
- [`../../CLAUDE.md`](../../CLAUDE.md) — 단일 소스 표 · 명령어 참조
- [`../../README.md`](../../README.md) — 빠른 시작 · 폴더 구조
