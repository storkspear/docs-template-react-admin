# 화면 가이드

`nav.tsx`(`NAV_ITEMS`)의 **사이드바 메뉴 화면 11개** + 헤더 계정 드롭다운으로 진입하는 설정(`/settings`) + 라벨 없이 라우트만 등록된 화면 4개(앱 상세 `/apps/:slug` · 게시물 작성 `/content/new` · 게시물 상세 `/content/:id` · 전체 이벤트 `/analytics/events`)로 구성돼요. 각 화면이 무엇을 보여주는지, 어떤 API 를 쓰는지, 모바일에서 어떻게 달라지는지를 정리했어요.

사이드바 메뉴는 세 **그룹**으로 나뉘어요(`nav.tsx`의 `group`/`scope` — § "공통 패턴" 참고):

| 그룹 | 화면 | 노출 조건 |
|---|---|---|
| **공통** | 대시보드 · 매출 분석 · 역할·권한 | 항상 (`scope: 'global'`) |
| **앱별** | 사용자 · 파일 · 게시물 · 이벤트 분석 · 결제 · 감사로그 · 발송 | 헤더에서 **앱을 선택했을 때만** (`scope: 'app'`) |
| **템플릿** | 컴포넌트 | 항상 (템플릿 데모/도구) |

> 컬럼 정의·필터 폼 등 실제 코드는 각 화면 이름으로 `src/pages/`에서 바로 찾을 수 있어요 (예: 대시보드 → `src/pages/DashboardPage.tsx`).

**정체성 원칙**: 대시보드는 **공장 전체 뷰**(비교·순위·합산 추이·치명 신호)만 보여줘요 — 개별 앱을 깊이 파고드는 화면이 아니에요. 앱 하나의 지표·운영 신호·시계열·빌링·사용자를 전부 보고 싶으면 앱 상세(`/apps/:slug`)로 가세요 — 개별 앱의 **유일한 집**이에요. (예전에 있던 최상위 "분석" 메뉴는 이 중복 때문에 삭제하고, 그 안에 있던 운영 신호·시계열 차트 블록은 앱 상세 개요탭으로 이전했어요. 지금의 "이벤트 분석" 메뉴는 그것과 다른, 제품 이벤트 전용 화면이에요 — §9.)

---

## 공통 그룹

### 1. 대시보드 (`/`)

전사(fleet) KPI 카드 7개(총 사용자·신규 30일·DAU·MAU·매출 30일·환불 30일·실패 24시간), 앱별 매출 점유(도넛 + 범례), 전체 매출·가입 추이 차트, 운영 신호 요약, 최근 실패 5건, 고객 결제 TOP 5를 3행으로 보여줘요.

카드 높이 불균일·이동 방식 비일관 문제를 없애려고 공통 래퍼 두 개로 전부 통일했어요:
- **`StatCard`**(`src/components/StatCard.tsx`) — Row1 스탯 카드 7개 전용. `onClick` 있으면 카드 전체가 클릭 대상(커서 포인터 + hover 시 그림자), 없으면 순수 정보 카드. 카드별 `extra` 링크(예전 매출 카드의 "내역 →") 같은 예외를 두지 않아 7카드 높이가 항상 동일해요.
- **`DashSectionCard`**(`src/components/DashSectionCard.tsx`) — Row2/Row3 섹션 카드 전용. `title` + `to?`(있으면 우상단에 화살표 아이콘 링크, 전 카드 동일 위치) + `bodyHeight`(행별 고정 — Row2 240px, Row3 260px) + `onClick?`(본문 전체 클릭 — 추이 카드용)로 구성돼서 "카드마다 제각각"이 구조적으로 불가능해요.

- **API**:
  - `GET /dashboard/metrics?window=30d` — Row1 스탯 7개 + Row2 매출 점유 도넛(`perSlug`) + Row3 운영 신호(`totals.renewalFailures7d`/`webhookPending`/`webhookFailed`)
  - `GET /analytics/revenue?slug=`, `GET /analytics/signups?slug=` — **슬러그마다 병렬 호출**(`useStackedTrend`, `useQueries`)해 상위 5개 앱은 각자 세그먼트·나머지는 '기타'로 합산한 적층 막대(`StackedBarChart`) → "전체 매출 추이"/"전체 가입 추이" 차트
  - `GET /audit-logs?result=FAILURE&size=5` — "최근 실패 5"
  - `GET /dashboard/top-customers?window=30d&size=5` — "고객 결제 TOP 5"
- **인터랙션**(이동 목적지 있는 카드는 전부 클릭형, extra 링크 텍스트는 없음):
  - Row1: 총 사용자 카드 클릭 → `/users`, 매출 카드 클릭 → `/payments`, 환불 카드 클릭 → `/payments?status=REFUNDED`, 실패 카드 클릭 → `/audit-logs?result=FAILURE`. 신규/DAU/MAU 는 클릭 없음(정보 전용).
  - Row2: "앱별 매출 점유" 우상단 화살표 → `/apps`, 도넛 세그먼트/범례 행 클릭 → 앱 상세(`/apps/{slug}`). "전체 매출 추이"/"전체 가입 추이"는 우상단 화살표·본문 전체 클릭 모두 `/payments`·`/users`로 바로 가지 않고 **앱별 비교 Drawer**(`TrendDrawer`, `src/components/TrendDrawer.tsx`)를 먼저 열어요 — 앱마다 시계열 차트 + "결제 보기"/"사용자 보기" 링크(`/payments?slug={slug}`·`/users?slug={slug}`)로 드릴다운해요.
  - Row3: "최근 실패 5" 우상단 화살표 및 행 클릭 → `/audit-logs?result=FAILURE`(감사로그 화면이 이 쿼리를 초기 필터로 반영). "고객 결제 TOP 5" 우상단 화살표 및 행 클릭 → `/users`(앱 필터 연동은 v2). "운영 신호"는 이동 목적지 없음(정보 전용).
- **차트**: `MiniPieChart`(`src/components/MiniPieChart.tsx`)는 의존성 0 SVG 도넛(stroke-dasharray 세그먼트) — 중앙에 총액 표시, hover 시 세그먼트 강조 + 툴팁(`<title>`), 세그먼트/범례 클릭으로 `onSegmentClick(label)` 콜백.
- **모바일**: Row1~3 전부 `Col xs={24}`로 세로 스택.

### 2. 매출 분석 (`/apps`)

등록된 앱(슬러그) 목록을 사용자 수·활성 구독 수와 함께 보여주는, 앱별 매출·지표 드릴다운의 진입점이에요. 행을 클릭하면 앱 상세로 이동해요. (사이드바 라벨은 "매출 분석" — 개별 앱의 매출·빌링을 파고드는 흐름의 시작점이라서요.)

- **API**: `GET /apps`
- **모바일**: 카드 목록으로 전환. 앱 개수가 적어 페이지네이션 없음(`AdminMiniGrid`).

### 3. 앱 상세 (`/apps/:slug`) — 메뉴 밖 라우트

개별 앱의 **유일한 집** — 개요탭에 지표 카드·빌링 요약·운영 신호(갱신 실패율·웹훅 미처리·리텐션 D1/D7)·시계열 차트(DAU·신규가입·매출)까지 한 화면에서 다 봐요. 사용자탭은 그 앱 사용자 목록.

- **API**(개요탭): `GET /apps/{slug}/metrics`, `GET /apps/{slug}/billing`, `GET /apps/{slug}/ops`, `GET /analytics/{metric}?slug=`(`metric`=`dau`/`signups`/`revenue`, 3번 호출)
- **컴포넌트**: 운영 신호 4카드는 `src/components/OpsSignalCards.tsx`(props: `slug`), 시계열 3차트는 `src/components/AnalyticsCharts.tsx`(props: `slug`) — 둘 다 재사용 가능한 컴포넌트로 추출돼 있어요(구 `AnalyticsPage`의 블록).
- **API**(사용자탭): `GET /apps/{slug}/users?query=&page=&size=`, `GET /apps/{slug}/users/{userId}`(드로어)
- **모바일**: 개요탭 카드들이 세로 스택. 사용자탭은 무한스크롤 + 카드 목록(아래 "6. 사용자"와 동일 패턴).
- `nav.tsx`에 `label` 없이 등록 — 사이드바 메뉴엔 안 뜨고 라우트만 잡혀요.

### 4. 역할·권한 (`/roles`)

RBAC 운영 화면 — **(1) 역할×권한 매트릭스 편집**과 **(2) 관리자 계정 관리** 두 블록이에요. 전부 실 API 에 배선돼 있고, `PERM_ADMIN_MANAGE`(기본 grant 로는 master 전용)가 필요해요. 저장·계층·의존 검증은 백엔드가 재검증하고, 화면은 UX(토글·의존 자동반영·잠금 표시)만 담당해요.

- **매트릭스**: 메뉴(1-depth)/세부 권한(2-depth) 트리 표 × 역할(viewer/support/admin/master) 열. 편집 가능한 칸은 토글(Switch), 고정 권한(대시보드·설정 등 `FIXED`)과 상급·본인 티어 칸은 잠김("권한있음/권한없음" 흐림 표시), master 열은 항상 전체 권한(열 병합). `UNMASK`/`WRITE` 토글 시 짝이 되는 `READ`가 자동으로 켜져요(백엔드 의존 규칙과 동일). "권한 저장"은 목표 집합을 통째로 저장하고, 대상 계정은 다음 로그인부터 반영돼요.
- **계정 관리**: 계정 목록(이메일·표시이름·역할 태그) + "관리자 추가" 모달(이메일·비밀번호·역할 — 자기보다 낮은 티어만) + 역할 변경 셀렉트 + **정보변경 모달(이름 + 새 비밀번호 — 비밀번호는 입력했을 때만 재설정)** + 삭제(확인 모달). 본인·마지막 master 는 삭제/강등이 서버에서 거부돼요. 두 표 모두 Card(흰 배경) 안에 `bordered` 로 렌더하고, 매트릭스 1-depth 메뉴 행은 전부 굵게 표시해요.
- **API**: `GET/PUT /roles/permissions`, `GET/POST /admins`, `PATCH/DELETE /admins/{id}`, `POST /admins/{id}/password` (계약·에러 코드는 [`admin-api.md`](../api-contract/admin-api.md) §31~§38)
- **모바일**: 별도 카드 전환 없이 `antd Table`이 가로 스크롤(`scroll={{ x: 'max-content' }}`)로 대응.

### 5. 설정 (`/settings`) — 헤더 계정 드롭다운 진입

계정 정보(이메일 표시), 본인 비밀번호 변경, **영수증 관리** 세 카드로 이뤄진 "폼 페이지" 표준 레퍼런스예요. **사이드바 메뉴가 아니라 헤더 우상단 계정 드롭다운의 [내 정보]로 들어가요**(`nav.tsx` 에서 `label` 은 두고 `group` 만 빼면 사이드바엔 안 뜨면서 헤더 섹션명·활성 판정은 정상 동작해요 — 라벨까지 빼면 대시보드로 잘못 잡혀요). **전 항목이 실제로 동작해요** — 저장되지 않으면서 저장된 척하던 스켈레톤(표시 이름·표 밀도·알림 토글·하단 가짜 [저장] 버튼)은 전부 제거했고, 테마 전환은 **헤더 우상단 전구 아이콘**(`AppLayout`)이 담당하므로 설정에 중복 노출하지 않아요. 카드마다 자체 저장 버튼을 가져요.

- **API**: `POST /me/password` — 비밀번호 변경(`changeMyPassword`, 현재 비밀번호 불일치면 `ADMIN_016` 메시지 표시). 계정 카드의 이메일은 읽기 전용 표시. 테마 토글은 `useThemeMode`(localStorage 지속) — 위치는 헤더.
- **파생 레포 참고**: 설정 항목을 추가할 땐 **저장 동작이 실제로 붙는 항목만** 노출하세요(Card + Form + 카드별 저장 버튼 패턴).
- **영수증 관리 카드**: 이메일·사업자등록번호·사업장 주소·사업장명·핸드폰번호를 입력·저장해요(`GET/PUT /settings/receipt` — mock 전용, admin-api.md §18d). 이 정보는 결제 영수증 발송 시 발행처 표기에 사용돼요(미리보기·이메일 동일 적용).
- **모바일**: 최대폭 컬럼이 화면 폭에 맞춰 줄어들 뿐 별도 분기 없음.

---

## 앱별 그룹 (헤더에서 앱 선택 시 노출)

### 6. 사용자 (`/users`)

상단 앱 컨텍스트로 선택된 앱의 사용자 목록을 검색·조회해요. 앱 상세의 사용자 탭과 같은 컬럼·드로어를 최상위 메뉴로 뺀 화면이에요(둘 다 `userColumns` 공유).

- **딥링크**: `?slug=` 쿼리 파라미터로 진입하면(대시보드 "전체 가입 추이" 드릴다운(TrendDrawer)의 앱별 "사용자 보기" 등) `useSearchParams`로 마운트 시 1회 읽어 앱 컨텍스트에 반영해요(감사로그의 `?result=FAILURE` 패턴과 동일).
- **PII 마스킹**: `PERM_USERS_UNMASK` 없는 세션(기본 grant 기준 support)은 이메일·닉네임 등이 마스킹돼 보여요. 상세 드로어의 "조회" 버튼으로 단건 원본을 열람하면 서버가 `user_read_history`에 기록해요 — § "공통 패턴" 참고.
- **API**: `GET /apps/{slug}/users?query=&page=&size=`, `GET /apps/{slug}/users/{userId}` (드로어), `GET /apps/{slug}/users/{userId}/reveal` (원본 조회)
- **모바일**: PC 는 필터가 그리드 위 인라인 바, 모바일은 제목 옆 "필터" 버튼 → 풀스크린 모달. 목록은 무한스크롤 + 카드 목록으로 전환(`useAdminList`가 PC=페이지네이션/모바일=무한스크롤 자동 분기).

### 7. 파일 (`/files`)

선택된 앱 스토리지의 업로드 파일을 조회·모더레이션(검역/삭제/복원)하는 화면이에요. **서버 페이지네이션**(`PageResponse` — 사용자·결제와 동일 계약)이라 `useAdminList`를 그대로 써요.

- **타입 탭**(`ViewTabs`): **전체 / 이미지 / 영상 / 오디오 / 삭제 대상** — 이미지·영상·오디오는 서버 `kind` 필터, "삭제 대상"은 soft-delete 된 파일(`status=deleted`)만 따로 봐요.
  - **전체·오디오·삭제 대상**: `AdminDataGrid` — 데스크톱 번호 페이지네이션 / 모바일 무한스크롤(`useAdminList` 표준). 오디오는 행에서 바로 재생(`AudioList`).
  - **이미지·영상**: 썸네일 **미디어 그리드**(`MediaGrid`, `useInfiniteQuery` 무한스크롤, 한 번에 24개) — 그리드/리스트 뷰 토글이 따로 있어요.
- **필터**: prefix · 파일명(원본 파일명/key 부분일치) · 업로더 · 상태(정상/검역) · 기간(생성일/수정일 선택 + 범위). 정렬 셀렉트(최신순/오래된순/용량순 — 현 페이지 한정) + 북마크(즐겨찾기)만 보기 토글.
- **미리보기·라이트박스**: 이미지·영상 셀/썸네일 클릭 → 반응형 **`MediaModal`**(라이트박스)로 크게 봐요(새 탭 아님). `url`은 presigned GET(~10분)이라 만료되면 재검색 안내가 떠요.
- **상세 드로어**(`FileDetailDrawer`): 파일 메타(원본 파일명·MIME·크기·연관 대상(게시물/사용자)·업로드/수정 시각) + 업로더·IP·기기 정보. 업로더·IP·기기는 `PERM_FILES_UNMASK` 없으면 마스킹 — "업로더·IP·기기 조회" 버튼으로 단건 원본 열람(`GET .../files/{key}/reveal`, 열람 기록됨).
- **액션**(`PERM_FILES_WRITE` — `canWrite('/files')` 게이팅):
  - **검역**: 확인 모달("사용자에게 안 보이게 숨겨요. 언제든 복원할 수 있어요.") → `POST .../quarantine`. 검역된 파일은 "복원" 버튼으로 원상 복구.
  - **삭제**: **사유 선택 모달(필수)** → `DELETE .../files?key=` — 즉시 영구삭제가 아니라 **soft-delete**예요. 사유는 콤보박스 프리셋(모더레이션 5종: 음란물·선정적 / 폭력적·혐오 / 개인정보 노출 / 저작권·초상권 침해 / 스팸·광고·사기 + 운영 3종: 사용자 삭제 요청 / 중복·잘못 업로드 / 테스트·임시 파일 정리)에서 고르고, "직접 입력…"을 선택했을 때만 자유 입력창이 열려요(`useReasonModal` 의 `presets` 옵션 — 사유 문구가 통일돼 감사로그 집계가 가능해져요). "삭제 대상으로 옮겼어요 — 30일 후 영구삭제돼요." 토스트가 뜨고, "삭제 대상" 탭에 삭제일·사유·**영구삭제 D-day**(`purgeAt`) 컬럼과 함께 나타나요.
  - **삭제 대상 복원**: `POST .../restore-deleted` — purge 전이면 언제든 정상으로 되돌려요.
- **모바일**: 필터는 다른 그리드 화면과 동일하게 "필터" 버튼 → 풀스크린 모달. 목록/그리드는 화면 폭에 맞춰 전환.

### 8. 게시물 (`/content`) + 게시물 상세 (`/content/:id`)

선택된 앱의 **공유(공개) 게시물**을 전량 조회하고 숨김/삭제/복원하는 콘텐츠 모더레이션 화면이에요. 프라이빗 기록은 이 화면에 안 와요(공개 전용). 파일 화면과 동일한 모더레이션 UX(상태 태그·행 틴트·사유 모달·복원 컨펌)를 써요.

- **목록**(`ContentPage`): 게시판(board)·상태(공개/숨김/삭제) 필터 + `GridPage`+`AdminDataGrid`+`useAdminList` 표준 목록. 행 클릭 → 게시물 상세로 이동.
- **상세**(`ContentDetailPage`, 메뉴 밖 라우트): **아티클 리더 레이아웃** — 모더레이터가 게시물을 실제 읽는 형태로 제목·본문 전체(읽기 시간 표시)·첨부이미지(파일 콘솔과 동일 원본, presigned URL)를 보고, 그 자리에서 숨김(사유)/삭제(사유)/복원/삭제 대상 복원을 실행해요.
- **작성**(`ContentComposePage`, `/content/new`): 흰 종이 캔버스 에디터(Tiptap). **상단 sticky 바가 `[목록] · [카테고리 선택]` (좌) / `[폭 토글] [등록]` (우)** 구조예요 — 카테고리(board)는 글의 내용이 아니라 발행 설정이라 본문 캔버스에서 분리해 이 바에 뒀어요(`UnderlineSelect`, 고정 4종 + "카테고리 선택"으로 해제). 본문 캔버스는 제목 → 해시태그 → 에디터 순서로 글 자체에만 집중해요.
- **본문 폭**(`usePaperWidth`): 작성 화면의 폭 토글(기본 760px ↔ 넓게)에서 정한 폭이 **글 속성 `properties.wide` 로 저장**되고, 상세(읽기)가 그 값으로 렌더해요 — **넓게 쓴 글은 넓게, 좁게 쓴 글은 좁게 읽혀요**. 상세엔 폭 토글을 두지 않아요(읽는 사람이 바꿀 값이 아니라 글의 속성이라서요). localStorage(`admin.composeWide`)는 "다음 새 글의 기본 폭"으로만 쓰여요.
- **soft-delete**: 파일과 동일 — 삭제는 사유 필수 + 30일 후 purge, 그 전엔 복원 가능(`purgeAt` D-day).
- **API**: `GET /apps/{slug}/content?board=&status=&page=&size=`, `GET .../content/{id}`, `POST .../content/{id}/hide|restore|restore-deleted`, `DELETE .../content/{id}` (계약은 [`admin-api.md`](../api-contract/admin-api.md) §25~§30)
- **권한**: 조회 `PERM_CONTENT_READ`, 액션 `PERM_CONTENT_WRITE`(`canWrite('/content')`). 공개 게시물이라 작성자 마스킹은 없어요.
- **모바일**: 목록은 무한스크롤 + 카드 목록, 상세는 세로 스택.

### 9. 이벤트 분석 (`/analytics`) + 전체 이벤트 (`/analytics/events`)

선택된 앱의 **제품 이벤트**(백엔드 `@TrackEvent` 자동 계측 → `analytics_daily` 야간 롤업) 요약·추이 화면이에요. 이벤트는 행동+메타데이터만 있고 콘텐츠 내용은 없어요(개발 방침).

- **메인**(`AnalyticsPage`): 기간 토글(7일/30일/90일) + 상단에 선택 이벤트의 일별 추이 차트(풀폭, `MiniBarChart`) + 발생수 **상위 5** 이벤트 표(발생수·비중·순사용자). "전체 이벤트 보기" 버튼으로 하위 페이지 진입.
- **전체 이벤트**(`AnalyticsEventsPage`, 메뉴 밖 라우트): 상위 5에 안 드는 사소한 이벤트까지 전량 표시. 행 클릭 시 상단 추이 차트가 그 이벤트로 갱신돼요.
- **API**: `GET /analytics/events?slug=&from=&to=`(요약), `GET /analytics/events/{eventName}?slug=`(단일 이벤트 추이)
- **모바일**: 차트·표가 세로 스택.

### 10. 결제 (`/payments`)

선택된 앱의 결제 내역을 이메일·채널·상태·유형·기간으로 필터링해서 봐요. "누가·언제·얼마"를 드릴다운으로 확인하는 화면이에요. 대시보드의 "매출 (30일)" 카드 클릭으로 `/payments`, "환불 (30일)" 카드 클릭으로 `/payments?status=REFUNDED`(환불 건만 미리 필터링된 상태)로, "전체 매출 추이" 드릴다운(TrendDrawer)의 앱별 "결제 보기" 클릭으로 `/payments?slug={slug}`(해당 앱 미리 선택된 상태)로 진입할 수 있어요.

- **딥링크**: `?status=REFUNDED`·`?slug=` 쿼리 파라미터로 진입하면(대시보드 "환불 (30일)" 카드·TrendDrawer 등) `useSearchParams`로 마운트 시 1회 읽어 상태 필터·앱 컨텍스트에 반영해요(감사로그의 `?result=FAILURE` 패턴과 동일).
- **API**: `GET /apps/{slug}/payments?query=&channel=&status=&type=&from=&to=&page=&size=`, `POST /apps/{slug}/payments/{paymentId}/refund` (환불), `GET /apps/{slug}/payments/{paymentId}/refunds` (환불 이력), `GET .../receipt` + `POST .../receipt-email` (영수증 조회·발송 — mock 전용, admin-api.md §18b~§18c)
- **컬럼**: 일련번호·상태(Tag — 한글 라벨 `lib/paymentStatus`: 결제완료 green/부분환불 gold/환불완료 gray/실패 red)·채널(Tag)·유형(Tag — 구독 purple/일반 default)·이메일(복사 버튼)·금액(우측정렬 — 부분/전액 환불 시 아래 `환불 −₩…` 누적액)·결제일시·환불일시(`null`이면 `—`)·액션(PG·`PAID`/`PARTIALLY_REFUNDED` 행만 "환불" 버튼, IAP 는 툴팁 + `—`). 행 틴트: 부분환불 amber·전액환불 옅은 회색.
- **결제 상세·환불 이력 팝업**(읽기 전용): 금액/상태 셀 클릭으로 진입. 영수증 스타일(결제 금액 − 환불됨 = 환불 가능 잔액, 잔액만 강조) + 일시 + 환불 이력 원장(건별 금액·사유·시각·처리자). 환불 가능(PG·PAID/부분환불)하고 쓰기 권한이 있으면 [환불 진행] 버튼이 환불 모달로 연결.
- **환불 모달**(중앙): 컨텍스트 서브라인(이메일·유형·채널) + 환불 가능 잔액 히어로 + 금액 입력([전액]·[잔여기간 비례 ₩N] 프리셋 칩 — 일할계산식은 칩 툴팁, 계산 불가 사유(기간 만료 등)는 disabled 칩 툴팁)·상태 예고 캡션·**사유(프리셋 콤보 — 단순 변심/구독 해지/중복 결제/결제 오류/불만족/직권 취소, "직접 입력…" 선택 시에만 TextArea)**. 잔액 다 환불 시 `REFUNDED`, 일부면 `PARTIALLY_REFUNDED`(재환불 가능). 환불 버튼은 `PERM_PAYMENTS_WRITE` 게이팅. 환불 이력은 상세 팝업이 전담("이력 보기" 링크로 이동).
- **영수증 팝업**: 환불 성공 직후 자동으로 열리고, 상세 팝업의 [영수증] 버튼(환불완료/부분환불 건)으로도 진입해요. 서버가 조립한 payload(§18b — 서비스명·영수증번호·결제수단(신용카드는 마스킹 카드번호)·거래번호·금액 원장·발행처)를 그대로 렌더하고, [이미지로 저장]으로 같은 데이터를 JPEG(캔버스 렌더)로 내려받을 수 있고, [이메일로 발송]은 **같은 payload** 로 만든 영수증을 결제자에게 보내요(§18c) — 미리보기와 발송본이 항상 일치. 발행처 정보는 설정 > 영수증 관리에서 입력해요.
- **모바일**: PC 는 필터가 그리드 위 인라인 바, 모바일은 제목 옆 "필터" 버튼 → 풀스크린 모달. 목록은 무한스크롤 + 카드 목록으로 전환(`useAdminList`).

### 11. 감사로그 (`/audit-logs`)

관리자·시스템 액션 로그를 slug·actor 이메일·action·result·기간으로 필터링해서 봐요.

- **API**: `GET /audit-logs?slug=&actorEmail=&action=&result=&from=&to=&page=&size=`
- **딥링크**: `?result=FAILURE` 쿼리 파라미터로 진입하면(대시보드 "최근 실패 5" 등) `useSearchParams`로 마운트 시 1회 읽어 결과 필터 폼 초기값 + 필터 상태에 반영해요.
- **모바일**: 제목 옆 "필터" 버튼(적용 개수 배지) → 풀스크린 모달(RangePicker 포함, 모달 내부에 팝업 렌더). 목록은 무한스크롤 + 카드 목록.

### 12. 발송 (`/send`)

선택된 앱 사용자에게 SMS/이메일/푸시를 보내는 화면이에요. "보내기"와 "발송 이력" 두 뷰(`ViewTabs`)로 구성돼요.

> ⚠ **실 API 배선 + mock 전용**: 이 화면은 발송 6개 엔드포인트(세그먼트·미리보기·수신자 목록·발송·이력·전달 상태)에 **실제로 배선돼** 있지만, `template-spring`엔 아직 대응 컨트롤러가 없어요 — **mock(MSW) 모드에서만 동작**하고 실서버 모드에선 404가 나요. 계약 상세는 [`admin-api.md`](../api-contract/admin-api.md) §M1~§M6 참고.

- **보내기(compose)**: 채널(SMS/이메일/푸시) 토글 + 대상(전체/프리미엄/무료/활성/신규/특정 사용자/토픽 — 토픽은 푸시 전용) + 제목/내용. 채널×대상의 **유효 수신자 수 미리보기**(`GET .../messages/preview`)와 **실제 수신자 목록**(`GET .../messages/recipients`)을 발송 전에 확인하고, 확인 모달을 거쳐 발송해요. 실기기 모양의 미리보기(`DevicePreview`), SMS 는 byte 카운터(90byte 초과 시 LMS 표시), 채널 미지원 앱(예: SMS 는 phone-auth 앱만)은 안내 Alert 로 차단돼요.
- **발송 이력(history)**: 채널·기간 필터 + 발송 건 목록(발신자·대상·건수·결과). 건수를 클릭하면 수신자별 전달 상태(DELIVERED/FAILED/PENDING)를 봐요.
- **API**: `GET .../messages/segments`, `GET .../messages/preview`, `GET .../messages/recipients`, `POST .../messages`, `GET .../messages`(이력), `GET .../messages/{id}/recipients`
- **권한**: `PERM_SEND`.
- **모바일**: 폼이 세로 스택으로 쌓이고, 이력 목록은 카드 전환.

---

## 템플릿 그룹

### 13. 컴포넌트 (`/components`)

이 템플릿이 제공하는 색·타이포·버튼·폼 입력·탭·태그·피드백·Statistic/차트를 한 화면에 모은 디자인 시스템 갤러리예요. 파생 레포가 재사용 가능한 UI 조각을 한눈에 확인하는 용도.

- **API**: 없음.
- **모바일**: 섹션이 세로로 쌓이고 `Space wrap`으로 요소가 줄바꿈돼요.

---

## 메뉴 밖 라우트

| 라우트 | 화면 | 비고 |
|---|---|---|
| `/login` | `LoginPage` | `POST /auth/login`. Mock 모드에서만 데모 자격증명이 채워짐(`master@example.com` / `password`). mock 은 이메일 **프리픽스**로 역할을 결정해요 — `viewer@`/`support@`/`admin@`/`master@`(그 외 전부 master)로 로그인하면 티어별 화면 차이를 바로 시연할 수 있어요 |
| `/apps/:slug` | `AppDetailPage` | 위 "3. 앱 상세" 참고 |
| `/content/:id` | `ContentDetailPage` | 위 "8. 게시물" 참고 |
| `/content/new` | `ContentComposePage` | 위 "8. 게시물" 작성 항목 참고 — 목록 우상단 [작성하기]로 진입 |
| `/analytics/events` | `AnalyticsEventsPage` | 위 "9. 이벤트 분석" 참고 |

`/login`만 `App.tsx`에 별도 `<Route>`로 등록되고, 나머지는 전부 `nav.tsx`에 `label` 없이 등록돼요 — 사이드바 메뉴엔 안 뜨고 라우트만 잡혀요.

## 공통 패턴

### 디자인 시스템 (2026-07 리프레시)
- **무드(스킨)**: `src/lib/moods.ts` 의 4종(Obsidian=블랙 미니멀 / Blurple=Stripe 풍 / Ember=슬레이트+오렌지 / Glacier=Linear 풍 고밀도) × 라이트/다크. 헤더 팔레트 아이콘으로 전환, localStorage 지속. antd 테마·AG Grid·사이드바·뱃지 문법이 전부 무드에서 파생돼요.
- **상태 뱃지**: `StatusBadge` + 의미 키(success/warning/danger/info/neutral/brand). antd `<Tag color="green">` 는 쓰지 않아요(예외: closable 칩·종이 캔버스 전용 칩).
- **아이콘**: lucide(1.75px 스트로크)를 `src/lib/icons.tsx` 가 antd 이름으로 재-export — 사용처는 이름 그대로, import 경로만 이 모듈.
- **필터 바**: CSS Grid(`repeat(auto-fit, minmax(200px,1fr))`) + 라벨 상단 — 브라우저 폭과 무관하게 정연한 격자, 기간(RangePicker)은 2칸.



- **앱 컨텍스트**: 헤더의 앱 스위처로 앱을 선택하면 `scope: 'app'` 메뉴(앱별 그룹)가 사이드바에 나타나고, 해당 화면들은 전부 선택된 앱 기준으로 조회해요. 앱 미선택 시엔 "상단에서 앱을 선택하세요" Empty 상태를 보여줘요.
- **RBAC 게이팅**(`src/lib/rbac.ts`): 로그인 응답의 `permissions`(효과 권한 `PERM_*`)로 — ① 사이드바 메뉴를 `canReadKey`로 거르고(권한 없는 메뉴는 아예 안 보임), ② 화면 안의 쓰기 액션(환불·검역·삭제·발송 등)을 `canWrite`로 숨겨요. 이건 UX 게이팅일 뿐이고 최종 강제는 백엔드(`hasAuthority`)예요.
- **PII 마스킹 + "조회"(reveal)**: 사용자(이메일·닉네임 등)와 파일(업로더·IP·기기)은 `PERM_*_UNMASK` 없는 세션에 마스킹돼 내려와요. 상세 드로어의 "조회" 버튼으로 **단건 원본**을 열람하면 서버가 `user_read_history`에 열람 기록을 남기고, 드로어를 다시 열면 다시 마스킹 상태로 시작해요(열람 = 명시적·기록되는 액션).
- **목록형 화면** (사용자·파일·게시물·결제·감사로그·앱 상세의 사용자 탭)은 `GridPage` + `AdminDataGrid` + `useAdminList` 조합이에요. PC 는 그리드 위 인라인 필터 바 + 하단 페이지네이션, 모바일은 "필터" 버튼(풀스크린 모달) + 카드 목록 + 무한스크롤로 자동 전환돼요. 파일의 이미지·영상 탭만 예외로 썸네일 미디어 그리드(무한스크롤)를 써요.
- **참조형 표** (매출 분석의 앱 목록)는 서버 페이지네이션이 필요 없는 `AdminMiniGrid`를 써요 — 모바일에선 카드 목록으로만 전환되고 무한스크롤은 없어요. (대시보드는 IA 재편으로 이 패턴을 더 이상 안 써요 — 앱별 지표 미니그리드를 삭제하고 도넛+범례(`MiniPieChart`)로 대체했어요. 파일 목록도 서버 페이지네이션 도입으로 `AdminDataGrid`로 옮겨갔어요.)
- **모더레이션 UX** (파일·게시물): 숨김(검역)=확인 모달, 삭제=**사유 필수 모달** + soft-delete(30일 후 purge, 그 전 복원 가능 — `purgeAt` D-day 표시), 복원=즉시. 상태 태그 + 행 틴트로 상태를 구분해요.
- 새 화면을 추가하는 방법은 [`derived-repo.md`](./derived-repo.md)의 "화면 추가 패턴" 참고.
