# 파생 레포 체크리스트 (심화판)

`CLAUDE.md` §8의 체크리스트를 단계별로 풀어썼어요. "Use this template"으로 새 레포를 만든 직후부터, 실서버 연결까지 순서대로 따라가면 돼요.

---

## 0. 첫 셋업 — factory CLI

```bash
git clone git@github.com:<org>/<your-admin>.git && cd <your-admin>
./factory install              # ~/.local/bin/<repo명> symlink 등록 (이후 <repo명> 으로 어디서든 호출)
<repo명> local init             # .env 생성(.env.example 복사, 있으면 보존) + npm install
<repo명> local start            # 백엔드(GET /api/admin/health) 프로브 → 있으면 실서버, 없으면 mock 자동 기동
```

`local`은 이 레포 기준 기본 env라 `<repo명> start`, `<repo명> test`로 줄여 써도 동일해요. `local start`가 백엔드를 못 찾으면 "백엔드 미연결 → mock 데이터로 실행합니다" 안내 후 mock으로 폴백하니, **처음엔 백엔드 없이도 전 화면을 바로 시연**할 수 있어요.

`dev start`는 `.env`의 `VITE_PROXY_TARGET_DEV`를 프로브 대상으로 써요(미설정 시 에러 안내). `prod start`는 로컬 dev 서버로 안 붙고 정적 배포 안내만 출력해요 — prod는 `npm run build` 결과물을 호스팅에 올리는 방식이에요.

---

## 1. 브랜딩 — appConfig.ts / palette.ts / favicon

이 셋을 바꾸면 로고 워드마크·로그인 화면·그리드·차트·브라우저 탭까지 전부 따라와요. 코드 어디에도 이 값들을 하드코딩하면 안 돼요 — 항상 이 세 파일 참조.

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
export const PRIMARY = brand[600]   // 버튼·링크·활성상태 전부 이 색
```

`brand` 스케일만 교체하면 antd `ConfigProvider` 테마(`lib/theme.ts`)·ag-grid 테마·로그인 그라디언트·로고·`ComponentsPage`의 스와치까지 전부 따라가요. **hex를 컴포넌트에 직접 쓰지 마세요** — `brand[N]` / `semantic.*` / `alpha(hex, a)`를 참조하세요(`CLAUDE.md` §6 컨벤션).

### `public/favicon.svg` (예외 — 정적 파일이라 별도 수정)

```svg
<stop stop-color="#4f46e5" />   <!-- brand[600] 과 같은 값으로 직접 교체 -->
<stop offset="1" stop-color="#4338ca" />  <!-- brand[700] -->
```

정적 SVG라 JS 변수를 못 써요. `brand[600]`/`brand[700]`을 바꿨으면 **이 파일의 hex 두 개도 같이 손으로 맞춰야** 브랜드가 완전히 일치해요.

**체크**: `npm run dev` 후 로그인 화면 그라디언트·사이드바·`/components` 페이지의 색 스와치가 새 브랜드로 보이는지 확인.

---

## 2. 메뉴 구성 — nav.tsx

`src/nav.tsx`의 `NAV_ITEMS`가 라우트·사이드바 메뉴·활성 표시(선택된 메뉴 강조)의 **단일 소스**예요. 이 배열 하나만 고치면 `App.tsx`(라우팅)와 `AppLayout`(사이드바)이 자동으로 따라와요 — 그 둘을 직접 손댈 필요 없어요.

```tsx
export const NAV_ITEMS: NavItem[] = [
  { path: '/', label: '대시보드', icon: <DashboardOutlined />, element: <DashboardPage /> },
  { path: '/apps', label: '앱', icon: <AppstoreOutlined />, element: <AppsPage /> },
  { path: '/apps/:slug', element: <AppDetailPage /> },  // label 없음 = 메뉴엔 안 뜨고 라우트만
  // … 이 앱에 필요 없는 화면은 통째로 줄 삭제
]
```

**이 앱에 필요 없는 화면 제거**: 예를 들어 발송(`/send`)·역할·권한(`/roles`)이 필요 없다면 `NAV_ITEMS`에서 해당 줄을 지우고 `src/pages/SendPage.tsx`/`RolesPage.tsx` 파일도 함께 삭제하세요(빌드에 안 잡히는 미사용 파일을 남겨두면 나중에 헷갈려요).

**상세/파라미터 라우트 추가**: `AppDetailPage`처럼 사이드바에 안 띄우고 라우트만 있는 화면은 `label`을 생략하면 돼요. `App.tsx`가 `NAV_ITEMS`를 순회하며 `label` 유무와 무관하게 전부 `<Route>`로 등록하고, `AppLayout`의 `MENU_ITEMS`(=`NAV_ITEMS.filter(n => n.label)`)만 사이드바에 표시해요.

---

## 3. Mock → 실서버 전환

기본은 **MSW mock**이라 백엔드 없이 전 화면이 동작해요. 실서버가 준비되면 코드 한 줄 안 고치고 `.env`만 바꾸면 돼요.

```bash
# .env (또는 .env.local)

# 로컬 dev 서버에서 Vite 프록시가 /api 를 전달할 백엔드 주소
VITE_PROXY_TARGET=http://localhost:8080

# dev 환경 백엔드 (factory dev start 의 프로브 대상)
VITE_PROXY_TARGET_DEV=https://api-dev.example.com

# 프록시 없이 절대 URL 로 직접 호출할 때만 (CORS 는 백엔드 책임)
# VITE_API_BASE=

# 수동으로 강제 지정하고 싶을 때만 — 보통은 factory 가 프로브해서 자동 결정하니 비워두세요
# VITE_USE_MOCK=false
```

**권장 흐름**: `VITE_USE_MOCK`을 직접 만지지 말고 `<repo명> local start`를 쓰세요 — `GET /api/admin/health`를 curl로 프로브해서 백엔드가 살아있으면 `VITE_USE_MOCK=false`로, 없으면 `true`(mock)로 **자동** 기동해요. 헤더의 MOCK 배지로 현재 모드가 표시돼요.

전환 시 확인할 것:

- [ ] `src/lib/types.ts`의 DTO가 실제 백엔드 응답 필드와 1:1인지 (`docs/api-contract/admin-api.md` 대조)
- [ ] `src/mocks/fixtures.ts`의 슬러그(`tradelog`/`gymlog`/`moodlog`)는 데모용 — 실서버 전환 후엔 무시되니 그대로 둬도 무방하지만, 로컬 mock 데모용 문구는 이 앱 도메인에 맞게 바꿔도 좋아요
- [ ] 로그인 자격증명이 실서버 admin 계정으로 바뀌었는지 (mock 전용 `admin@example.com`/`password` 안내 문구는 `USE_MOCK`일 때만 노출되니 신경 안 써도 됨 — `LoginPage.tsx` 참고)
- [ ] `npm run build` 그린

---

## 4. 화면 추가 패턴

### 그리드형(목록) 화면 — 가장 흔한 패턴

`GridPage` + `AdminDataGrid` + `useAdminList` 세 조합이면 **PC 페이지네이션 / 모바일 무한스크롤 + 카드 목록**이 전부 자동으로 배선돼요.

```tsx
// src/pages/XxxPage.tsx
import { useState } from 'react'
import { Form, Input, Alert } from 'antd'
import { useAdminList } from '../lib/useAdminList'
import { useAppOptions } from '../lib/useAppOptions'
import GridPage from '../components/GridPage'
import AdminDataGrid from '../components/AdminDataGrid'
import { getXxx } from '../api/client'   // client.ts 에 엔드포인트 함수 추가 필요
import type { Xxx } from '../lib/types'  // types.ts 에 DTO 추가 필요

export default function XxxPage() {
  const [form] = Form.useForm()
  const { options } = useAppOptions()      // 앱 셀렉트는 하드코딩 금지, 항상 이 훅
  const [filters, setFilters] = useState<{ slug?: string; query?: string }>({})

  const list = useAdminList<Xxx>({
    key: ['xxx', filters],
    fetchPage: (page, size) => getXxx({ ...filters, page, size }),
  })

  return (
    <GridPage
      title="Xxx"
      filters={<Form form={form} layout="inline" onFinish={(v) => { setFilters(v); list.resetPage() }}>{/* Form.Item… */}</Form>}
      onSearch={() => form.submit()}
      onReset={() => { form.resetFields(); setFilters({}); list.resetPage() }}
    >
      {list.isError ? (
        <Alert type="error" showIcon message="목록 오류" description={list.error?.message} />
      ) : (
        <AdminDataGrid<Xxx>
          columns={xxxColumns}          // ag-grid ColDef[] — 별도 lib/xxxColumns.tsx 로 분리 권장
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

1. `src/nav.tsx`의 `NAV_ITEMS`에 **한 줄** 추가: `{ path: '/xxx', label: '…', icon: <…/>, element: <XxxPage /> }`
2. 새 엔드포인트가 필요하면 `src/lib/types.ts`(DTO)·`src/api/client.ts`(함수)·`src/mocks/handlers.ts`+`fixtures.ts`(mock) 세 곳 동기화 — 실서버 계약은 `docs/api-contract/admin-api.md`와 대조
3. `npm run build` 그린 확인

---

## 5. 최종 체크리스트

- [ ] `appConfig.ts` — 앱 이름·로그인 문구 교체, `index.html`의 `<title>`도 맞춤
- [ ] `palette.ts` — `brand` 스케일 교체 + `public/favicon.svg`의 hex 2곳 수동 동기화
- [ ] `nav.tsx` — 이 앱에 필요한 메뉴만 남기고, 안 쓰는 페이지 파일도 삭제
- [ ] `src/mocks/` — 데모 데이터를 이 앱 도메인으로 교체 (또는 실서버 전환으로 무시)
- [ ] `src/api/client.ts` / `src/lib/types.ts` — 백엔드 계약과 1:1 확인 (`docs/api-contract/admin-api.md`)
- [ ] `.env` — `VITE_PROXY_TARGET`(/`_DEV`) 설정, `<repo명> local start`로 자동 mock↔실서버 전환 확인
- [ ] `npm run build` 그린

---

## 관련 문서

- [`../api-contract/README.md`](../api-contract/README.md) — 백엔드 계약 구성
- [`../api-contract/admin-api.md`](../api-contract/admin-api.md) — 엔드포인트 상세
- [`./screens.md`](./screens.md) — 기존 9개 화면이 각각 무엇을 하는지
- [`../../CLAUDE.md`](../../CLAUDE.md) — 단일 소스 표 · 명령어 참조
- [`../../README.md`](../../README.md) — 빠른 시작 · 폴더 구조
