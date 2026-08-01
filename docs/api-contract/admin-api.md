# Admin API — 엔드포인트 계약

`template-spring`의 `core/core-admin-impl` 모듈이 제공하는 `/api/admin/*` 전체 계약이에요. 프론트 타입은 `src/lib/types.ts`, 클라이언트 함수는 `src/api/client.ts`, mock 구현은 `src/mocks/handlers.ts` + `fixtures.ts`에 있어요.

**범위(실측)**: 백엔드 매핑 **49개**(§1~§49 — 데이터 47 + `login` + `health`) 전부와, 백엔드 구현 전이라 **mock 전용**인 발송(메시징) 6개(§M1~§M6)·영수증 4개(§18b~§18d — 설정 GET/PUT 포함)를 다뤄요. `src/api/client.ts`의 export 함수는 60개(= 백엔드 대응 48 + mock 전용 발송 6 + mock 전용 영수증 4 + 헬퍼 2(`isServerDown`·`uploadFileToUrl` — presigned PUT 직행이라 admin 엔드포인트 함수가 아니에요) — `health`는 인프라 프로브라 클라이언트 함수가 없어요)예요. 섹션 번호 §1~§49는 `template-spring`의 [`docs/api-and-functional/admin-console.md`](https://github.com/storkspear/template-spring/blob/main/docs/api-and-functional/admin-console.md) §3 엔드포인트 카탈로그와 같은 번호를 써요 — 두 레포 문서를 나란히 놓고 대조할 수 있게요. 개수는 시간이 지나면 어긋나기 쉬우니, 의심되면 `client.ts`와 spring 카탈로그를 먼저 보세요.

> **인증 스코프 — RBAC 4티어**: admin 로그인은 앱 유저 인증과 완전히 분리돼요. 백엔드는 별도 `admin.admin_users` 스키마 계정으로 콘솔 JWT 를 발급하는데, `role` claim 은 **RBAC 4티어 코드**(`viewer` < `support` < `admin` < `master` — 누적)이고, 역할에서 계산된 효과 권한(`PERM_*` 목록)이 **`permissions` claim** 으로 실려요. 백엔드 `SecurityConfig`가 `/api/admin/**` 리소스별로 `hasAuthority(PERM_*)`를 검사해요(예: 환불 POST 는 `PERM_PAYMENTS_WRITE`). 앱 유저 JWT 는 `permissions` claim 이 없어 `/api/admin/**`에서 403, 반대로 콘솔 JWT(`appSlug="admin"`)로 `/api/apps/{slug}/**`에 접근하면 `AppSlugVerificationFilter`가 403 — 양방향 격리예요. 프론트는 로그인 응답의 `admin.permissions`로 메뉴/버튼을 게이팅하지만(`src/lib/rbac.ts`), 최종 강제는 항상 백엔드예요.

기본 grant(백엔드 V003 seed 기준, 매트릭스 편집으로 조정 가능 — §37~§38):

| 티어 | 기본 권한 |
|---|---|
| `viewer` (1) | 앱·분석 조회 |
| `support` (2) | + 사용자(마스킹)·파일(마스킹) 조회 · 발송 |
| `admin` (3) | + 사용자 원본 · 결제(조회·환불) · 감사로그 |
| `master` (4) | 전 도메인 + 계정관리(`PERM_ADMIN_MANAGE`, 코드 고정) |

---

## 공통 응답 envelope

모든 응답은 `{ data, error }`로 래핑돼요 (`ApiResponse<T>`).

```ts
interface ApiError {
  code: string
  message: string
  details: unknown | null
}
interface ApiResponse<T> {
  data: T | null
  error: ApiError | null
}
```

목록 엔드포인트(`users` · `audit-logs` · `payments` · `files` · `content`)는 `data` 안에 `PageResponse<T>`가 또 한 겹 있어요.

```ts
interface PageResponse<T> {
  content: T[]
  page: number        // 0-based
  size: number
  totalElements: number
  totalPages: number
}
```

`src/api/client.ts`의 `request()`가 이 envelope을 벗기고 `error`가 있으면 `ApiRequestError`로 던져요 — 화면 코드는 항상 벗겨진 DTO만 다뤄요.

---

## 인증 · 계정

### [1] `POST /api/admin/auth/login`

관리자 로그인. **refresh token 없음** — access-token-only예요 (앱 유저 인증과 달리 회전/재발급 흐름이 없어요. 만료되면 재로그인 — 콘솔 access token TTL 은 앱 유저의 15분과 별도로 기본 12시간이에요).

- **Request body**: `{ email: string, password: string }`
- **Response** (`AdminLogin`):
  ```ts
  type AdminRole = 'viewer' | 'support' | 'admin' | 'master'
  interface AdminAccount {
    userId: number
    email: string
    role: AdminRole
    appSlug: string
    permissions: string[]   // 로그인 시 계산된 효과 권한(PERM_*) — 프론트 게이팅의 소스
  }
  interface AdminLogin { accessToken: string; refreshToken?: string; admin: AdminAccount }
  ```
  `refreshToken`은 타입상 optional인데 실서버는 필드 자체를 안 내려줘요(`AdminLoginResponse`에 없음) — 실서버 기준 항상 `undefined`예요(mock 은 placeholder 문자열을 채우지만 쓰이지 않아요). `admin.appSlug`는 항상 `"admin"` 고정값(JWT의 `appSlug` claim이 non-blank를 요구해서 넣은 placeholder — 실제 앱 슬러그가 아니에요).
- **에러**: 자격증명 불일치 → `401 ADMIN_001`
- **Mock 로그인 규칙**: 비밀번호는 `password` 고정, 이메일은 아무거나 — 이메일 **프리픽스**가 역할을 결정해요(`viewer*`→viewer, `support*`→support, `admin*`→admin, 그 외 전부 master). 로그인 폼 데모 자격증명은 `master@example.com` / `password`. 실 백엔드는 `admin_users.role`로 결정해요.

### [2] `GET /api/admin/health`

**공개 엔드포인트** (인증 불필요). React `factory` CLI가 로컬 백엔드 연결 여부를 확인하는 용도로만 써요 — `src/api/client.ts`엔 대응 함수가 없고(데이터 계약이 아니라 인프라 프로브라서), `factory` 셸 스크립트가 직접 `curl -sf`로 호출해요.

```bash
curl -sf -m 3 -o /dev/null "$target/api/admin/health"
```

`local start`가 이 요청 성공 여부로 `VITE_USE_MOCK` 값을 자동 결정해요 (§ "Mock ↔ 실서버 토글" 참고).

---

## 앱 · 대시보드

### [3] `GET /api/admin/apps`

등록된 전체 앱(슬러그) 요약.

- **Response**: `AppSummary[]`
  ```ts
  interface AppSummary {
    slug: string
    userCount: number
    activeSubscriptions: number
    revenue30d: number            // 최근 30일 결제 매출
    iosVersion: string | null     // 스토어 배포 중인 현재 버전 — app_versions 최신 릴리스에서 유도
    androidVersion: string | null
    releasedAt: string | null     // 최초 출시일(YYYY-MM-DD) — 릴리스 이력에서 유도
    lastUpdatedAt: string | null  // 최종 업데이트일
    status: 'ok' | 'warn'         // 운영 상태 — 운영 신호에서 파생
    issueLabel: string | null     // 이슈 요약 라벨(예: '갱신실패 2'). 없으면 null
  }
  ```
- **데이터 소스**: 슬러그 열거 + 각 앱 스키마의 `users` count · `subscriptions`(ACTIVE) count · 최근 30일 `payment_history` 합산 · `app_versions` 최신 릴리스 · 운영 신호(갱신실패·웹훅)에서 `status`/`issueLabel` 파생.
- **소비**: "서비스 현황"(`/apps`) 보드가 KPI·컬럼에서 전 필드를 써요.

### [4] `GET /api/admin/dashboard/metrics`

전사(cross-app) 합산 지표.

- **Query**: `window` (기본 `30d`) — 현재 백엔드 구현은 값 자체보다 라벨 용도(윈도우별 재계산 로직은 고정 30일 집계 위주)
- **Response** (`DashboardMetrics`):
  ```ts
  interface DashboardTotals {
    users: number; newUsers: number; dau: number; mau: number
    revenue: number; refunded: number; activeSubscriptions: number; failures24h: number
    renewalFailures7d: number; webhookPending: number; webhookFailed: number
  }
  interface PerSlugMetrics { slug: string; users: number; newUsers: number; dau: number; mau: number
    revenue: number; refunded: number; activeSubscriptions: number; failures24h: number }
  interface DashboardMetrics {
    generatedAt: string; window: string
    totals: DashboardTotals; perSlug: PerSlugMetrics[]
  }
  ```
  `renewalFailures7d`/`webhookPending`/`webhookFailed`는 `ops`(§8)의 동명 필드를 **전 슬러그 fan-out 합산**한 값이에요 — 대시보드 "운영 신호" 카드가 이 3필드로 전사 신호 유무를 판단해요(전부 0이면 "정상").
- **데이터 소스**: 전 슬러그 fan-out(슬러그별 DataSource 순회) 후 메모리 합산 — users·신규(`created_at`)·DAU/MAU(§ DAU/MAU 정의)·매출/환불(`payment_history`)·활성구독·`failures24h`(`audit_logs WHERE result='FAILURE'` 24시간 이내 count)·`renewalFailures7d`/`webhookPending`/`webhookFailed`(§8과 동일 쿼리의 fleet 합).
- **에러**: 없음(전사 지표라 slug 파라미터가 없어서 `ADMIN_003` 대상이 아님).

### [5] `GET /api/admin/dashboard/top-customers`

전앱(fleet) 결제 금액 TOP N — 대시보드 "고객 결제 TOP 5" 카드가 써요.

- **Query**: `window`(기본 `30d`), `size`(기본 5)
- **Response**: `TopCustomer[]`
  ```ts
  interface TopCustomer {
    slug: string; userId: number; userEmail: string
    totalAmount: number; paymentCount: number
  }
  ```
- **데이터 소스**: 전 슬러그 fan-out 후 `payment_history`를 `(slug, userId)`로 그룹핑해 `window` 기간 내 합산·건수 집계, `totalAmount` 내림차순 `size`건. `gross`(§ gross/net 정의)와 동일하게 `PAID`/`REFUNDED`/`PARTIALLY_REFUNDED` 상태를 포함해요(환불 여부와 무관하게 한 번이라도 수금된 금액 기준).
- **에러**: 없음(전사 지표라 slug 파라미터가 없어서 `ADMIN_003` 대상이 아님).

### [6] `GET /api/admin/apps/{slug}/metrics`

앱 하나의 지표(§4와 같은 지표 계산을 단일 스키마로).

- **Response** (`AppMetrics`):
  ```ts
  interface AppMetrics {
    slug: string; generatedAt: string
    users: number; newUsers7d: number; premiumUsers: number
    dau: number; mau: number; revenue30d: number; activeSubscriptions: number
  }
  ```
- **에러**: 알 수 없는 슬러그 → `404 ADMIN_003`

### [7] `GET /api/admin/apps/{slug}/billing`

빌링 요약(최근 기간).

- **Query**: `from`, `to` (ISO-8601, 생략 시 최근 30일)
- **Response** (`BillingSummary`):
  ```ts
  interface BillingByChannel { channel: string; amount: number; count: number }
  interface BillingDaily { date: string; amount: number }
  interface BillingSummary {
    slug: string; from: string; to: string
    gross: number; refunded: number; net: number
    byChannel: BillingByChannel[]; activeSubscriptions: number; dailySeries: BillingDaily[]
  }
  ```
- **에러**: 알 수 없는 슬러그 → `404 ADMIN_003`. `from`/`to` 파싱 실패(`DateTimeParseException`) → `400 ADMIN_004`.
- **시맨틱**: § "gross/net 정의" 참고 — **환불 여부와 무관하게 한 번이라도 수금된 총합**이에요.

### [8] `GET /api/admin/apps/{slug}/ops`

운영 신호 — 구독 갱신·웹훅 처리 지연·리텐션.

- **Response** (`AppOpsSignals`):
  ```ts
  interface AppOpsSignals {
    slug: string
    renewalAttempts7d: number; renewalFailures7d: number
    webhookPending: number; webhookFailed: number
    retentionD1: number | null; retentionD7: number | null
  }
  ```
- **필드 정의**:
  - `renewalAttempts7d` / `renewalFailures7d`: `subscription_renewals`에서 `attempted_at ≥ now-7d`. 실패는 `status <> 'SUCCESS'`(`FAILED` + `ABANDONED` — 이탈 조기 신호로 둘 다 포함).
  - `webhookPending`: `payment_webhook_events`의 `processed_at IS NULL AND process_error IS NULL` (아직 처리 안 됐지만 에러도 아닌, 순수 밀림).
  - `webhookFailed`: `process_error IS NOT NULL`.
  - `retentionD1` / `retentionD7`: § "리텐션 D1/D7 코호트" 참고.
- **에러**: 알 수 없는 슬러그 → `404 ADMIN_003`.

---

## 사용자

### [9] `GET /api/admin/apps/{slug}/users`

앱 사용자 목록 검색 + 페이지네이션.

- **Query**: `query`(이메일·표시이름·닉네임 ILIKE), `page`(기본 0), `size`(기본 20, **서버가 1~100으로 clamp**). 클라이언트(`getAppUsers`)가 넘길 수 있는 `email`/`name`/`nickname` 개별 필드 검색과 `status`/`membership` 필터는 **mock 전용 확장**이에요 — 실서버 컨트롤러(`AdminUsersController`)는 `query`/`page`/`size`만 읽고 나머지는 무시해요. 사용자 화면의 상태·멤버십 셀렉트는 mock 에서만 실제로 걸러져요.
- **Response**: `PageResponse<AdminUser>`
  ```ts
  interface AdminUser {
    id: number; email: string; displayName: string | null; nickname: string | null
    role: string; isPremium: boolean; emailVerified: boolean
    createdAt: string; deletedAt: string | null
  }
  ```
- **PII 마스킹**: 세션에 `PERM_USERS_UNMASK`가 없으면(기본 grant 기준 support 티어) 이메일·닉네임 등 PII 가 마스킹돼 내려와요. 원본은 §11 "조회"로 단건 열람해요.
- **에러**: 알 수 없는 슬러그 → `404 ADMIN_003`

### [10] `GET /api/admin/apps/{slug}/users/{userId}`

사용자 상세 — 기기·구독·최근 결제 포함. §9와 같은 마스킹 규칙이 적용돼요.

- **Response** (`AdminUserDetail`):
  ```ts
  interface AdminUserFull extends AdminUser { updatedAt: string }
  interface AdminDevice { id: number; platform: string; deviceName: string | null; lastSeenAt: string | null; createdAt: string }
  interface AdminSubscription {
    id: number; planId: number; status: string // ACTIVE | CANCELLED | EXPIRED
    startedAt: string; expiresAt: string | null; cancelledAt: string | null; cancelReason: string | null
  }
  interface AdminPayment {
    id: number; channel: string // PG | IAP
    amount: number; currency: string
    status: string // READY | PAID | PARTIALLY_REFUNDED | REFUNDED | FAILED | CANCELLED | VBANK_ISSUED
    paidAt: string | null; refundedAt: string | null
  }
  interface AdminUserDetail {
    user: AdminUserFull; devices: AdminDevice[]
    subscriptions: AdminSubscription[]; recentPayments: AdminPayment[]  // 최근 10건
  }
  ```
- **에러**: 사용자 없음 → `404 ADMIN_005`. 슬러그 자체가 없으면 → `404 ADMIN_003`(슬러그 해석이 먼저 일어남).

### [11] `GET /api/admin/apps/{slug}/users/{userId}/reveal`

사용자 **원본 열람("조회")** — 마스킹 티어가 특정 사용자의 원본 PII 를 단건 확인해요. 클라이언트 함수는 `revealUser`.

- **Response**: `AdminUserDetail`(§10과 동일 shape, 단 마스킹 없이 원본)
- **감사**: 서버가 열람 사실을 `user_read_history`에 기록해요 — "누가 누구의 원본을 봤는지"가 남아요. 프론트는 드로어를 다시 열면 다시 마스킹 상태로 시작해요(열람 = 명시 액션).
- **에러**: §10과 동일(`ADMIN_005`/`ADMIN_003`).

### [42] `GET /api/admin/apps/{slug}/users/{userId}/export` — GDPR 개인정보 export (v1.11)

GDPR 열람권(Art.15) 대응 — 한 사용자의 연관 데이터를 **JSON 번들 1개**로 반환해요. **전 PII 원본**을 노출하므로 `PERM_USERS_UNMASK` 로 게이팅(READ 만으로는 403 `CMN_005`)하고, 발급 사실은 `user_read_history`에 `EXPORT`로 기록돼요. 첨부 **파일 실체는 미포함** — `attachments[].storageKey` 메타만 담고 개별 다운로드는 파일 화면(§19~§20)에서 해요. 클라이언트 함수는 `exportUser`. 프론트는 사용자 상세 드로어에서 "데이터 내보내기" 버튼(`PERM_USERS_UNMASK` 게이팅, reveal 버튼 선례)으로 호출해 응답을 `user-{id}-export.json` 파일로 내려받아요.

- **Response** (`AdminUserExportResponse`):
  ```ts
  interface AdminNotificationSetting { kind: string; pushEnabled: boolean; emailEnabled: boolean }
  interface AdminExportPost { id: number; board: string; title: string | null; status: string; createdAt: string }
  interface AdminExportAttachment {
    id: number; storageKey: string; originalFilename: string | null
    sizeBytes: number; status: string; createdAt: string
  }
  interface AdminUserExportResponse {
    exportedAt: string; slug: string
    user: AdminUserFull                        // §10
    socialProviders: string[]                  // provider 목록(예: ["google"])
    devices: AdminDevice[]; subscriptions: AdminSubscription[]
    payments: AdminPayment[]                   // 전체(상세의 최근 10건 제한 없음)
    notificationSettings: AdminNotificationSetting[]
    activityDays: string[]                     // LocalDate('YYYY-MM-DD') 목록
    posts: AdminExportPost[]                    // 메타(본문 제외)
    attachments: AdminExportAttachment[]        // 메타(오브젝트 실체 제외)
  }
  ```
- **에러**: 사용자 없음 → `404 ADMIN_005`. **이미 익명화된 사용자** → `410 ADMIN_025`. 슬러그 없음 → `404 ADMIN_003`. 권한 없음(UNMASK 부재) → `403 CMN_005`.

### [43] `DELETE /api/admin/apps/{slug}/users/{userId}` — 콘솔 탈퇴 (soft-delete, v1.11)

GDPR 삭제권(Art.17) 대응의 접수 단계 — 앱 유저 `withdraw` 와 동일 시맨틱(`deleted_at` 세팅 + 해당 유저 refresh token 전체 revoke)을 콘솔 경로로 노출해요. 쓰기 권한 `PERM_USERS_WRITE`(**신규** — 기본 grant 는 `master`만. `UNMASK` 만으로는 삭제 불가)로 게이팅해요. soft-delete 후 **30일 유예**가 지나면 `UserErasureScheduler`가 도메인별 처리표대로 완전삭제/익명화해요(auth 토큰·소셜·기기·알림설정·활동일은 hard delete, `users`·`payment_history`·`posts`·`analytics_events`는 익명화, 결제·구독·감사 원장은 법정 보존). 클라이언트 함수는 `deleteUser`. 프론트는 상세 드로어의 "계정 삭제" 버튼(`PERM_USERS_WRITE` 게이팅)으로 호출하며, 되돌릴 수 없음·30일 유예 익명화를 확인 모달로 안내해요.

- **Response**: 본문 없는 성공 — `200` + `{ data: null, error: null }`(`ApiResponse.empty()`). 클라이언트(`deleteUser`)는 `Promise<void>`.
- **에러**: **이미 탈퇴 처리** → `400 ADMIN_024`. **이미 익명화** → `410 ADMIN_025`. 사용자 없음 → `404 ADMIN_005`. 슬러그 없음 → `404 ADMIN_003`. 권한 없음(WRITE 부재) → `403 CMN_005`.

---

## 감사로그 · 분석

### [12] `GET /api/admin/audit-logs`

관리자·시스템 액션 로그.

- **Query**: `slug`(생략 가능 — 생략 시 전 슬러그 fan-out 병합), `actorEmail`, `action`, `result`(`SUCCESS`|`FAILURE`), `from`, `to`, `page`, `size`
- **Response**: `PageResponse<AuditLog>`
  ```ts
  type AuditResult = 'SUCCESS' | 'FAILURE'
  interface AuditLog {
    id: number; actorUserId: number | null; actorEmail: string | null
    action: string; resourceType: string | null; resourceId: string | null
    slug: string | null; result: AuditResult; ipAddress: string | null; occurredAt: string
  }
  ```
- **데이터 소스**: `slug` 지정 시 단일 스키마 조회. 미지정 시 전 슬러그 fan-out 후 `occurred_at` 기준 병합 정렬 + **메모리 페이징**(설계 스펙의 "알려진 한계" — 솔로 규모(앱 수 ~10, 로그 수만 건)에선 문제없지만, 커지면 slug 필터를 유도하거나 커서 방식으로 바꿔야 해요).
- **에러**: `from`/`to` 파싱 실패 → `400 ADMIN_004`.

### [13] `GET /api/admin/analytics/{metric}`

시계열 차트 데이터.

- **Path**: `metric` = `dau` | `signups` | `revenue` | `net` | `refunds` | `failures` (백엔드 `switch`문 — 그 외 값은 `400 ADMIN_002`)
- **Query**: `slug`(**선택** — 생략 시 전앱 합산 시계열), `from`, `to` (생략 시 최근 30일). **`interval`은 요청에 안 쓰여요** — 응답의 `interval` 필드는 항상 `"day"`로 고정 반환돼요(백엔드가 파라미터를 안 읽음). 프론트 클라이언트(`getAnalytics`)가 `interval` 옵션을 넘겨도 무시돼요. (단, mock 은 넘긴 값을 응답 `interval`에 그대로 echo 해요 — points 는 어느 쪽이든 일별이에요.)
- **Response** (`TimeSeries`):
  ```ts
  interface TimeSeriesPoint { ts: string; value: number }
  interface TimeSeries { metric: string; interval: string; points: TimeSeriesPoint[] }
  ```
  응답 shape은 `slug` 유무와 무관하게 동일해요 — `slug` 생략 시 값만 전앱 합산으로 바뀌어요.
- **데이터 소스**: `slug` 지정 시 해당 앱 스키마만 집계. 생략 시 전 슬러그 fan-out 후 **날짜별로 합산**(대시보드 "전체 매출 추이"·"전체 가입 추이" 차트가 이 경로를 써요).

  | metric | 소스 | 비고 |
  |---|---|---|
  | `signups` | `users.created_at` 일별 | |
  | `revenue` | `payment_history.paid_at` 일별 | § gross 와 동일 시맨틱 |
  | `dau` | `user_activity_days` | § DAU/MAU 정의 |
  | `refunds` | `payment_refunds.refunded_at` 일별 | 건별 환불일 귀속 — 다회 부분환불도 발생일별로 분리돼요 |
  | `net` | `revenue` − `refunds` 일별 | 환불을 음수로 합산. 결제 없이 환불만 있는 날은 **음수** 포인트가 나와요 |
  | `failures` | `audit_logs.occurred_at` 일별 (`result='FAILURE'`) | 대시보드 "실패" 카드와 같은 소스(카드는 24시간 단일값, 이쪽은 일별 추이) |

  `MAU`·`활성 구독`은 시계열 metric 이 없어요 — rolling distinct(MAU)와 시점 복원(구독)이라 쿼리 성격이 달라서 별도 판단 대상이에요.

### [14] `GET /api/admin/analytics/events`

제품 이벤트 요약 — 이벤트별 발생수·순사용자(발생수 내림차순). "이벤트 분석" 화면(`/analytics`)과 "전체 이벤트" 페이지(`/analytics/events`)가 써요. 클라이언트 함수는 `getEventSummary`.

- **Query**: `slug`(선택), `from`, `to`
- **Response**: `EventSummary[]`
  ```ts
  interface EventSummary { eventName: string; count: number; uniqueUsers: number }
  ```
  `uniqueUsers`는 **일별 순사용자의 기간 합**이에요 — 기간 전체의 절대 unique 가 아니에요(데이터 원천이 pre-aggregated daily 롤업 테이블 `analytics_daily`라서). 이벤트는 이름+카운트만 있고 콘텐츠 내용은 없어요(행동+메타데이터만 계측하는 개발 방침).

### [15] `GET /api/admin/analytics/events/{eventName}`

단일 이벤트의 일별 발생수 추이. 클라이언트 함수는 `getEventSeries`.

- **Query**: `slug`(선택), `from`, `to`
- **Response**: `TimeSeries`(§13과 동일 shape — `metric` 자리에 이벤트 이름)

---

## 결제

### [16] `GET /api/admin/apps/{slug}/payments`

앱 결제 내역 검색 + 페이지네이션 — "누가·언제·얼마"를 드릴다운으로 보는 화면(`/payments`)이 써요.

- **Query**: `query`(사용자 이메일 ILIKE), `channel`(`PG`|`IAP`), `status`(`PAID`|`PARTIALLY_REFUNDED`|`REFUNDED`|`FAILED`|`READY`|`CANCELLED`), `type`(`SUBSCRIPTION`|`ONE_TIME`), `from`, `to`(ISO-8601, 생략 가능), `page`(기본 0), `size`(기본 20)
- **Response**: `PageResponse<AdminPaymentListItem>`
  ```ts
  interface AdminPaymentListItem {
    id: number; userId: number; userEmail: string; channel: string
    amount: number; refundedAmount: number; currency: string; status: string
    paymentType: 'SUBSCRIPTION' | 'ONE_TIME'
    paidAt: string | null; refundedAt: string | null; externalId: string
    method?: string | null // 결제수단(신용카드/계좌이체/간편결제/Google Play 등 — PG 웹훅 pay_method)
    methodDetail?: string | null // 결제수단 상세 — 신용카드는 마스킹 카드번호(BIN 6 + 뒤 4)
    refundReason?: string | null
    periodStart?: string | null; periodEnd?: string | null // 구독 결제의 현재 기간(일할계산 근거)
  }
  ```
- **`refundedAmount`**: 누적 환불액(`payment_history.refunded_amount`). `amount - refundedAmount`가 남은 환불 가능액이에요(부분환불 §17).
- **`paymentType`**: 결제가 `subscriptions`/`subscription_renewals`를 참조하면 `SUBSCRIPTION`(구독 갱신/최초 결제), 아니면 `ONE_TIME`(부가 아이템 등 단건 구매) — 백엔드가 참조 관계로 유도해 내려주는 파생 필드예요(별도 저장 컬럼이 아님).
- **`periodStart`/`periodEnd`**: 결제에 링크된 구독의 현재 기간(`subscriptions.started_at`/`expires_at`). 프론트 환불 모달의 **일할계산** 근거이고, 비구독/미링크는 `null`.
- **상태 표시**: 프론트는 이 영어 enum 을 `lib/paymentStatus`로 한글 라벨(결제완료/부분환불/환불완료/실패/대기/취소) + 태그색으로 변환해 표시해요(백엔드는 영어 유지).
- **에러**: 알 수 없는 슬러그 → `404 ADMIN_003`.

### [17] `POST /api/admin/apps/{slug}/payments/{paymentId}/refund`

결제 환불(전액/부분) — 결제 내역 화면(`/payments`)의 "환불" 버튼이 써요. **PG 채널 + `PAID`/`PARTIALLY_REFUNDED` 상태**인 결제가 대상이에요 — IAP(인앱결제)는 스토어(Google Play/App Store) 쪽 환불 절차를 따로 타기 때문에 이 엔드포인트로는 처리할 수 없어요.

- **Request Body**: `{ amount?: number | null; reason: string }` — `amount` 생략(또는 `null`)이면 **남은 잔액 전액** 환불, 양수면 그 금액만 **부분환불**(잔액 이하). `reason` 필수(`@Positive amount`).
- **Response**: 갱신된 `AdminPaymentListItem`(§16과 동일 shape) — 잔액을 다 환불하면 `status: 'REFUNDED'`, 일부만 남기면 `status: 'PARTIALLY_REFUNDED'`(잔액 남는 한 여러 번 환불 가능), `refundedAmount` 누적·`refundedAt` 갱신.
- **일할계산(프론트)**: 구독 결제는 `periodStart`/`periodEnd`로 모달에서 잔여기간 비례 환불액(`amount × 잔여일 ÷ 전체일`)을 제안해요.
- **에러**: PG 채널 아님 → `400 ADMIN_006`. 환불 불가 상태(이미 전액 환불·`PAID`/`PARTIALLY_REFUNDED` 아님) → `400 ADMIN_021`. 요청 금액이 남은 잔액 초과 → `400 ADMIN_020`(`amount≤0`은 그 전에 `@Positive` bean validation `422 CMN_*`). 결제 없음 → `404 ADMIN_007`. 슬러그 없음 → `404 ADMIN_003`.
- **프론트 처리**: `PaymentsPage` 액션 컬럼은 `channel==='PG' && (status==='PAID' || status==='PARTIALLY_REFUNDED')`인 행만 버튼을 노출하고, 모달에서 결제금액/이미환불/잔액 + 금액 입력([전액] 프리셋) + 사유를 받아요. 사유는 **프리셋 콤보**(단순 변심·구독 해지·중복 결제·결제 오류·불만족·직권 취소)에서 고르고, "직접 입력…" 선택 시에만 TextArea 로 자유 입력해요. 환불 성공 시 곧바로 영수증 팝업(§18b)이 열려요.

### [18] `GET /api/admin/apps/{slug}/payments/{paymentId}/refunds`

환불 이력(원장) — 환불 모달의 "환불 이력" 리스트가 써요. 다회 부분환불 시 건별 금액·사유·시각·처리자를 최신순으로 반환해요(`payment_history`의 누적값만으로는 건별 정보가 덮이기 때문).

- **Response**: `AdminRefundHistoryItem[]`
  ```ts
  interface AdminRefundHistoryItem {
    id: number; amount: number; reason: string | null
    operator: string | null; refundedAt: string
  }
  ```
- **데이터 소스**: `payment_refunds` 원장(환불 1건 = 1행). 매출 `refunded` 집계(§ gross/net)도 이 원장을 씁니다.

### [18b] `GET /api/admin/apps/{slug}/payments/{paymentId}/receipt` — **mock 전용**

결제 영수증 payload — **미리보기 팝업과 이메일 발송이 같은 이 payload 를 렌더**해요(단일 소스). 서버가 금액 원장(결제 + 환불 차감 행)과 발행처(§18d 설정)까지 조립해 내려주므로 화면과 메일 내용이 어긋날 수 없어요. 환불완료/부분환불 건의 상세 팝업 [영수증] 버튼과 환불 성공 직후 자동 팝업이 써요.

- **Response**: `AdminPaymentReceipt`
  ```ts
  interface AdminPaymentReceipt {
    receiptNo: string; issuedAt: string
    serviceName: string                   // 결제가 발생한 서비스(앱)명 — 영수증 상단 워드마크
    issuer: AdminReceiptSettings          // §18d — 발행처(사업자) 정보
    paymentId: number; userEmail: string; channel: string
    method: string                        // 결제수단(신용카드/계좌이체 등)
    methodDetail: string | null           // 신용카드의 마스킹 카드번호(매출전표 관행: BIN 6 + 뒤 4)
    externalId: string                    // 거래번호(결제수단과 별도 행)
    status: string; paidAt: string | null
    lines: { label: string; amount: number }[] // 결제 행 양수, 환불 차감 행 음수
    netAmount: number                     // 최종 결제액 = 결제 금액 − 환불 합
  }
  ```
- **실서버 반영 필요**: template-spring `AdminPaymentsController` 에 아직 없어요. `VITE_USE_MOCK=false` 에선 404 — 백엔드에 §18b~§18d 3종(영수증 조립 + 메일 발송 + 발행처 설정 저장)이 함께 추가돼야 해요.

### [18c] `POST /api/admin/apps/{slug}/payments/{paymentId}/receipt-email` — **mock 전용**

§18b 와 **같은 payload** 로 만든 영수증을 결제자 이메일로 발송해요(서버측 메일 템플릿 렌더). 관리자가 수신자를 바꿀 수 없어요(결제자 고정).

- **Response**: `{ sentTo: string }` — 발송된 수신자 이메일.
- **에러**: 결제 없음 → `404 ADMIN_007`.

### [18d] `GET`/`PUT /api/admin/settings/receipt` — **mock 전용**

영수증 발행처 설정(설정 > 영수증 관리 카드) — 영수증(§18b/§18c) 하단 발행처 표기에 쓰여요.

- **Body/Response**: `AdminReceiptSettings`
  ```ts
  interface AdminReceiptSettings {
    email: string; businessNumber: string  // 사업자등록번호
    address: string; businessName: string  // 사업장 주소 · 사업장명(상호)
    phone: string                          // 대표 연락처(핸드폰번호)
  }
  ```
- **에러(PUT)**: 5개 필드 중 하나라도 빈 값 → `422 CMN_001`.

---

## 파일 (스토리지 모더레이션)

### [19] `GET /api/admin/apps/{slug}/files`

파일 화면(`/files`)이 쓰는 업로드 파일 목록 — **서버 페이지네이션**이에요(`PageResponse`, users/payments 와 동일 계약이라 `useAdminList`를 그대로 재사용). 정렬은 최신순.

- **Query(실서버)**: `prefix`(접두사 매치), `kind`(`image`|`video`|`audio` — 타입 탭), `status`(`deleted`면 soft-delete 된 "삭제 대상"만, 없으면 정상+검역), `source`(`user`|`post`|`other` — 출처(연관 대상) 필터: 사용자/게시물/그 외·미연관. 서버는 `USER`/`POST`/`OTHER` 로 매핑해 적용), `assocId`(출처 드릴다운 — 목록의 출처 태그 클릭 시 `source`와 함께 특정 연관 id 로 좁힘. `source` 와 AND 결합), `unassigned`(`true`면 연관 id 없는 선업로드(orphan)만 — 번호 없는 출처 칩 드릴다운), `page`(기본 0), `size`
- **Query(mock 전용 확장)**: `filename`(원본 파일명/`key` 부분일치), `uploader`(업로더 부분일치), `quarantined`(`true`=검역만/`false`=정상만), `createdFrom`/`createdTo`(업로드 시각 범위, ISO) · `modifiedFrom`/`modifiedTo`(수정 시각 범위, ISO) — 실서버 컨트롤러는 아직 안 읽는 파라미터라 **실서버에선 무시**돼요. 클라이언트(`getAppFiles`)는 전부 넘길 수 있어요.
- **Response**: `AdminFileList` = `PageResponse<AdminFile>`
  ```ts
  type AdminFileStatus = 'ACTIVE' | 'QUARANTINED' | 'DELETED'
  interface AdminFile {
    key: string; size: number; lastModified: string; url: string; quarantined: boolean
    createdAt?: string | null          // 업로드 시각(attachment_file.created_at)
    status?: AdminFileStatus           // DELETED = soft-delete(30일 후 purge)
    originalFilename?: string | null   // 원본 업로드 파일명 — key(UUID 기반)와 별개
    contentType?: string | null        // MIME 타입
    durationSec?: number | null        // 오디오/영상 재생 길이(초) — 실서버 스토리지 미구현, mock 제공
    associatedType?: string | null     // polymorphic 연관 대상("USER"/"POST")
    associatedId?: number | null
    uploadedBy?: string | null         // 업로더 — PII, 마스킹 대상
    uploadedIp?: string | null         // 업로드 시점 IP — PII, 마스킹 대상
    userAgent?: string | null          // 업로드 기기 UA — PII, 마스킹 대상
    deleteReason?: string | null       // soft-delete 사유
    deletedAt?: string | null
    purgeAt?: string | null            // 영구삭제 예정 시각(= deletedAt + 30일) — D-day 표시용
  }
  ```
  `url`은 presigned GET URL(약 10분 유효) — 목록을 오래 띄워둔 채 미리보기/원본보기를 누르면 만료돼 있을 수 있어요(그럴 땐 재검색으로 새 URL을 받아요).
- **PII 마스킹**: `PERM_FILES_UNMASK` 없는 세션엔 `uploadedBy`/`uploadedIp`/`userAgent`가 마스킹돼 내려와요. 원본은 §20 "조회"로 단건 열람.
- **에러**: 알 수 없는 슬러그 → `404 ADMIN_003`

### [20] `GET /api/admin/apps/{slug}/files/{key}/reveal`

파일 **원본 열람("조회")** — 마스킹 티어가 특정 파일의 업로더·IP·기기 원본을 확인해요. 클라이언트 함수는 `revealFile`. 서버가 열람을 `user_read_history`에 기록해요(§11과 같은 패턴).

- **Response**: 원본 `AdminFile`
- **에러**: 파일 없음 → `404 ADMIN_010`. 슬러그 없음 → `404 ADMIN_003`.

### [21] `POST /api/admin/apps/{slug}/files/quarantine?key=`

파일을 사용자에게 안 보이게 검역(숨김) 처리 — 오브젝트 키를 `quarantine/` 프리픽스로 옮기는 시맨틱이에요(별도 플래그가 아니라 저장 위치 자체가 바뀜). 갱신된 `AdminFile`(새 `key`, `quarantined: true`)을 반환해요.

- **Query**: `key`(필수)
- **Response**: 갱신된 `AdminFile`
- **에러**: 슬러그 없음 → `404 ADMIN_003`. `key` 파일 없음 → `404 ADMIN_010`. 이미 검역된 파일을 다시 검역 시도 → `400 ADMIN_008`

### [22] `POST /api/admin/apps/{slug}/files/restore?key=`

검역된 파일을 원상 복구(`quarantine/` 프리픽스 제거). 갱신된 `AdminFile`(`quarantined: false`)을 반환해요.

- **Query**: `key`(필수 — 검역된 상태의 현재 키, 즉 `quarantine/` 프리픽스가 붙은 값)
- **Response**: 갱신된 `AdminFile`
- **에러**: 슬러그 없음 → `404 ADMIN_003`. `key` 파일 없음 → `404 ADMIN_010`. 검역 상태가 아닌 파일을 복원 시도 → `400 ADMIN_009`

### [23] `POST /api/admin/apps/{slug}/files/restore-deleted?key=`

soft-delete 된 파일 복원 — "삭제 대상"(`status: 'DELETED'`)을 정상으로 되돌려요(purge 예약 취소). 클라이언트 함수는 `restoreDeletedFile`.

- **Query**: `key`(필수)
- **Response**: 갱신된 `AdminFile`
- **에러**: 슬러그 없음 → `404 ADMIN_003`. `key` 파일 없음 → `404 ADMIN_010`. (mock 은 삭제 대상(`DELETED`)이 아닌 파일이면 `400 ADMIN_009`를 재사용해 거절하지만, 실서버엔 이 분기가 없어요 — 상태 검증 없이 그대로 ACTIVE 로 되돌려요.)

### [24] `DELETE /api/admin/apps/{slug}/files?key=`

파일 삭제 — **즉시 영구삭제가 아니라 soft-delete**예요. 사유가 필수이고, 30일 뒤 purge 스케줄러(`AttachmentPurgeScheduler`)가 실제 오브젝트/row 를 지워요. 그 전엔 "삭제 대상" 탭에서 §23으로 복원할 수 있어요.

- **Query**: `key`(필수, 삭제 대상 오브젝트 키)
- **Request Body**: `{ reason: string }` — 삭제 사유 필수
- **Response**: 실서버는 갱신된 `AdminFile`(`status: 'DELETED'`, `deleteReason`/`deletedAt`/`purgeAt` 채워짐)을 내려주지만, 클라이언트(`deleteAppFile`)는 반환값을 쓰지 않고 `Promise<void>`로 버려요(mock 도 `data: null` 반환) — 갱신 행이 필요하면 목록을 재조회해요.
- **에러**: 슬러그 없음 → `404 ADMIN_003`. `key` 파일 없음 → `404 ADMIN_010`.

---

## 게시물 (콘텐츠 모더레이션)

**공유(공개) 게시물** 콘솔 계약이에요 — 앱들의 공개 게시판(`posts`)을 전량 조회하고 숨김/삭제/복원해요. 프라이빗 기록은 이 도메인에 안 와요(각 앱 자체 테이블). 파일(§19~§24)과 동일한 상태 전이·soft-delete·purge 패턴이고, 상태는 `ACTIVE`/`HIDDEN`/`DELETED` 3종이에요. **운영 작성**(공지·이벤트 등 관리자가 직접 쓰는 글 — 작성/수정/이미지 업로드)은 §39~§41, 본문(markdown + `attachment://`) 계약은 § "본문 계약"을 보세요.

### [25] `GET /api/admin/apps/{slug}/content`

게시물 목록(전 상태) + 필터. 클라이언트 함수는 `getAppContent`.

- **Query**: `board`, `status`(`ACTIVE`|`HIDDEN`|`DELETED`), `page`(기본 0), `size`(기본 20)
- **Response**: `PageResponse<AdminPost>`
  ```ts
  type PostAuthorType = 'USER' | 'ADMIN'
  interface AdminPost {
    id: number
    authorUserId: number | null      // 앱 회원 작성자 id — ADMIN 작성 글은 null
    authorType: PostAuthorType       // 'USER'(앱 회원, 기본) | 'ADMIN'(콘솔 관리자 — §39)
    authoredBy: string | null        // ADMIN 작성 시 콘솔 계정 email, USER 작성이면 null
    authorNickname: string | null    // 작성 시점 회원 닉네임 스냅샷 — ADMIN 작성이면 null
    board: string
    title: string | null; body: string | null
    status: string // ACTIVE | HIDDEN | DELETED
    hiddenAt: string | null; hiddenReason: string | null
    deletedAt: string | null; deleteReason: string | null
    purgeAt: string | null; createdAt: string
  }
  ```
  공개 게시물이라 작성자(`authorUserId`) 마스킹은 없어요(파일/유저의 PII reveal 패턴 불필요 — 백엔드 방침). `authorType`/`authoredBy`는 운영 작성(§39) 도입으로 추가된 필드예요 — `authorType='ADMIN'`이면 `authorUserId=null` + `authoredBy`=관리자 email(`hidden_by`/`deleted_by`와 동일 규약)이에요.

  `authorNickname` 은 **작성 시점 닉네임 스냅샷**이에요 — 앱 회원이 글을 쓸 때 닉네임이 없으면 서버가 `users.nickname` 을 `익명의누군가#{userId}` 로 강제 세팅한 뒤 그 값을 `posts.author_nickname` 에 박제해요(이후 닉 변경에도 과거 글 표기는 불변). 콘솔 목록·상세의 작성자 표기는 이 필드를 쓰고, 없으면(구 데이터) `회원 #{id}` 로 폴백해요.
- **에러**: 알 수 없는 슬러그 → `404 ADMIN_003`

### [26] `GET /api/admin/apps/{slug}/content/{id}`

게시물 상세 — 본문 전체 + `properties` + 첨부 목록(모더레이션 판단용). 클라이언트 함수는 `getPostDetail`.

- **Response** (`AdminPostDetail`): `AdminPost` 필드(§25 — `authorType`/`authoredBy` 포함) + `properties: string | null` + `attachments: AdminPostAttachment[]`
  ```ts
  interface AdminPostAttachment {
    id: number; originalFilename: string | null; contentType: string | null
    sizeBytes: number | null; url: string | null   // presigned GET(~10분)
  }
  ```
- **본문 렌더**: `body`는 markdown(관리자 작성 글) 또는 plain text(기존 회원 글)일 수 있어요 — § "본문 계약" 참고. 본문 내 이미지는 `![alt](attachment://{id})` 참조라 `attachments` 배열의 `id`→`url`로 해석해서 렌더해요.
- **에러**: 게시물 없음 → `404 ADMIN_022`. 슬러그 없음 → `404 ADMIN_003`.

### [27] `POST /api/admin/apps/{slug}/content/{id}/hide`

게시물 숨김(`ACTIVE → HIDDEN`) — 사유 필수. **Body**: `{ reason: string }`. 갱신된 `AdminPost` 반환. 에러: `404 ADMIN_022`/`ADMIN_003`.

### [28] `POST /api/admin/apps/{slug}/content/{id}/restore`

숨김 해제(`HIDDEN → ACTIVE`, 재공개). 갱신된 `AdminPost` 반환. 에러: `404 ADMIN_022`/`ADMIN_003`.

### [29] `POST /api/admin/apps/{slug}/content/{id}/restore-deleted`

soft-delete 복원(`DELETED → ACTIVE`, purge 예약 취소). 갱신된 `AdminPost` 반환. 에러: `404 ADMIN_022`/`ADMIN_003`.

### [30] `DELETE /api/admin/apps/{slug}/content/{id}`

soft-delete(`→ DELETED`) — 사유 필수, `purgeAt = now + 30일`에 purge 스케줄러가 실삭제. **Body**: `{ reason: string }`. 갱신된 `AdminPost` 반환. 에러: `404 ADMIN_022`/`ADMIN_003`.

---

## 관리자 계정 · 역할 (master 전용)

`PERM_ADMIN_MANAGE`(기본 grant 로는 master 만) 필요 — 역할·권한 화면(`/roles`)이 써요. 편집은 **자기보다 낮은 티어만** 가능해요(상급자·동급·본인 편집 시도 → `403 ADMIN_017`).

### [31] `GET /api/admin/admins`

관리자 계정 목록. 비밀번호 해시는 내려오지 않아요.

- **Response**: `AdminAccountRow[]`
  ```ts
  interface AdminAccountRow { id: number; email: string; displayName: string | null; role: AdminRole }
  ```

### [32] `POST /api/admin/admins`

계정 생성. **Body**: `{ email, password, role, displayName? }`. 생성된 `AdminAccountRow` 반환.

- **에러**: 이메일 중복 → `409 ADMIN_011`. 미상 역할 → `400 ADMIN_013`. 상급/동급 티어 생성 시도 → `403 ADMIN_017`.

### [33] `PATCH /api/admin/admins/{id}`

역할/표시이름 변경. **Body**: `{ role?, displayName? }`. 갱신된 `AdminAccountRow` 반환.

- **에러**: 계정 없음 → `404 ADMIN_012`. 본인 변경 → `400 ADMIN_014`. 마지막 master 강등 → `400 ADMIN_015`. 미상 역할 → `400 ADMIN_013`. 상급/동급 → `403 ADMIN_017`.

### [34] `DELETE /api/admin/admins/{id}`

계정 삭제. 갱신된 목록(`AdminAccountRow[]`)을 반환해요.

- **에러**: 계정 없음 → `404 ADMIN_012`. 본인 삭제 → `400 ADMIN_014`. 마지막 master 삭제 → `400 ADMIN_015`. 상급/동급 → `403 ADMIN_017`.

### [35] `POST /api/admin/admins/{id}/password`

타 계정 비밀번호 재설정. **Body**: `{ newPassword: string }`. 성공 시 빈 데이터.

- **에러**: 계정 없음 → `404 ADMIN_012`. 상급/동급 → `403 ADMIN_017`.

### [36] `POST /api/admin/me/password`

**본인** 비밀번호 변경 — 모든 콘솔 계정이 쓸 수 있어요(`PERM_ADMIN_MANAGE` 불필요, 인증만). 설정 화면(`/settings`)이 써요.

- **Body**: `{ currentPassword: string, newPassword: string }`
- **에러**: 현재 비밀번호 불일치 → `400 ADMIN_016`

### [37] `GET /api/admin/roles/permissions`

역할×권한 매트릭스 조회.

- **Response** (`RolePermissionMatrix`):
  ```ts
  type PermCategory = 'FIXED' | 'DOMAIN' | 'GOVERNANCE'
  interface RolePermCol { key: string; category: PermCategory }
  interface RolePermCell { permission: string; granted: boolean; editable: boolean }
  interface RolePermRow { role: AdminRole; tier: number; cells: RolePermCell[] }
  interface RolePermissionMatrix { permissions: RolePermCol[]; roles: RolePermRow[] }
  ```
  `editable`은 호출자 기준이에요 — `FIXED` 권한(대시보드 등 코드 고정)과 상급/동급 티어 행은 토글 불가.

### [38] `PUT /api/admin/roles/permissions`

매트릭스 편집 — 역할별 **목표 집합**을 통째로 저장해요(diff 아님). 서버(`PermissionCatalog`)가 계층·의존(`WRITE`/`UNMASK` ⇒ 해당 `READ`)·editable 범위를 재검증해요.

- **Body**: `{ roles: { role: AdminRole; permissions: string[] }[] }`
- **Response**: 갱신된 `RolePermissionMatrix`
- **에러**: 상급/동급 티어 편집 → `403 ADMIN_017`. 편집 불가 권한 → `400 ADMIN_018`. READ 없이 WRITE/UNMASK 부여 → `400 ADMIN_019`.

---

## 게시물 작성 (운영 콘텐츠)

모더레이션(§25~§30)에 더해, 관리자가 콘솔에서 **직접 게시물을 작성**하는 계약이에요 — 공지·이벤트 같은 운영 글을 앱들의 공개 게시판(`posts`)에 올려요. 백엔드는 §25~§30과 같은 `AdminContentController`이고, 권한은 **메서드별로 분리**돼 있어요 — 백엔드 `SecurityConfig`가 `/api/admin/apps/*/content/**` 에 아래 순서(구체→일반)로 매처를 걸어요.

| 메서드 | 권한 |
|---|---|
| `DELETE` | `PERM_CONTENT_DELETE` |
| `POST` · `PUT` (작성·수정·모더레이션·복원) | `PERM_CONTENT_MODERATE` |
| `GET` | `PERM_CONTENT_READ` |

프론트는 `NAV_PERM['/content']` 를 `read=PERM_CONTENT_READ` · `write=PERM_CONTENT_MODERATE` 로 걸어 작성/수정 버튼을 숨기지만(`src/lib/rbac.ts`), 최종 강제는 백엔드예요. `PERM_CONTENT_WRITE` 는 폐기된 권한이에요(마이그레이션 `V008` 에서 `CONTENT_MODERATE`/`CONTENT_DELETE` 로 분리, `V009` 에서 제거).

### 본문 계약 — markdown + `attachment://`

`posts.body`의 계약이에요. 작성(§39)·수정(§40)·상세 렌더(§26)가 전부 이 규칙을 따라요.

- **`body`는 markdown**이에요 — 관리자 작성 글(공지·이벤트)은 제목·굵게·목록·링크 등 markdown 문법으로 저장돼요.
- **인라인 이미지는 `![alt](attachment://{attachmentId})` 참조**로 넣어요. `attachmentId`는 선업로드(§41)로 발급받은 첨부 id예요.
  - **presigned URL 을 본문에 박제하지 않는 이유**: presigned GET URL 은 ~10분이면 만료돼요 — 본문에 URL 을 저장하면 저장 직후부터 깨진 이미지가 돼요. 참조(`attachment://{id}`)만 저장하고, 렌더 시점에 상세 응답(§26)의 `attachments[]`에서 `id`→`url`(그때그때 새로 발급된 presigned GET)로 해석해요.
- **plain text 하위호환**: 기존 앱 회원 글은 markdown 이 아닌 평문이에요 — 렌더러(프론트 `PostBody`)가 평문 문장도 문단으로 안전하게 렌더하도록 통일 처리해요. 서버는 본문 형식을 검증하지 않아요(계약상 관례).
- **작성 주체 필드**: `authorType`(`'USER'` | `'ADMIN'`)이 작성 주체를 구분하고, `authorType='ADMIN'`이면 `authorUserId=null` + `authoredBy`=콘솔 계정 email 이에요(§25 스키마 참고).

### [39] `POST /api/admin/apps/{slug}/content`

관리자 게시물 작성 — 서버가 `authorType='ADMIN'`, `authoredBy`=현재 콘솔 계정 email 로 저장하고, `attachmentIds`의 선업로드 첨부를 게시물에 **연관 확정**해요. 클라이언트 함수는 `createPost`.

- **Request body** (`AdminPostWriteBody` — 백엔드 `AdminPostWriteRequest` 미러):
  ```ts
  interface AdminPostWriteBody {
    board: string            // 선택, ≤50 — 미선택은 '' (미분류). 콘솔 셀렉트는 고정 4종(notice/event/qna/free)
    title: string            // ≤300 (백엔드 스키마상 선택이지만 콘솔 폼은 필수 입력)
    body: string             // markdown — § "본문 계약"
    properties?: string | null // 자유 JSON 문자열 — 콘솔 작성 화면은 해시태그를 {"tags": string[]} 로 저장
    attachmentIds?: number[] // §41 로 발급받은 첨부 id — 저장 시 연관 확정
  }
  ```
- **board 시맨틱**: 콘솔에선 선택값이에요 — 미선택이면 서버가 빈 문자열(미분류)로 저장하고, 상세 화면은 board 태그를 숨겨요. (앱 회원용 `POST /api/apps/{slug}/posts` 의 `board`는 여전히 필수 — 콘솔 계약만 완화.)
- **해시태그**: 작성 화면의 `#태그` 칩(다중 선택)은 `properties = JSON.stringify({ tags })` 로 실려요. 상세 화면이 `properties.tags` 배열을 파싱해 칩으로 렌더해요 — 서버는 `properties`를 자유 JSON 으로 취급하므로 별도 스키마 검증은 없어요.
- **Response**: `AdminPostDetail`(§26과 동일 shape — 방금 쓴 글의 상세)
- **감사**: `admin.content.create`(대상 앱 스키마 `audit_logs`).
- **에러**: 슬러그 없음 → `404 ADMIN_003`. 첨부 연관 확정 실패(첨부 부재·slug 불일치·비 ACTIVE·이미 다른 게시물에 연관) → `400 ADMIN_023`.

### [40] `PUT /api/admin/apps/{slug}/content/{id}`

게시물 수정 — `board`/`title`/`body`/`properties`를 교체하고 첨부를 재연관해요. **작성자(`authorType`/`authoredBy`)·상태는 불변**이에요. 클라이언트 함수는 `updatePost`. 권한은 §39와 동일하게 `PERM_CONTENT_MODERATE`(PUT matcher).

- **Request body**: §39와 동일(`AdminPostWriteBody`). `attachmentIds`는 **전체 재전송**이에요 — 이미 이 게시물에 연관된 id 는 멱등 통과, 새 선업로드 id 만 연관 확정돼요.
- **Response**: 갱신된 `AdminPostDetail`
- **감사**: `admin.content.update`.
- **에러**: 게시물 없음 → `404 ADMIN_022`. 슬러그 없음 → `404 ADMIN_003`. 첨부 연관 실패 → `400 ADMIN_023`.

### [41] `POST /api/admin/apps/{slug}/content/uploads`

본문 이미지 **선업로드 티켓 발급** — 에디터의 이미지 버튼이 파일 선택 직후 호출해요. 서버가 첨부 메타를 미연관(`associatedId=null`) 상태로 선등록하고 presigned PUT/GET URL 쌍을 발급해요(`<slug>-uploads` 버킷, 둘 다 10분 만료). 클라이언트 함수는 `uploadContentImage`.

- **Request body**: `{ filename: string, contentType: string, sizeBytes: number }` — `filename` ≤255, `contentType` ≤100 · **`image/*`만 허용**, `sizeBytes` 양수.
- **Response** (`ContentUploadTicket`):
  ```ts
  interface ContentUploadTicket {
    attachmentId: number  // 본문 참조(attachment://{id})·attachmentIds 연관에 쓰는 id
    uploadUrl: string     // presigned PUT — 파일 바디를 그대로 업로드
    previewUrl: string    // presigned GET — 에디터 미리보기용(~10분 만료)
    expiresAt: string     // uploadUrl 만료 시각
  }
  ```
- **업로드 → 작성 시퀀스**:
  1. 이미지 선택 → §41 호출로 티켓 발급
  2. `uploadUrl`(presigned PUT)로 파일 직접 업로드(`uploadFileToUrl` — JSON 봉투·인증 헤더 없이 스토리지 직행)
  3. 에디터 본문엔 `![alt](attachment://{attachmentId})` 참조 삽입(화면 미리보기는 `previewUrl`)
  4. 글 저장(§39/§40) 시 `attachmentIds`에 해당 id 를 실어 **연관 확정** — 확정 전 첨부는 미연관 상태로 남아요
- **감사**: `admin.content.upload`(resourceType `AttachmentFile`).
- **에러**: 슬러그 없음 → `404 ADMIN_003`. `contentType`이 `image/*` 아님 → `422 CMN_001`(bean validation 계약, 실서버·mock 동일).

---

## 앱 버전 관리 (강제 업데이트)

**앱-스코프**예요 — 사이드바에서 앱을 선택하면 나타나는 앱별 메뉴(사용자·파일 등과 동일 패턴)로, `{slug}` path 변수로 해당 앱만 조회·교체해요. 콘솔 자신의 admin 스키마 `app_min_versions` 테이블에서 `slug` 컬럼으로 좁혀요. "앱 버전" 화면(`/app-versions`, 앱 선택 시 노출)이 써요. 플랫폼은 **iOS·Android 2개만**(전역 `ALL` 없음) — 앱당 고정 2행이라 우선순위 해석·dedup 검증이 구조적으로 불필요해요. **2단계**(강제/경고) 임계값 — `forceMinVersion`·`warnMinVersion` 모두 nullable이라 정책 없음(둘 다 비움)도 허용해요.

### [44] `GET /api/admin/apps/{slug}/app-versions`

해당 앱의 iOS/Android 최소버전(강제 업데이트) 규칙(최대 2행, 플랫폼 순 정렬). 권한 `PERM_APPS_READ`. 클라이언트 함수는 `getAppVersions(slug)`.

- **Response**: `AppVersionRule[]` — `slug` 는 path 로만 특정되므로 이 타입엔 없어요.
  ```ts
  interface AppVersionRule {
    platform: 'IOS' | 'ANDROID'
    enabled: boolean                // 이 플랫폼 규칙 사용 여부 — 화면이 Switch 로 편집해 PUT 으로 전송해요
    forceMinVersion: string | null  // "x.y.z" — major.minor.patch 형식만 파싱해요(빌드/프리릴리스 접미사 무시). null=강제 정책 없음
    warnMinVersion: string | null   // 위와 동일 형식. null=경고 정책 없음
    storeUrl?: string | null   // optional — 강제/경고 화면에서 스토어 딥링크로 쓰여요
    message?: string | null    // optional — 안내 메시지 override(비면 기본 문구)
  }
  ```

### [45] `PUT /api/admin/apps/{slug}/app-versions`

해당 앱 몫만 **통째 교체**(부분 PATCH 아님) — §38 역할×권한 매트릭스 PUT과 동일 패턴이에요. 다른 슬러그의 행은 건드리지 않아요. 권한 `PERM_APP_VERSION_WRITE`(기본 grant는 `master`만이고, 고정 코드가 아니라 §37~§38 역할×권한 매트릭스로 다른 역할에도 부여할 수 있는 **DOMAIN** 권한이에요 — 고임팩트라 기본은 좁게 시작). 클라이언트 함수는 `putAppVersions(slug, rows)`.

- **Request body**: `{ rules: AppVersionRule[] }`
- **Response**: 교체 후 이 앱의 전체 목록 `AppVersionRule[]`(§44와 동일 shape)
- **검증(서버, 화이트리스트 선거부)**: `slug`가 등록된 슬러그인지, `platform ∈ {IOS, ANDROID}`인지, `forceMinVersion`·`warnMinVersion`이 각각 값이 있을 때 `x.y.z`로 파싱 가능한지, **둘 다 있으면 `forceMinVersion ≤ warnMinVersion`**인지 — 하나라도 실패하면 `422 CMN_001`. 전 건 검증을 통과해야 트랜잭션 1개로 replace 돼요(부분 실패 없음). `(slug, platform)` 중복 방지는 DB 유니크 제약에 위임(앱당 고정 2행이라 UI 가 같은 platform 을 두 번 보낼 여지가 구조적으로 없어요).
- **캐시**: 저장 성공 시 서버 캐시(`DbAppVersionResolver`, TTL 30초)를 즉시 evict해서 다음 요청부터 곧바로 반영돼요.
- **프론트 게이팅**: `rbac.ts`의 `NAV_PERM['/app-versions']`는 read=`PERM_APP_VERSION_READ`, write=`PERM_APP_VERSION_WRITE`로 **분리**돼 있어요 — 읽기 권한만 있는 역할(mock 기본 grant 의 admin)은 화면을 보되 사용 토글·저장 버튼이 잠겨요. 두 권한 모두 없으면 메뉴 자체가 안 보여요.
- **미리보기**: 화면은 저장 전에도 입력(message·forceMinVersion·warnMinVersion·storeUrl)을 실시간 반영해요 — 두 사용자 화면(강제=닫을 수 없는 차단 다이얼로그 / 경고=닫을 수 있는 안내)을 **세그먼트 탭**으로 직접 골라 봐요. API 계약과는 무관한 순수 프론트 UX예요.

### 시맨틱 — 실제 게이트

- 클라이언트 버전 < `forceMinVersion`(설정 시) → 강제(닫을 수 없는 차단). 서버 API 는 426.
- `forceMinVersion` ≤ 클라이언트 버전 < `warnMinVersion`(설정 시) → 경고(닫을 수 있는 안내). 서버 API 는 정상 통과.
- 그 외(클라이언트 버전 ≥ `warnMinVersion`, 또는 둘 다 미설정) → 아무것도 안 함.
- 이 화면·API는 규칙을 **관리**할 뿐이에요 — 실제로 게이트를 거는 건 앱 API 쪽이에요: `MinAppVersionFilter`(`X-App-Platform` 헤더 + `426 CMN_010`, 강제만 426)와, 앱 스플래시가 능동 조회하는 공개 엔드포인트 `GET /api/apps/{slug}/app-version`. 두 경로 다 `template-spring`의 Flutter↔Backend 통합 문서가 정본이에요.

---

## 스키마 조회 (ERD 콘솔)

**앱-스코프**예요 — 사이드바에서 앱을 선택하면 나타나는 앱별 메뉴(`/schema`, 사용자·파일 등과 동일 패턴)로, `{slug}` path 변수로 해당 앱의 스키마만 조회해요. ERD 콘솔 Phase 1(읽기 전용 스키마 뷰어)이 써요 — 테이블·컬럼·FK·인덱스를 시각화만 할 뿐 DDL 은 실행하지 않아요(Flyway 가 진실의 원천).

### [46] `GET /api/admin/apps/{slug}/schema`

해당 앱 스키마의 테이블·컬럼·FK·인덱스 전체 스냅샷(`information_schema`/`pg_indexes` 기반). 권한 `PERM_SCHEMA_READ`. 클라이언트 함수는 `getAppSchema(slug)`.

- **Response**: `SchemaResponse`
  ```ts
  interface SchemaResponse {
    tables: SchemaTable[]
  }
  interface SchemaTable {
    name: string
    comment: string | null
    columns: SchemaColumn[]
    foreignKeys: SchemaForeignKey[]
    indexes: SchemaIndex[]
  }
  interface SchemaColumn {
    name: string
    type: string              // information_schema.columns 를 사람이 읽기 쉬운 문자열로 합성한 값(예: character varying(50))
    nullable: boolean
    defaultValue: string | null
    primaryKey: boolean
    comment: string | null
  }
  interface SchemaForeignKey {
    column: string             // 이 테이블의 컬럼 → refTable.refColumn
    refTable: string
    refColumn: string
    onDelete: string
  }
  interface SchemaIndex {
    name: string
    columns: string[]          // 단일/복합 컬럼 모두 표현
    unique: boolean
  }
  ```
- **에러**: 알 수 없는 슬러그 → `404 ADMIN_003`. 빈 스키마는 에러가 아니라 `tables: []`. 권한 없음(`PERM_SCHEMA_READ` 부재) → `403 CMN_005`.
- **읽기전용**: UI 는 이 응답을 시각화만 할 뿐 DDL 을 실행하지 않아요 — 쓰기(PUT/POST/DELETE) 엔드포인트가 없어요.

### [47] `POST /api/admin/analytics/rollup`

오늘 발생분 제품 이벤트를 `analytics_daily` 로 즉시 집계하는 온디맨드 롤업이에요(평소엔 매일 새벽 자동 집계 — 하루 지연을 없애는 디버깅·검증용). 권한 `PERM_ANALYTICS_ROLLUP`(쓰기 작업 전용, `PERM_ANALYTICS_READ` 의존 — 기본 grant 는 admin·master). 클라이언트 함수는 `rollupAnalytics(opts)`, 이벤트 분석 화면의 "지금 집계" 버튼이 호출해요.

- **Query**: `slug?`(생략 시 전 앱 집계)
- **Response**: `AnalyticsRollup`
  ```ts
  interface AnalyticsRollup {
    slugs: number              // 집계된 앱 수
    eventNames: number         // 반영된 이벤트 이름 수
    aggregatedThrough: string  // 집계 반영 시점(ISO)
  }
  ```
- **에러**: 알 수 없는 슬러그 → `404 ADMIN_003`. 권한 없음 → `403 CMN_005`.

### [48] `GET /api/admin/apps/{slug}/ops/renewal-failures`

운영 신호의 "갱신 실패 (7일)" 드릴다운 — `subscription_renewals` 중 최근 7일 status≠SUCCESS(FAILED=재시도 대기, ABANDONED=재시도 소진) 건을 최신순으로 줘요. 구독자 이메일(결제 PII)이 실려 **권한 `PERM_PAYMENTS_READ`**(운영신호 카드의 APPS_READ 보다 강함 — 매출분석 최근거래와 같은 논리). 클라이언트 함수 `getRenewalFailures(slug)`, 앱 개요의 갱신 실패 카드 클릭이 호출해요.

- **Response**: `RenewalFailure[]` — `{ id, subscriptionId, userEmail, attemptNo, status, attemptedAt, nextRetryAt, errorCode, errorMessage }` (최근 100건 — 초과분 총계는 운영신호 카드 숫자가 기준)
- **에러**: 알 수 없는 슬러그 → `404 ADMIN_003`. 권한 없음 → `403 CMN_005`.

### [49] `GET /api/admin/apps/{slug}/ops/webhooks`

운영 신호의 웹훅 드릴다운 — `payment_webhook_events` 를 `status=pending`(미처리) / `failed`(처리 실패)로 나눠 최신순으로 줘요. payload(JSONB)는 민감할 수 있어 **미노출** — 메타만. 권한 `PERM_APPS_READ`. 클라이언트 함수 `getOpsWebhooks(slug, status)`.

- **Query**: `status`(필수) — `pending` | `failed`
- **Response**: `WebhookEvent[]` — `{ id, source, externalId, receivedAt, processedAt, processError }` (최근 100건)
- **에러**: 알 수 없는 슬러그 → `404 ADMIN_003`. status 미상 값 → `422 CMN_001`. 권한 없음 → `403 CMN_005`.

---

## 발송(메시징) — ⚠ mock 전용 (백엔드 미구현)

발송 화면(`/send`)은 아래 6개 엔드포인트에 **실제로 배선**돼 있지만, `template-spring`엔 아직 대응 컨트롤러가 없어요 — **mock(MSW)에서만 동작**하고, 실서버 모드에선 404가 나요. 백엔드 구현이 따라오면 이 절이 §50+ 로 승격될 예정이에요. 그때까지는 계약(경로·DTO)이 mock 쪽 단독 정의라는 걸 감안하고 보세요.

**권한**: 채널별 — `PERM_SEND_SMS`·`PERM_SEND_EMAIL`·`PERM_SEND_PUSH`(백엔드 V012 가 구 `PERM_SEND` 를 폐기·승계). 하나라도 있으면 화면 진입, 채널 탭·이력 채널 필터는 보유 채널만 노출돼요.

| # | 엔드포인트 | 클라이언트 함수 | 용도 |
|---|---|---|---|
| M1 | `GET /api/admin/apps/{slug}/messages/segments` | `getSendSegments` | 세그먼트 기본 규모(채널 교집합 전) — 대상 카드 |
| M2 | `GET /api/admin/apps/{slug}/messages/preview?channel&segment&userRef` | `getSendPreview` | 채널×대상 유효 수신자 수 미리보기(발송 없음) |
| M3 | `GET /api/admin/apps/{slug}/messages/recipients?channel&segment&page&size` | `getSendRecipients` | 발송 전 실제 수신자 목록 미리보기 |
| M4 | `POST /api/admin/apps/{slug}/messages` | `sendMessage` | 발송 실행 |
| M5 | `GET /api/admin/apps/{slug}/messages?channel&from&to&page&size` | `getMessageHistory` | 발송 이력(`message_send_history` 상당) |
| M6 | `GET /api/admin/apps/{slug}/messages/{messageId}/recipients?page&size` | `getSentRecipients` | 발송된 건의 수신자별 전달 상태 |

핵심 타입(`src/lib/types.ts`):

```ts
type SendChannel = 'SMS' | 'EMAIL' | 'PUSH'
type SendSegmentKey = 'all' | 'premium' | 'free' | 'active' | 'new' | 'user' | 'topic'
type SendTargetType = 'USER' | 'SEGMENT' | 'BROADCAST' | 'TOPIC'
interface SendSegment { key: SendSegmentKey; label: string; count: number }
interface SendPreview { channel: SendChannel; supported: boolean; recipientCount: number }
interface SendResult { result: 'SUCCESS' | 'PARTIAL' | 'FAILED'; recipientCount: number }
```

- `SendPreview.supported: false`는 그 앱이 해당 채널을 못 쓰는 경우예요(예: SMS 는 phone-auth 앱만).
- `sendMessage` Body: `{ channel, targetType, targetRef?, subject?, text }` — SMS 는 `subject: null`, `topic`은 푸시 전용(`TOPIC`+토픽명).

---

## 에러 코드

### Admin 전용 (`AdminError`, 백엔드 `core-admin-impl`) — `ADMIN_001`~`ADMIN_025`

| 코드 | HTTP | enum | 의미 |
|---|---|---|---|
| `ADMIN_001` | 401 | `INVALID_CREDENTIALS` | 이메일 또는 비밀번호가 올바르지 않아요 (로그인 실패) |
| `ADMIN_002` | 400 | `UNSUPPORTED_METRIC` | `analytics/{metric}`의 `metric`이 지원 목록(`dau`/`signups`/`revenue`/`refunds`/`net`/`failures`)에 없음 |
| `ADMIN_003` | 404 | `UNKNOWN_SLUG` | 슬러그를 찾을 수 없음 — 슬러그를 받는 **모든** admin 엔드포인트 공통 |
| `ADMIN_004` | 400 | `INVALID_DATE_RANGE` | `from`/`to`가 ISO-8601 형식이 아님 (billing/audit-logs/analytics) |
| `ADMIN_005` | 404 | `USER_NOT_FOUND` | 사용자 상세/조회 시 해당 `userId` 없음 |
| `ADMIN_006` | 400 | `PG_REFUND_ONLY` | 환불 대상 결제가 PG 채널이 아님(IAP 는 스토어 환불) |
| `ADMIN_007` | 404 | `PAYMENT_NOT_FOUND` | 환불 대상 `paymentId`에 해당하는 결제가 없음 |
| `ADMIN_008` | 400 | `FILE_ALREADY_QUARANTINED` | 이미 검역된 파일을 다시 검역 시도 |
| `ADMIN_009` | 400 | `FILE_NOT_QUARANTINED` | 검역되지 않은 파일을 복원 시도 |
| `ADMIN_010` | 404 | `FILE_NOT_FOUND` | 검역/복원/삭제/열람 대상 파일 없음 |
| `ADMIN_011` | 409 | `ADMIN_EMAIL_EXISTS` | 관리자 계정 생성 시 이미 사용 중인 이메일 |
| `ADMIN_012` | 404 | `ADMIN_ACCOUNT_NOT_FOUND` | `/admins/{id}` 대상 관리자 계정 없음 |
| `ADMIN_013` | 400 | `ADMIN_INVALID_ROLE` | 4티어 코드(viewer/support/admin/master)가 아닌 역할 입력 |
| `ADMIN_014` | 400 | `ADMIN_CANNOT_MODIFY_SELF` | 본인 계정 삭제·역할 변경 시도 |
| `ADMIN_015` | 400 | `ADMIN_LAST_MASTER` | 마지막 master 계정 삭제·강등 시도 |
| `ADMIN_016` | 400 | `ADMIN_WRONG_PASSWORD` | 본인 비밀번호 변경 시 현재 비밀번호 불일치 |
| `ADMIN_017` | 403 | `ADMIN_ROLE_EDIT_FORBIDDEN` | 상급자·본인 티어의 권한/계정 편집 시도 |
| `ADMIN_018` | 400 | `ADMIN_PERM_NOT_EDITABLE` | 편집 불가 권한(`FIXED` 등) 편집 시도 |
| `ADMIN_019` | 400 | `ADMIN_PERM_DEPENDENCY` | WRITE/UNMASK 권한을 해당 READ 권한 없이 부여 시도 |
| `ADMIN_020` | 400 | `ADMIN_REFUND_AMOUNT_INVALID` | 환불 요청 금액이 남은 잔액을 초과(부분환불 §17) |
| `ADMIN_021` | 400 | `ADMIN_REFUND_NOT_ALLOWED` | 환불 불가 상태(이미 전액 환불됐거나 `PAID`/`PARTIALLY_REFUNDED` 아님) |
| `ADMIN_022` | 404 | `ADMIN_CONTENT_NOT_FOUND` | `/content/{id}` 모더레이션 대상 게시물 없음 |
| `ADMIN_023` | 400 | `ATTACHMENT_ASSOCIATION_FAILED` | 작성/수정(§39~§40) 시 `attachmentIds` 연관 확정 실패 — 첨부 부재·slug 불일치·비 ACTIVE·이미 타 게시물에 연관(탈취 시도) |
| `ADMIN_024` | 400 | `USER_ALREADY_DELETED` | 이미 탈퇴 처리된 사용자를 다시 삭제 시도(§43) |
| `ADMIN_025` | 410 | `USER_ERASED` | 유예 만료로 이미 완전삭제(익명화)된 사용자의 export·삭제 시도(§42~§43) |

### 공통 (`CommonError`, `common-web`)

| 코드 | HTTP | enum | 의미 |
|---|---|---|---|
| `CMN_001` | 422 | `VALIDATION_ERROR` | 입력값 검증에 실패했습니다 (bean validation — §17 `@Positive amount`·§18d·§41 의 422 가 전부 이 코드예요) |
| `CMN_004` | 401 | `UNAUTHORIZED` | 인증이 필요합니다 (토큰 없음/무효 — admin 토큰은 refresh가 없으니 만료되면 재로그인) |
| `CMN_005` | 403 | `FORBIDDEN` | 권한이 없습니다 (필요한 `PERM_*` 없음, 앱 유저 JWT 로 `/api/admin/**` 접근, 콘솔 JWT 로 `/api/apps/{slug}/**` 접근) |
| `CMN_006` | 500 | (서버 내부 오류) | catch-all — 처리 안 된 예외가 여기로 떨어져요 |

프론트 쪽 매핑은 `src/api/client.ts`의 `ApiRequestError.code`로 노출돼요(`body.error.code`). 코드별 분기 UI가 필요하면 이 값을 보고 처리하세요 — 현재 화면들은 대부분 `isError`만 보고 공통 `<Alert type="error">`로 뭉뚱그려 처리해요 (`docs/guide/screens.md` 참고).

---

## 시맨틱 노트

### gross / net 정의 (빌링)

`payment_history.status`는 환불 시 `PAID` → `REFUNDED`(전액) 또는 `PARTIALLY_REFUNDED`(부분, §17)로 **덮어써요** (별도 플래그가 아니라 상태 자체가 뒤집힘). 그래서:

- **`gross`(총매출) = `status IN ('PAID', 'REFUNDED', 'PARTIALLY_REFUNDED')`인 건의 `amount` 합** — 환불 여부와 무관하게 "한 번이라도 수금된 금액"의 총합이에요. `status='PAID'`로만 집계하면 환불된 결제가 gross에서도 빠져버리고, 이어서 `net = gross - refunded`로 또 한 번 차감돼 **이중차감** 버그가 생겨요. 부분환불 건도 전액이 한 번 수금됐으므로 gross엔 전액이 들어가요.
- **`refunded` = `payment_refunds` 원장(§18)에서 건별 `refunded_at`이 기간에 속하는 `amount` 합** — 부분환불은 `refunded_amount < amount`. `payment_history.amount`를 합하면 부분환불을 전액환불로 과다 집계하고, 누적 `refunded_amount`는 `refunded_at`이 마지막 값이라 다회 환불의 기간 귀속이 틀어져서, 반드시 원장 건별로 집계해요.
- **`net`(순매출) = `gross - refunded`**
- 대시보드(§4)·앱 metrics(§6)·billing(§7)·revenue 시계열(§13) 전부 이 시맨틱을 따라요.

### DAU / MAU — 실데이터 기반 (`user_activity_days`)

DAU/MAU는 더미가 아니라 **진짜 활동 기록**이에요. 백엔드에 앱 스키마별 `user_activity_days(user_id, activity_date)` 테이블이 있고, **인증된 API 요청을 가로채는 인터셉터**가 `(user_id, 오늘)`을 upsert(`ON CONFLICT DO NOTHING`)해요. "오늘"은 앱 서버의 로컬 시계가 아니라 **DB의 `CURRENT_DATE`**로 결정돼요 — 앱 서버와 DB 서버의 시계/타임존이 어긋나도 기록 시점과 집계 쿼리(`WHERE activity_date = CURRENT_DATE`)의 날짜 기준이 항상 일치하게 하기 위해서예요.

- **DAU** = 날짜별 distinct user 수
- **MAU** = 최근 30일 distinct user 수
- **추적 시작일 이전 구간은 데이터가 없어요** — 신규 지표라 코호트가 안 쌓인 경우와 마찬가지로, DAU 시계열 차트는 활동 추적을 시작한 날짜 이후 구간만 존재해요(`src/mocks/fixtures.ts`의 `TRACKING_START` 상수가 이 상태를 mock에서도 재현).
- `template-flutter`는 이 활동 신호를 위해 별도 코드 수정이 필요 없어요 — 부팅 시 device 등록/토큰 refresh가 이미 인증 API를 치기 때문에 그 요청 자체가 활동 신호가 돼요. (포그라운드 복귀 수준의 정밀 ping은 `POST /api/apps/{slug}/users/me/activity`로 별도 확장돼 있어요 — auth_kit 쪽 계약은 `template-flutter`의 `docs/api-contract/user-profile.md` 참고.)

### 리텐션 D1/D7 코호트

`ops`(§8)의 `retentionD1`/`retentionD7`는 코호트 리텐션 %예요.

- **코호트 정의**: 가입일 기준 상대일 — D1 코호트는 가입일이 `[-15, -2]`일 구간, D7 코호트는 `[-21, -8]`일 구간(오늘 기준). 즉 "정확히 지금으로부터 N일 전 가입자"가 아니라, "가입 후 N일째가 이미 지난" 유저들의 집합이에요.
- **생존 판정**: 코호트에 속한 유저가 `user_activity_days`에 `가입일 + N일` 행이 있으면 "생존"으로 카운트.
- **값**: 코호트 크기 대비 생존 비율 %, 소수 1자리. **코호트 크기가 0이면 `null`**(정상 케이스 — 신설 지표라 아직 데이터가 안 쌓인 상태를 뜻해요. `retentionD1`이 `null`이면 `retentionD7`도 항상 `null`이에요 — D7은 D1보다 더 늦게 채워지니까요).
- 화면(앱 상세 개요탭의 `OpsSignalCards`)은 `null`을 "— (데이터 수집 중)"으로 표시해요 — 에러가 아니라 정상 대기 상태로 다뤄야 해요.

### Mock ↔ 실서버 토글

`.env`의 `VITE_USE_MOCK`이 유일한 스위치예요.

- `true`(기본): `src/mocks/browser.ts`의 MSW 서비스워커가 브라우저에서 `/api/admin/*`를 가로채요. `BASE_URL`도 강제로 빈 문자열이라 항상 same-origin으로 요청이 나가고 MSW가 반드시 매치해요.
- `false`: MSW 미기동, `VITE_API_BASE`(또는 Vite dev proxy의 `VITE_PROXY_TARGET`)로 실제 백엔드에 요청해요.
- `./factory local start`는 `GET /api/admin/health`를 직접 curl로 프로브해서 이 값을 **자동** 결정해요 — 백엔드가 떠 있으면 실서버 모드, 없으면 mock 모드로 폴백하고 안내 메시지를 출력해요.
- 두 모드가 스키마 드리프트 없이 동일하게 동작하는 건 `src/lib/types.ts`를 mock/실서버가 공유하기 때문이에요. 다만 **예외 경로의 세부 동작은 다를 수 있어요** — 예를 들어 현재 mock 은 `metrics`/`billing`/`ops`의 슬러그 404에 `APP_404`, 사용자 404에 `USR_404`라는 자체 코드를 쓰는데, 실서버는 각각 `ADMIN_003`/`ADMIN_005`예요. 발송(§M1~§M6)은 아예 실서버에 없고요. 에러 코드로 분기하는 로직은 **실서버 코드(`ADMIN_*`) 기준으로 작성**하고, 에러 케이스는 mock만 보고 100% 신뢰하지 마세요.

---

## 관련 문서

- [`README.md`](./README.md) — 계약 문서 전체 구성 · 쌍 운영 규칙
- [`../guide/screens.md`](../guide/screens.md) — 화면별 엔드포인트 사용처
- [`짝 백엔드: template-spring`](https://github.com/storkspear/template-spring) — 백엔드 쪽 상세는 `docs/api-and-functional/admin-console.md`(§3 카탈로그가 이 문서와 같은 번호)
- 설계 스펙 원문: `template-spring` 저장소의 `docs/superpowers/specs/2026-07-06-admin-module-design.md`
