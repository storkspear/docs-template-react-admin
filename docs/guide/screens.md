# 화면 가이드

`nav.tsx`(`NAV_ITEMS`)의 **사이드바 메뉴 화면 14개** + 헤더 계정 드롭다운으로 진입하는 설정(`/settings`) + 라벨 없이 라우트만 등록된 화면 6개(앱 상세 `/apps/:slug` · 매출 상세 `/insights/revenue` · 가입 상세 `/insights/signups` · 게시물 작성 `/content/new` · 게시물 상세 `/content/:id` · 전체 이벤트 `/analytics/events`)로 구성돼요. 각 화면이 무엇을 보여주는지, 어떤 API 를 쓰는지, 모바일에서 어떻게 달라지는지를 정리했어요.

사이드바 메뉴는 세 **그룹**으로 나뉘어요(`nav.tsx`의 `group`/`scope` — § "공통 패턴" 참고):

| 그룹 | 화면 | 노출 조건 |
|---|---|---|
| **공통** | 대시보드 · 서비스 현황 · 역할·권한 | 항상 (`scope: 'global'`) |
| **앱별** | 앱 버전 · 사용자 · 게시물 · 파일 · 결제 · 발송 · 매출 분석 · 이벤트 분석 · 감사 로그 · 시스템 설계도 | 헤더에서 **앱을 선택했을 때만** (`scope: 'app'`) |
| **템플릿** | 컴포넌트 | 항상 (템플릿 데모/도구) |

> 컬럼 정의·필터 폼 등 실제 코드는 각 화면 이름으로 `src/pages/`에서 바로 찾을 수 있어요 (예: 대시보드 → `src/pages/DashboardPage.tsx`).

**정체성 원칙**: 대시보드는 **공장 전체 뷰**(비교·순위·합산 추이·치명 신호)만 보여줘요 — 개별 앱을 깊이 파고드는 화면이 아니에요. 앱 하나의 지표·운영 신호·시계열·빌링·사용자를 전부 보고 싶으면 앱 상세(`/apps/:slug`)로 가세요 — 개별 앱의 **유일한 집**이에요. "이벤트 분석" 메뉴는 제품 이벤트 전용 화면이에요(§14) — 앱 지표·운영 신호·시계열은 앱 상세 개요탭이 담당해요.

---

## 공통 그룹

### 1. 대시보드 (`/`)

전사(fleet) KPI 카드 7개(총 사용자·신규 30일·DAU·MAU·매출 30일·환불 30일·실패 24시간), 앱별 매출 점유(도넛 + 범례), 전체 매출·가입 추이 차트, 운영 신호 요약, 최근 실패 5건, 고객 결제 TOP 5를 3행으로 보여줘요.

카드 높이와 이동 방식을 공통 래퍼 두 개로 통일했어요:
- **`StatCard`**(`src/components/StatCard.tsx`) — Row1 스탯 카드 7개 전용. `onClick` 있으면 카드 전체가 클릭 대상(커서 포인터 + hover 시 그림자), 없으면 순수 정보 카드. 카드별 `extra` 링크 예외가 없어 7카드 높이가 항상 동일해요.
- **`DashSectionCard`**(`src/components/DashSectionCard.tsx`) — Row2/Row3 섹션 카드 전용. `title` + `to?`(있으면 우상단에 화살표 아이콘 링크, 전 카드 동일 위치) + `bodyHeight`(행별 고정 — Row2 240px, Row3 260px) + `onClick?`(본문 전체 클릭 — 추이 카드용)로 구성돼서 "카드마다 제각각"이 구조적으로 불가능해요.

- **API**:
  - `GET /dashboard/metrics?window=30d` — Row1 스탯 7개 + Row2 매출 점유 도넛(`perSlug`) + Row3 운영 신호(`totals.renewalFailures7d`/`webhookPending`/`webhookFailed`)
  - `GET /analytics/revenue?slug=`, `GET /analytics/signups?slug=` — **슬러그마다 병렬 호출**(`useStackedTrend`, `useQueries`)해 상위 5개 앱은 각자 세그먼트·나머지는 '기타'로 합산한 적층 막대(`StackedBarChart`) → "전체 매출 추이"/"전체 가입 추이" 차트
  - `GET /audit-logs?result=FAILURE&size=5` — "최근 실패 5"
  - `GET /dashboard/top-customers?window=30d&size=5` — "고객 결제 TOP 5"
- **인터랙션**(이동 목적지 있는 카드는 전부 클릭형, extra 링크 텍스트는 없음):
  - Row1: 총 사용자 카드 클릭 → `/users`, 매출 카드 클릭 → `/payments`, 환불 카드 클릭 → `/payments?status=REFUNDED`, 실패 카드 클릭 → `/audit-logs?result=FAILURE`. 신규/DAU/MAU 는 클릭 없음(정보 전용).
  - Row2: "앱별 매출 점유" 우상단 화살표 → `/apps`, 도넛 세그먼트/범례 행 클릭 → 앱 상세(`/apps/{slug}`). "전체 매출 추이"/"전체 가입 추이"는 우상단 화살표·본문 전체 클릭 모두 전용 상세 페이지(`/insights/revenue`·`/insights/signups`, `src/pages/InsightDetailPage.tsx`)로 이동해요 — 날짜×앱 표 + 합계 + CSV 내보내기로 드릴다운해요.
  - Row3: "최근 실패 5" 우상단 화살표 및 행 클릭 → `/audit-logs?result=FAILURE`(감사로그 화면이 이 쿼리를 초기 필터로 반영). "고객 결제 TOP 5" 우상단 화살표 및 행 클릭 → `/users`(앱 필터 연동은 v2). "운영 신호"는 이동 목적지 없음(정보 전용).
- **차트**: `MiniPieChart`(`src/components/MiniPieChart.tsx`)는 의존성 0 SVG 도넛(stroke-dasharray 세그먼트) — 중앙에 총액 표시, hover 시 세그먼트 강조 + 툴팁(`<title>`), 세그먼트/범례 클릭으로 `onSegmentClick(label)` 콜백.
- **모바일**: Row1~3 전부 `Col xs={24}`로 세로 스택.

### 2. 서비스 현황 (`/apps`)

운영 중인 앱들의 배포·규모·매출·이슈를 한 보드에서 보여주는, 앱별 드릴다운의 진입점이에요. 행을 클릭하면 앱 상세로 이동해요. (사이드바 라벨은 "서비스 현황" — 앱별 매출 드릴다운은 앱별 그룹의 "매출 분석"(`/revenue`)이 담당해요.)

- **API**: `GET /apps`
- **모바일**: 카드 목록으로 전환. 앱 개수가 적어 페이지네이션 없음(`AdminMiniGrid`).

### 3. 앱 상세 (`/apps/:slug`) — 메뉴 밖 라우트

개별 앱의 **유일한 집** — 개요탭에 지표 카드·빌링 요약·운영 신호(갱신 실패·웹훅 미처리/실패·리텐션 D1/D7)·시계열 차트(DAU·신규가입·매출)까지 한 화면에서 다 봐요. 사용자탭은 그 앱 사용자 목록.

- **운영 신호 드릴다운**: "갱신 실패" 카드 클릭 → 실패 건 드로어(시각·구독자·시도·상태(재시도 대기/소진)·에러·다음 재시도) — 구독자 이메일이 실려 `PERM_PAYMENTS_READ` 세션에만 클릭이 열려요. "웹훅 미처리 · 실패" 카드 클릭 → 이벤트 드로어(미처리/실패 세그먼트, payload 미노출).
- **API**(개요탭): `GET /apps/{slug}/metrics`, `GET /apps/{slug}/billing`, `GET /apps/{slug}/ops`(+드릴다운 `GET /apps/{slug}/ops/renewal-failures` §48 · `GET /apps/{slug}/ops/webhooks?status=` §49), `GET /analytics/{metric}?slug=`(`metric`=`dau`/`signups`/`revenue`, 3번 호출)
- **컴포넌트**: 운영 신호 4카드는 `src/components/OpsSignalCards.tsx`(props: `slug`), 시계열 3차트는 `src/components/AnalyticsCharts.tsx`(props: `slug`) — 둘 다 재사용 가능한 컴포넌트로 추출돼 있어요(구 `AnalyticsPage`의 블록).
- **API**(사용자탭): `GET /apps/{slug}/users?query=&page=&size=`, `GET /apps/{slug}/users/{userId}`(드로어)
- **모바일**: 개요탭 카드들이 세로 스택. 사용자탭은 무한스크롤 + 카드 목록(아래 "8. 사용자"와 동일 패턴).
- `nav.tsx`에 `label` 없이 등록 — 사이드바 메뉴엔 안 뜨고 라우트만 잡혀요.

### 4. 추이 상세 (`/insights/revenue` · `/insights/signups`) — 메뉴 밖 라우트

대시보드의 "전체 매출 추이"/"전체 가입 추이" 카드에서 진입하는 드릴다운 전용 페이지예요(`InsightDetailPage`, `metric` prop 으로 매출/가입 분기). 날짜×앱 표에 앱별 값과 합계를 펼쳐 보여주고, CSV 로 내보낼 수 있어요.

- **API**: `GET /analytics/revenue?slug=` · `GET /analytics/signups?slug=` — 대시보드와 동일하게 슬러그마다 병렬 호출(`useStackedTrend`)해 앱별 시리즈를 조립해요.
- **권한**: `PERM_DASHBOARD_READ`(대시보드와 동일 게이트 — 대시보드 카드의 드릴다운이라서요).
- **모바일**: 표가 가로 스크롤로 대응.
- `nav.tsx`에 `label` 없이 등록 — 사이드바 메뉴엔 안 뜨고 라우트만 잡혀요.

### 5. 역할·권한 (`/roles`)

RBAC 운영 화면 — **(1) 역할×권한 매트릭스 편집**과 **(2) 관리자 계정 관리** 두 블록이에요. 전부 실 API 에 배선돼 있고, `PERM_ADMIN_MANAGE`(기본 grant 로는 master 전용)가 필요해요. 저장·계층·의존 검증은 백엔드가 재검증하고, 화면은 UX(토글·의존 자동반영·잠금 표시)만 담당해요.

- **매트릭스**: 메뉴(1-depth)/세부 권한(2-depth) 트리 표 × 역할(viewer/support/admin/master) 열. 편집 가능한 칸은 토글(Switch), 고정 권한(대시보드·설정 등 `FIXED`)과 상급·본인 티어 칸은 잠김("권한있음/권한없음" 흐림 표시), master 열은 항상 전체 권한(열 병합). `UNMASK`/`WRITE` 토글 시 짝이 되는 `READ`가 자동으로 켜져요(백엔드 의존 규칙과 동일). "권한 저장"은 목표 집합을 통째로 저장하고, 대상 계정은 다음 로그인부터 반영돼요.
- **계정 관리**: 계정 목록(이메일·표시이름·역할 태그) + "관리자 추가" 모달(이메일·비밀번호·역할 — 자기보다 낮은 티어만) + 역할 변경 셀렉트 + **정보변경 모달(이름 + 새 비밀번호 — 비밀번호는 입력했을 때만 재설정)** + 삭제(확인 모달). 본인·마지막 master 는 삭제/강등이 서버에서 거부돼요. 두 표 모두 Card(흰 배경) 안에 `bordered` 로 렌더하고, 매트릭스 1-depth 메뉴 행은 전부 굵게 표시해요.
- **API**: `GET/PUT /roles/permissions`, `GET/POST /admins`, `PATCH/DELETE /admins/{id}`, `POST /admins/{id}/password` (계약·에러 코드는 [`admin-api.md`](../api-contract/admin-api.md) §31~§38)
- **모바일**: 별도 카드 전환 없이 `antd Table`이 가로 스크롤(`scroll={{ x: 'max-content' }}`)로 대응.

### 6. 설정 (`/settings`) — 헤더 계정 드롭다운 진입

계정 정보(이메일 표시), 본인 비밀번호 변경, **영수증 관리** 세 카드로 이뤄진 "폼 페이지" 표준 레퍼런스예요. **사이드바 메뉴가 아니라 헤더 우상단 계정 드롭다운의 [내 정보]로 들어가요**(`nav.tsx` 에서 `label` 은 두고 `group` 만 빼면 사이드바엔 안 뜨면서 헤더 섹션명·활성 판정은 정상 동작해요 — 라벨까지 빼면 대시보드로 잘못 잡혀요). 설정에 노출되는 항목은 전부 실제 저장 동작이 붙어 있어요. 테마 전환은 **헤더 우상단 전구 아이콘**(`AppLayout`)이 담당하므로 설정에 중복 노출하지 않아요. 카드마다 자체 저장 버튼을 가져요.

- **API**: `POST /me/password` — 비밀번호 변경(`changeMyPassword`, 현재 비밀번호 불일치면 `ADMIN_016` 메시지 표시). 계정 카드의 이메일은 읽기 전용 표시. 테마 토글은 `useThemeMode`(localStorage 지속) — 위치는 헤더.
- **파생 레포 참고**: 설정 항목을 추가할 땐 **저장 동작이 실제로 붙는 항목만** 노출하세요(Card + Form + 카드별 저장 버튼 패턴).
- **영수증 관리 카드**: 이메일·사업자등록번호·사업장 주소·사업장명·핸드폰번호를 입력·저장해요(`GET/PUT /settings/receipt` — mock 전용, admin-api.md §18d). 이 정보는 결제 영수증 발송 시 발행처 표기에 사용돼요(미리보기·이메일 동일 적용).
- **모바일**: 최대폭 컬럼이 화면 폭에 맞춰 줄어들 뿐 별도 분기 없음.

---

## 앱별 그룹 (헤더에서 앱 선택 시 노출)

### 7. 앱 버전 (`/app-versions`)

선택된 앱의 iOS·Android **2단계 최소 버전 규칙**(강제/경고)을 관리해요. 플랫폼별 아코디언 2행 고정이고, 접힘 상태에도 헤더에 강제·경고 기준값과 사용 여부 표시등이 보여요.

- **2단계 규칙**: 설치 버전이 `forceMinVersion` 미만이면 **닫을 수 없는 차단**, `warnMinVersion` 미만(강제 기준 이상)이면 **닫을 수 있는 경고**, 그 외엔 통과. 두 값 모두 nullable — 비우면 **정책 없음**(`null`)이에요.
- **입력**: major.minor.patch **3칸 숫자 스테퍼**(`VersionStepper`)라 구조적으로 항상 유효한 semver 가 나와요. 세 칸을 모두 비우면 `null`(미설정)을 방출해요.
- **미리보기**(`AppVersionPreview`): 사용자가 보게 될 두 화면(강제/경고)을 세그먼트 탭으로 직접 보여줘요 — flutter `_ForceUpdateGate` 를 모사(둘 다 정중앙 `AlertDialog` + 어두운 배리어, 차이는 닫기 가능 여부뿐).
- **저장**: 플랫폼별 [저장] — bulk-replace API 라 두 행을 다 보내되, 반대편은 서버 원본(없으면 빈 규칙)을 그대로 재전송해 미저장 변경이 딸려가지 않아요. 저장 전 semver 파싱과 `강제 ≤ 권장` 을 검증해요.
- **API**: `GET/PUT /apps/{slug}/app-versions` (계약은 [`admin-api.md`](../api-contract/admin-api.md) §44~§45)
- **권한**: 조회 `PERM_APP_VERSION_READ`, 저장 `PERM_APP_VERSION_WRITE` — 읽기 권한만 있으면 화면은 보되 토글·저장이 잠겨요.
- **모바일**: 아코디언·미리보기가 세로 스택.

### 8. 사용자 (`/users`)

상단 앱 컨텍스트로 선택된 앱의 사용자 목록을 검색·조회해요. 앱 상세의 사용자 탭과 같은 컬럼·드로어를 최상위 메뉴로 뺀 화면이에요(둘 다 `userColumns` 공유).

- **딥링크**: `?slug=` 쿼리 파라미터로 진입하면(대시보드 "전체 가입 추이" 드릴다운인 가입 상세(`/insights/signups`) 등) `useSearchParams`로 마운트 시 1회 읽어 앱 컨텍스트에 반영해요(감사로그의 `?result=FAILURE` 패턴과 동일).
- **PII 마스킹**: `PERM_USERS_UNMASK` 없는 세션(기본 grant 기준 support)은 이메일·닉네임 등이 마스킹돼 보여요. 상세 드로어의 "조회" 버튼으로 단건 원본을 열람하면 서버가 `user_read_history`에 기록해요 — § "공통 패턴" 참고.
- **API**: `GET /apps/{slug}/users?query=&page=&size=`, `GET /apps/{slug}/users/{userId}` (드로어), `GET /apps/{slug}/users/{userId}/reveal` (원본 조회)
- **모바일**: PC 는 필터가 그리드 위 인라인 바, 모바일은 제목 옆 "필터" 버튼 → 풀스크린 모달. 목록은 무한스크롤 + 카드 목록으로 전환(`useAdminList`가 PC=페이지네이션/모바일=무한스크롤 자동 분기).

### 9. 게시물 (`/content`) + 게시물 상세 (`/content/:id`)

선택된 앱의 **공유(공개) 게시물**을 전량 조회하고 숨김/삭제/복원하는 콘텐츠 모더레이션 화면이에요. 프라이빗 기록은 이 화면에 안 와요(공개 전용). 파일 화면과 동일한 모더레이션 UX(상태 태그·행 틴트·사유 모달·복원 컨펌)를 써요.

- **목록**(`ContentPage`): 상태(공개/숨김/삭제)·게시판(board, 고정 목록)·일련번호·제목·작성자 5종 필터 + 숨김일시 컬럼 + `GridPage`+`AdminDataGrid`+`useAdminList` 표준 목록. 행 클릭 → 게시물 상세로 이동.
- **상세**(`ContentDetailPage`, 메뉴 밖 라우트): **아티클 리더 레이아웃** — 모더레이터가 게시물을 실제 읽는 형태로 제목·작성자 바이라인 pill(아바타·이름·상대시간)·본문 전체·첨부이미지(파일 콘솔과 동일 원본, presigned URL)를 보고, 그 자리에서 숨김(사유)/삭제(사유)/복원/삭제 대상 복원을 실행해요.
- **작성**(`ContentComposePage`, `/content/new`): 흰 종이 캔버스 에디터(Tiptap). **상단 sticky 바가 `[목록] · [카테고리 선택]` (좌) / `[폭 토글] [등록]` (우)** 구조예요 — 카테고리(board)는 글의 내용이 아니라 발행 설정이라 본문 캔버스에서 분리해 이 바에 뒀어요(`UnderlineSelect`, 고정 4종 + "카테고리 선택"으로 해제). 본문 캔버스는 제목 → 해시태그 → 에디터 순서로 글 자체에만 집중해요.
- **본문 폭**(`usePaperWidth`): 작성 화면의 폭 토글(기본 760px ↔ 넓게)에서 정한 폭이 **글 속성 `properties.wide` 로 저장**되고, 상세(읽기)가 그 값으로 렌더해요 — **넓게 쓴 글은 넓게, 좁게 쓴 글은 좁게 읽혀요**. 상세엔 폭 토글을 두지 않아요(읽는 사람이 바꿀 값이 아니라 글의 속성이라서요). localStorage(`admin.composeWide`)는 "다음 새 글의 기본 폭"으로만 쓰여요.
- **soft-delete**: 파일과 동일 — 삭제는 사유 필수 + 30일 후 purge, 그 전엔 복원 가능(`purgeAt` D-day).
- **API**: `GET /apps/{slug}/content?board=&status=&page=&size=`, `GET .../content/{id}`, `POST .../content/{id}/hide|restore|restore-deleted`, `DELETE .../content/{id}`, `POST .../content`(작성, §39), `POST .../content/uploads`(이미지 선업로드, §41) (계약은 [`admin-api.md`](../api-contract/admin-api.md) §25~§30·§39·§41)
- **권한**: 조회 `PERM_CONTENT_READ`, 작성·수정·공개·숨김·복원 `PERM_CONTENT_MODERATE`, 삭제 `PERM_CONTENT_DELETE`. 공개 게시물이라 작성자 마스킹은 없어요.
- **모바일**: 목록은 무한스크롤 + 카드 목록, 상세는 세로 스택.

### 10. 파일 (`/files`)

선택된 앱 스토리지의 업로드 파일을 조회·모더레이션(검역/삭제/복원)하는 화면이에요. **서버 페이지네이션**(`PageResponse` — 사용자·결제와 동일 계약)이라 `useAdminList`를 그대로 써요.

- **타입 탭**(`ViewTabs`): **전체 / 이미지 / 영상 / 오디오 / 삭제 대상** — 이미지·영상·오디오는 서버 `kind` 필터, "삭제 대상"은 soft-delete 된 파일(`status=deleted`)만 따로 봐요.
  - **전체·삭제 대상**: `AdminDataGrid` — 데스크톱 번호 페이지네이션 / 모바일 무한스크롤(`useAdminList` 표준).
  - **오디오**: 커스텀 `AudioList`(데이터는 `useAdminList` — 데스크톱 번호 페이지네이션 / 모바일 무한스크롤) — 행에서 바로 재생.
  - **이미지·영상**: 그리드 뷰는 썸네일 **미디어 그리드**(`MediaGrid`, `useInfiniteQuery` 무한스크롤, 한 번에 24개), 리스트 뷰는 `AdminMiniGrid`(`mediaListColumns`) — 그리드/리스트 뷰 토글이 따로 있어요.
- **필터**: 파일명(원본 파일명/key 부분일치) · 업로더 · 출처(사용자/게시물/그 외) · 상태(정상/검역) · 생성기간 · 수정기간(RangePicker 2개). 정렬 셀렉트(최신순/오래된순/용량순 — 현 페이지 한정) + 북마크(즐겨찾기)만 보기 토글.
- **미리보기·라이트박스**: 이미지·영상 셀/썸네일 클릭 → 반응형 **`MediaModal`**(라이트박스)로 크게 봐요(새 탭 아님). `url`은 presigned GET(~10분)이라 만료되면 재검색 안내가 떠요.
- **상세 드로어**(`FileDetailDrawer`): 파일 메타(원본 파일명·MIME·크기·연관 대상(게시물/사용자)·업로드/수정 시각) + 업로더·IP·기기 정보. 업로더·IP·기기는 `PERM_FILES_UNMASK` 없으면 마스킹 — "업로더·IP·기기 조회" 버튼으로 단건 원본 열람(`GET .../files/{key}/reveal`, 열람 기록됨).
- **미디어 열람 게이팅**: 이미지·영상·오디오 원본은 각각 `PERM_FILES_IMAGE`·`PERM_FILES_VIDEO`·`PERM_FILES_AUDIO` 가 있어야 탭·썸네일·상세에서 보여요. 없으면 탭 자체가 감춰지고(URL `?type=image` 우회도 차단), 목록 썸네일은 종류 뱃지로, 상세는 "미디어 열람 권한이 없어요" 안내로 대체돼요.
- **액션**(권한 분리 게이팅 — 검역·검역복원 `PERM_FILES_QUARANTINE` / soft-delete·삭제복원 `PERM_FILES_DELETE`. 권한별로 버튼이 개별 노출돼요):
  - **검역**: 확인 모달("사용자에게 안 보이게 숨겨요. 언제든 복원할 수 있어요.") → `POST .../quarantine`. 검역된 파일은 "복원" 버튼으로 원상 복구.
  - **삭제**: **사유 선택 모달(필수)** → `DELETE .../files?key=` — 즉시 영구삭제가 아니라 **soft-delete**예요. 사유는 콤보박스 프리셋(모더레이션 5종: 음란물·선정적 / 폭력적·혐오 / 개인정보 노출 / 저작권·초상권 침해 / 스팸·광고·사기 + 운영 3종: 사용자 삭제 요청 / 중복·잘못 업로드 / 테스트·임시 파일 정리)에서 고르고, "직접 입력…"을 선택했을 때만 자유 입력창이 열려요(`useReasonModal` 의 `presets` 옵션 — 사유 문구가 통일돼 감사로그 집계가 가능해져요). "삭제 대상으로 옮겼어요 — 30일 후 영구삭제돼요." 토스트가 뜨고, "삭제 대상" 탭에 삭제일·사유·**영구삭제 D-day**(`purgeAt`) 컬럼과 함께 나타나요.
  - **삭제 대상 복원**: `POST .../restore-deleted` — purge 전이면 언제든 정상으로 되돌려요.
- **모바일**: 필터는 다른 그리드 화면과 동일하게 "필터" 버튼 → 풀스크린 모달. 목록/그리드는 화면 폭에 맞춰 전환.

### 11. 결제 (`/payments`)

선택된 앱의 결제 내역을 이메일·채널·상태·유형·기간으로 필터링해서 봐요. "누가·언제·얼마"를 드릴다운으로 확인하는 화면이에요. 대시보드의 "매출 (30일)" 카드 클릭으로 `/payments`, "환불 (30일)" 카드 클릭으로 `/payments?status=REFUNDED`(환불 건만 미리 필터링된 상태)로, "전체 매출 추이" 드릴다운인 매출 상세(`/insights/revenue`)에서 `/payments?slug={slug}`(해당 앱 미리 선택된 상태)로 진입할 수 있어요.

- **딥링크**: `?status=REFUNDED`·`?slug=` 쿼리 파라미터로 진입하면(대시보드 "환불 (30일)" 카드·추이 상세(`/insights/*`) 등) `useSearchParams`로 마운트 시 1회 읽어 상태 필터·앱 컨텍스트에 반영해요(감사로그의 `?result=FAILURE` 패턴과 동일).
- **API**: `GET /apps/{slug}/payments?query=&channel=&status=&type=&from=&to=&page=&size=`, `POST /apps/{slug}/payments/{paymentId}/refund` (환불), `GET /apps/{slug}/payments/{paymentId}/refunds` (환불 이력), `GET .../receipt` + `POST .../receipt-email` (영수증 조회·발송 — mock 전용, admin-api.md §18b~§18c)
- **컬럼**: 일련번호·상태(`StatusBadge` — 한글 라벨 `lib/paymentStatus`: 결제완료 success/부분환불 warning/환불완료 neutral/실패 danger, 대기(`READY`) info)·채널(`StatusBadge` info)·유형(`StatusBadge` — 구독 brand/일반 neutral)·이메일(복사 버튼)·금액(가운데 정렬 — 부분/전액 환불 시 아래 `환불 −₩…` 누적액)·결제일시·환불일시(`null`이면 `—`)·액션(PG·`PAID`/`PARTIALLY_REFUNDED` 행만 "환불" 버튼, IAP 미환불 행은 툴팁 + `—`. 환불완료(`REFUNDED`)·부분환불 건은 채널 무관하게 [영수증] 버튼이 환불 가능 시 [환불]과 함께 노출). 행 틴트: 부분환불 amber·전액환불 옅은 회색.
- **결제 상세·환불 이력 팝업**(읽기 전용): 금액/상태 셀 클릭으로 진입. 영수증 스타일(결제 금액 − 환불됨 = 환불 가능 잔액, 잔액만 강조) + 일시 + 환불 이력 원장(건별 금액·사유·시각·처리자). 환불 가능(PG·PAID/부분환불)하고 쓰기 권한이 있으면 [환불 진행] 버튼이 환불 모달로 연결.
- **환불 모달**(중앙): 컨텍스트 서브라인(이메일·유형·채널) + 환불 가능 잔액 히어로 + 금액 입력([전액]·[잔여기간 비례 ₩N] 프리셋 칩 — 일할계산식은 칩 툴팁, 계산 불가 사유(기간 만료 등)는 disabled 칩 툴팁)·상태 예고 캡션·**사유(프리셋 콤보 — 단순 변심/구독 해지/중복 결제/결제 오류/불만족/직권 취소, "직접 입력…" 선택 시에만 TextArea)**. 잔액 다 환불 시 `REFUNDED`, 일부면 `PARTIALLY_REFUNDED`(재환불 가능). 환불 버튼은 `PERM_PAYMENTS_WRITE` 게이팅. 환불 이력은 상세 팝업이 전담("이력 보기" 링크로 이동).
- **영수증 팝업**: 환불 성공 직후 자동으로 열리고, 상세 팝업의 [영수증] 버튼(환불완료/부분환불 건)으로도 진입해요. 서버가 조립한 payload(§18b — 서비스명·영수증번호·결제수단(신용카드는 마스킹 카드번호)·거래번호·금액 원장·발행처)를 그대로 렌더하고, [이미지로 저장]으로 같은 데이터를 JPEG(캔버스 렌더)로 내려받을 수 있고, [이메일로 발송]은 **같은 payload** 로 만든 영수증을 결제자에게 보내요(§18c) — 미리보기와 발송본이 항상 일치. 발행처 정보는 설정 > 영수증 관리에서 입력해요.
- **모바일**: PC 는 필터가 그리드 위 인라인 바, 모바일은 제목 옆 "필터" 버튼 → 풀스크린 모달. 목록은 무한스크롤 + 카드 목록으로 전환(`useAdminList`).

### 12. 발송 (`/send`)

선택된 앱 사용자에게 SMS/이메일/푸시를 보내는 화면이에요. "보내기"와 "발송 이력" 두 뷰(`ViewTabs`)로 구성돼요.

> ⚠ **실 API 배선 + mock 전용**: 이 화면은 발송 6개 엔드포인트(세그먼트·미리보기·수신자 목록·발송·이력·전달 상태)에 **실제로 배선돼** 있지만, `template-spring`엔 아직 대응 컨트롤러가 없어요 — **mock(MSW) 모드에서만 동작**하고 실서버 모드에선 404가 나요. 계약 상세는 [`admin-api.md`](../api-contract/admin-api.md) §M1~§M6 참고.

- **보내기(compose)**: 채널(SMS/이메일/푸시) 토글 + 대상(전체/프리미엄/무료/활성/신규/특정 사용자/토픽 — 토픽은 푸시 전용) + 제목/내용. 채널×대상의 **유효 수신자 수 미리보기**(`GET .../messages/preview`)와 **실제 수신자 목록**(`GET .../messages/recipients`)을 발송 전에 확인하고, 확인 모달을 거쳐 발송해요. 실기기 모양의 미리보기(`DevicePreview`), SMS 는 byte 카운터(90byte 초과 시 LMS 표시), 채널 미지원 앱(예: SMS 는 phone-auth 앱만)은 안내 Alert 로 차단돼요.
- **발송 이력(history)**: 채널·기간 필터 + 발송 건 목록(발신자·대상·건수·결과). 건수를 클릭하면 수신자별 전달 상태(DELIVERED/FAILED/PENDING)를 봐요.
- **API**: `GET .../messages/segments`, `GET .../messages/preview`, `GET .../messages/recipients`, `POST .../messages`, `GET .../messages`(이력), `GET .../messages/{id}/recipients`
- **권한**: 채널별 — `PERM_SEND_SMS`·`PERM_SEND_EMAIL`·`PERM_SEND_PUSH`. 하나라도 있으면 메뉴가 열리고, 발송 탭의 채널 선택과 이력 검색의 채널 필터엔 **권한 있는 채널만** 노출돼요(예: SMS 권한만 있으면 SMS 만 보임).
- **모바일**: 폼이 세로 스택으로 쌓이고, 이력 목록은 카드 전환.

---

### 13. 매출 분석 (`/revenue`)

선택된 앱의 매출을 기간(프리셋 + 절대 날짜 RangePicker)으로 조회하는 앱별 빌링 심화 화면이에요. 순매출·총매출·환불·활성구독 KPI + 일별 매출 차트 + 최근 거래 표로 구성돼요.

- **KPI 드릴다운**: KPI 카드를 누르면 결제·사용자 화면의 해당 목록으로 이동해요 — 거래 상세는 단일 화면(결제)에서 관리해요.
- **차트**: 공유 `MiniBarChart`(accent 색 + X/Y축)와 `Sparkline` — 대시보드·이벤트분석·앱개요와 동일 톤·축이에요(`useChartAccent` 단일 소스).
- **표기 단일 소스**: 상태·유형 뱃지는 `lib/paymentStatus`(`paymentStatusLabel`/`paymentStatusColor`/`paymentTypeLabel`)를 써서 결제 화면과 항상 같은 색·같은 글자로 보여요.
- **CSV 내보내기**: 조회 기간의 거래를 CSV 로 내려받아요.
- **API**: `GET /apps/{slug}/billing?from=&to=`, `GET /apps/{slug}/payments?from=&to=&size=`
- **권한**: `PERM_APPS_READ` — 앱 빌링 데이터라 앱 조회 권한과 같은 게이트예요.
- **모바일**: KPI·차트·표가 세로 스택.

### 14. 이벤트 분석 (`/analytics`) + 전체 이벤트 (`/analytics/events`)

선택된 앱의 **제품 이벤트**(백엔드 `@TrackEvent` 자동 계측 → `analytics_daily` 야간 롤업) 요약·추이 화면이에요. 이벤트는 행동+메타데이터만 있고 콘텐츠 내용은 없어요(개발 방침).

- **메인**(`AnalyticsPage`): 기간 프리셋 + **절대 날짜 RangePicker** + 상단에 선택 이벤트의 일별 추이 차트(풀폭, `MiniBarChart`) + 발생수 **상위 5** 이벤트 표(발생수·비중·순사용자). "전체 이벤트 보기" 버튼으로 하위 페이지 진입. 우상단 **[지금 집계]** 버튼(`rollupAnalytics`)은 야간 롤업을 기다리지 않고 오늘 발생분을 즉시 `analytics_daily` 로 집계해 하루 지연을 해소해요(집계된 slug 수·이벤트명 수·집계 기준 시각을 토스트로 안내).
- **전체 이벤트**(`AnalyticsEventsPage`, 메뉴 밖 라우트): 상위 5에 안 드는 사소한 이벤트까지 전량 표시. 행 클릭 시 상단 추이 차트가 그 이벤트로 갱신돼요.
- **API**: `GET /analytics/events?slug=&from=&to=`(요약), `GET /analytics/events/{eventName}?slug=`(단일 이벤트 추이), `POST /analytics/rollup?slug=`(온디맨드 집계)
- **권한**: 조회 `PERM_ANALYTICS_READ`(이벤트 분석 전용, 읽기 전용 메뉴). 대시보드·매출분석의 비즈니스 지표(`/analytics/{metric}`)는 `PERM_APPS_READ` 로 분리돼요 — 제품 이벤트 집계(`@TrackEvent`)와 매출/DAU 시계열은 별개 권한이에요. 운영자 행위 감사(`감사로그`, `PERM_AUDIT_READ`)와도 데이터가 겹치지 않아요(고객 행동 집계 vs 운영자 조작 이력).
- **모바일**: 차트·표가 세로 스택.

### 15. 감사 로그 (`/audit-logs`)

관리자·시스템 액션 로그를 slug·actor 이메일·action·result·IP·기간으로 필터링해서 봐요.

- **API**: `GET /audit-logs?slug=&actorEmail=&action=&result=&ipAddress=&from=&to=&page=&size=`
- **CSV 내보내기**: 현재 로드된 로그를 CSV 로 내려받아요(`PERM_AUDIT_EXPORT` 게이팅 — 권한 없으면 버튼 미노출).
- **딥링크**: `?result=FAILURE` 쿼리 파라미터로 진입하면(대시보드 "최근 실패 5" 등) `useSearchParams`로 마운트 시 1회 읽어 결과 필터 폼 초기값 + 필터 상태에 반영해요.
- **모바일**: 제목 옆 "필터" 버튼(적용 개수 배지) → 풀스크린 모달(RangePicker 포함, 모달 내부에 팝업 렌더). 목록은 무한스크롤 + 카드 목록.

### 16. 시스템 설계도 (`/schema`)

선택된 앱 DB 스키마를 읽기 전용 **ERD 콘솔**로 보여줘요(`@xyflow/react` 기반 그래프). 테이블·컬럼·관계를 탐색해 백엔드 데이터 모델을 파악하는 용도예요.

- **번들 분리**: react-flow 가 무거워서 `SchemaPageLazy` 가 `React.lazy` + `Suspense` 로 별도 청크로 분리해요 — 메인 번들 크기는 그대로예요.
- **API**: `GET /apps/{slug}/schema`
- **권한**: `PERM_SCHEMA_READ` — 데이터 모델 노출이라 `PERM_APPS_READ` 재사용 대신 별도 권한이고, 기본 grant 로는 admin·master 만 가져요(권한 매트릭스에서 `PERM_APPS_READ` 를 함께 요구해요).
- **모바일**: 그래프가 팬/줌으로 대응.

## 템플릿 그룹

### 17. 컴포넌트 (`/components`)

이 템플릿이 제공하는 색·타이포·버튼·폼 입력·탭·태그·피드백·Statistic/차트를 한 화면에 모은 디자인 시스템 갤러리예요. 파생 레포가 재사용 가능한 UI 조각을 한눈에 확인하는 용도.

- **API**: 없음.
- **모바일**: 섹션이 세로로 쌓이고 `Space wrap`으로 요소가 줄바꿈돼요.

---

## 메뉴 밖 라우트

| 라우트 | 화면 | 비고 |
|---|---|---|
| `/login` | `LoginPage` | `POST /auth/login`. Mock 모드에서만 데모 자격증명이 채워짐(`master@example.com` / `password`). mock 은 이메일 **프리픽스**로 역할을 결정해요 — `viewer@`/`support@`/`admin@`/`master@`(그 외 전부 master)로 로그인하면 티어별 화면 차이를 바로 시연할 수 있어요 |
| `/apps/:slug` | `AppDetailPage` | 위 "3. 앱 상세" 참고 |
| `/insights/revenue` | `InsightDetailPage` | 대시보드 "전체 매출 추이" 드릴다운 — 날짜×앱 표 + 합계 + CSV |
| `/insights/signups` | `InsightDetailPage` | 대시보드 "전체 가입 추이" 드릴다운 — 날짜×앱 표 + 합계 + CSV |
| `/content/:id` | `ContentDetailPage` | 위 "9. 게시물" 참고 |
| `/content/new` | `ContentComposePage` | 위 "9. 게시물" 작성 항목 참고 — 목록 우상단 [작성하기]로 진입 |
| `/analytics/events` | `AnalyticsEventsPage` | 위 "14. 이벤트 분석" 참고 |

`/login`만 `App.tsx`에 별도 `<Route>`로 등록되고, 나머지는 전부 `nav.tsx`에 `label` 없이 등록돼요 — 사이드바 메뉴엔 안 뜨고 라우트만 잡혀요.

## 공통 패턴

### 디자인 시스템 (2026-07 리프레시)
- **무드(스킨)**: `src/lib/moods.ts` 의 4종(Obsidian=블랙 미니멀 / Blurple=Stripe 풍 / Ember=슬레이트+오렌지 / Glacier=Linear 풍 고밀도) × 라이트/다크. 헤더 팔레트 아이콘으로 전환, localStorage 지속. antd 테마·AG Grid·사이드바·뱃지 문법이 전부 무드에서 파생돼요.
- **상태 뱃지**: `StatusBadge` + 의미 키(success/warning/danger/info/neutral/brand). antd `<Tag color="green">` 는 쓰지 않아요(예외: closable 칩·종이 캔버스 전용 칩).
- **아이콘**: lucide(1.75px 스트로크)를 `src/lib/icons.tsx` 가 antd 이름으로 재-export — 사용처는 이름 그대로, import 경로만 이 모듈.
- **필터 바**: CSS Grid(`repeat(auto-fit, minmax(200px,1fr))`) + 라벨 상단 — 브라우저 폭과 무관하게 정연한 격자, 기간(RangePicker)은 2칸.



- **앱 컨텍스트**: 헤더의 앱 스위처로 앱을 선택하면 `scope: 'app'` 메뉴(앱별 그룹)가 사이드바에 나타나고, 해당 화면들은 전부 선택된 앱 기준으로 조회해요. 앱 미선택 시엔 "상단에서 앱을 선택하세요" Empty 상태를 보여줘요.
- **RBAC 게이팅**(`src/lib/rbac.ts`): 로그인 응답의 `permissions`(효과 권한 `PERM_*`)로 — ① 사이드바 메뉴를 `canReadKey`로 거르고(권한 없는 메뉴는 아예 안 보임), ② 화면 안의 쓰기 액션(환불·검역·삭제·발송 등)을 `canWrite`(메뉴 단위) + `hasPerm(PERM.*)`(액션 단위 세분화)로 숨겨요. 이건 UX 게이팅일 뿐이고 최종 강제는 백엔드(`hasAuthority`)예요.
- **PII 마스킹 + "조회"(reveal)**: 사용자(이메일·닉네임 등)와 파일(업로더·IP·기기)은 `PERM_*_UNMASK` 없는 세션에 마스킹돼 내려와요. 상세 드로어의 "조회" 버튼으로 **단건 원본**을 열람하면 서버가 `user_read_history`에 열람 기록을 남기고, 드로어를 다시 열면 다시 마스킹 상태로 시작해요(열람 = 명시적·기록되는 액션).
- **목록형 화면** (사용자·파일·게시물·결제·감사로그·앱 상세의 사용자 탭)은 `GridPage` + `AdminDataGrid` + `useAdminList` 조합이에요. PC 는 그리드 위 인라인 필터 바 + 하단 페이지네이션, 모바일은 "필터" 버튼(풀스크린 모달) + 카드 목록 + 무한스크롤로 자동 전환돼요. 파일의 이미지·영상 탭은 썸네일 미디어 그리드(`MediaGrid`, 무한스크롤), 오디오 탭은 커스텀 `AudioList`(데이터는 `useAdminList`)로 예외 처리해요.
- **참조형 표** (서비스 현황의 앱 목록)는 서버 페이지네이션이 필요 없는 `AdminMiniGrid`를 써요 — 모바일에선 카드 목록으로만 전환되고 무한스크롤은 없어요. (대시보드는 도넛+범례(`MiniPieChart`), 파일 목록은 서버 페이지네이션의 `AdminDataGrid` 를 써요.)
- **모더레이션 UX** (파일·게시물): 숨김(검역)=확인 모달, 삭제=**사유 필수 모달** + soft-delete(30일 후 purge, 그 전 복원 가능 — `purgeAt` D-day 표시), 복원=즉시. 상태 태그 + 행 틴트로 상태를 구분해요.
- 새 화면을 추가하는 방법은 [`derived-repo.md`](./derived-repo.md)의 "화면 추가 패턴" 참고.
