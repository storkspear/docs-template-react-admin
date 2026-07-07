# Admin API — 엔드포인트 계약

`template-spring`의 `core/core-admin-impl` 모듈이 제공하는 `/api/admin/*` 전체 계약이에요. 프론트 타입은 `src/lib/types.ts`, 클라이언트 함수는 `src/api/client.ts`, mock 구현은 `src/mocks/handlers.ts` + `fixtures.ts`에 있어요.

**총 11개** — 데이터 엔드포인트 10개(아래 §1~§10) + `health` 프로브 전용 1개. `README.md`/`CLAUDE.md`가 "9개"라고 적어둔 건 `ops`(§10)와 `health`가 나중에 추가되기 전 기준이라 오래된 숫자예요 — 실제 계약은 여기 이 문서가 최신이에요.

> **인증 스코프**: admin 로그인은 앱 유저 인증과 완전히 분리돼요. 백엔드는 별도 `admin.admin_users` 스키마 + `role=superadmin` JWT claim을 쓰고, `/api/admin/**`는 `hasRole('SUPERADMIN')`만 허용해요. 앱 내부 `role='admin'`(`ROLE_ADMIN`) 유저는 이 콘솔에 들어올 수 없고, 반대로 superadmin 토큰으로 `/api/apps/{slug}/**`에 접근하면 기존 `AppSlugVerificationFilter`가 403을 내려요 — 양방향 격리예요.

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

목록 엔드포인트(`users`, `audit-logs`)는 `data` 안에 `PageResponse<T>`가 또 한 겹 있어요.

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

## [1] `POST /api/admin/auth/login`

관리자 로그인. **refresh token 없음** — access-token-only예요 (앱 유저 인증과 달리 회전/재발급 흐름이 없어요. 만료되면 재로그인).

- **Request body**: `{ email: string, password: string }`
- **Response** (`AdminLogin`):
  ```ts
  interface AdminAccount { userId: number; email: string; role: string; appSlug: string }
  interface AdminLogin { accessToken: string; refreshToken?: string; admin: AdminAccount }
  ```
  `refreshToken`은 타입상 optional이지만 백엔드가 실제로 채워주지 않아요(`AdminLoginResponse`에 필드 자체가 없음) — 항상 `undefined`라고 생각하면 돼요. `admin.appSlug`는 항상 `"admin"` 고정값(JWT의 `appSlug` claim이 non-blank를 요구해서 넣은 placeholder — 실제 앱 슬러그가 아니에요).
- **에러**: 자격증명 불일치 → `401 ADMIN_001`

> **Mock ↔ 실서버 필드 불일치 주의**: 현재 `src/mocks/handlers.ts`는 로그인 실패 시 `ATH_001`을 반환하는데(앱 유저 인증 코드를 복사한 흔적), 실제 백엔드는 `ADMIN_001`이에요. 에러 코드로 분기하는 화면 로직을 짠다면 **`ADMIN_001` 기준으로 작성**하고, mock 쪽 불일치는 알고 있는 상태로 두거나 고쳐서 맞추세요.

## [2] `GET /api/admin/health`

**공개 엔드포인트** (인증 불필요). React `factory` CLI가 로컬 백엔드 연결 여부를 확인하는 용도로만 써요 — `src/api/client.ts`엔 대응 함수가 없고(데이터 계약이 아니라 인프라 프로브라서), `factory` 셸 스크립트가 직접 `curl -sf`로 호출해요.

```bash
curl -sf -m 3 -o /dev/null "$target/api/admin/health"
```

`local start`가 이 요청 성공 여부로 `VITE_USE_MOCK` 값을 자동 결정해요 (§ "Mock ↔ 실서버 전환" 참고).

## [3] `GET /api/admin/apps`

등록된 전체 앱(슬러그) 요약.

- **Response**: `AppSummary[]`
  ```ts
  interface AppSummary { slug: string; userCount: number; activeSubscriptions: number }
  ```
- **데이터 소스**: 슬러그 열거 + 각 앱 스키마의 `users` count · `subscriptions`(ACTIVE) count.

## [4] `GET /api/admin/dashboard/metrics`

전사(cross-app) 합산 지표.

- **Query**: `window` (기본 `30d`) — 현재 백엔드 구현은 값 자체보다 라벨 용도(윈도우별 재계산 로직은 고정 30일 집계 위주)
- **Response** (`DashboardMetrics`):
  ```ts
  interface DashboardTotals {
    users: number; newUsers: number; dau: number; mau: number
    revenue: number; refunded: number; activeSubscriptions: number; failures24h: number
  }
  interface PerSlugMetrics extends DashboardTotals { slug: string }  // 형태만 동일, 실제론 별도 인터페이스
  interface DashboardMetrics {
    generatedAt: string; window: string
    totals: DashboardTotals; perSlug: PerSlugMetrics[]
  }
  ```
- **데이터 소스**: 전 슬러그 fan-out(슬러그별 DataSource 순회) 후 메모리 합산 — users·신규(`created_at`)·DAU/MAU(§ DAU/MAU 정의)·매출/환불(`payment_history`)·활성구독·`failures24h`(`audit_logs WHERE result='FAILURE'` 24시간 이내 count).
- **에러**: 없음(전사 지표라 slug 파라미터가 없어서 `ADMIN_003` 대상이 아님).

## [5] `GET /api/admin/apps/{slug}/metrics`

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

## [6] `GET /api/admin/apps/{slug}/users`

앱 사용자 목록 검색 + 페이지네이션.

- **Query**: `query`(이메일·표시이름·닉네임 ILIKE), `page`(기본 0), `size`(기본 20, **서버가 1~100으로 clamp** — 컨트롤러에서 `Math.min(Math.max(size, 1), 100)`)
- **Response**: `PageResponse<AdminUser>`
  ```ts
  interface AdminUser {
    id: number; email: string; displayName: string | null; nickname: string | null
    role: string; isPremium: boolean; emailVerified: boolean
    createdAt: string; deletedAt: string | null
  }
  ```
- **에러**: 알 수 없는 슬러그 → `404 ADMIN_003`

## [7] `GET /api/admin/apps/{slug}/users/{userId}`

사용자 상세 — 기기·구독·최근 결제 포함.

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
    status: string // READY | PAID | FAILED | CANCELLED | VBANK_ISSUED
    paidAt: string | null; refundedAt: string | null
  }
  interface AdminUserDetail {
    user: AdminUserFull; devices: AdminDevice[]
    subscriptions: AdminSubscription[]; recentPayments: AdminPayment[]  // 최근 10건
  }
  ```
- **에러**: 사용자 없음 → `404 ADMIN_005`. 슬러그 자체가 없으면 → `404 ADMIN_003`(슬러그 해석이 먼저 일어남).

## [8] `GET /api/admin/apps/{slug}/billing`

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

## [9] `GET /api/admin/audit-logs`

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

## [10] `GET /api/admin/analytics/{metric}`

시계열 차트 데이터.

- **Path**: `metric` = `dau` | `signups` | `revenue` (백엔드 `switch`문 — 그 외 값은 `400 ADMIN_002`)
- **Query**: `slug`(**필수**), `from`, `to` (생략 시 최근 30일). **`interval`은 요청에 안 쓰여요** — 응답의 `interval` 필드는 항상 `"day"`로 고정 반환돼요(백엔드가 파라미터를 안 읽음). 프론트 클라이언트(`getAnalytics`)가 `interval` 옵션을 넘겨도 무시돼요.
- **Response** (`TimeSeries`):
  ```ts
  interface TimeSeriesPoint { ts: string; value: number }
  interface TimeSeries { metric: string; interval: string; points: TimeSeriesPoint[] }
  ```
- **⚠️ 알려진 문제 — `slug` 누락 시 500**: `slug`는 컨트롤러 파라미터에 `required=false`가 없어 Spring 기본값(`required=true`)이 적용돼요. 그런데 이 예외(`MissingServletRequestParameterException`)를 잡는 핸들러가 없어서 **깔끔한 400이 아니라 catch-all → `500 CMN_006`**으로 떨어져요. 프론트는 `useAppOptions()`의 `firstSlug`가 로드되기 전엔 쿼리를 `enabled: !!slug`로 막아서 이 경로를 실전에서 밟지 않지만, 직접 API를 호출할 땐 `slug` 누락에 주의하세요. (mock 핸들러는 이 케이스를 `400 VAL_001`로 방어하는데, 실제 백엔드 동작과 다르니 혼동하지 마세요.)
- **데이터 소스**: `signups`=`users.created_at` 일별 집계, `revenue`=`payment_history.paid_at` 일별 집계(§ gross/net과 동일 시맨틱), `dau`=`user_activity_days`(§ DAU/MAU 정의).

## [11] `GET /api/admin/apps/{slug}/ops`

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

## 에러 코드

### Admin 전용 (`AdminError`, 백엔드 `core-admin-impl`)

| 코드 | HTTP | enum | 의미 |
|---|---|---|---|
| `ADMIN_001` | 401 | `INVALID_CREDENTIALS` | 이메일 또는 비밀번호가 올바르지 않아요 (로그인 실패) |
| `ADMIN_002` | 400 | `UNSUPPORTED_METRIC` | `analytics/{metric}`의 `metric`이 `dau`/`signups`/`revenue` 중 하나가 아님 |
| `ADMIN_003` | 404 | `UNKNOWN_SLUG` | 슬러그를 찾을 수 없음 — users/metrics/billing/audit/dashboard/analytics 등 슬러그를 받는 **모든** admin 엔드포인트 공통 |
| `ADMIN_004` | 400 | `INVALID_DATE_RANGE` | `from`/`to`가 ISO-8601 형식이 아님 (billing/audit-logs/analytics) |
| `ADMIN_005` | 404 | `USER_NOT_FOUND` | 사용자 상세 조회 시 해당 `userId` 없음 |

### 공통 (`CommonError`, `common-web`)

| 코드 | HTTP | enum | 의미 |
|---|---|---|---|
| `CMN_004` | 401 | `UNAUTHORIZED` | 인증이 필요합니다 (토큰 없음/무효 — admin 토큰은 refresh가 없으니 만료되면 재로그인) |
| `CMN_005` | 403 | `FORBIDDEN` | 권한이 없습니다 (예: 앱 `role='admin'` 유저가 `/api/admin/**` 접근 시도, 또는 superadmin 토큰으로 `/api/apps/{slug}/**` 접근 시도) |
| `CMN_006` | 500 | (서버 내부 오류) | catch-all — 위 "analytics `slug` 누락" 같은 처리 안 된 예외가 여기로 떨어져요 |

프론트 쪽 매핑은 `src/api/client.ts`의 `ApiRequestError.code`로 노출돼요(`body.error.code`). 코드별 분기 UI가 필요하면 이 값을 보고 처리하세요 — 현재 화면들은 대부분 `isError`만 보고 공통 `<Alert type="error">`로 뭉뚱그려 처리해요 (`docs/guide/screens.md` 참고).

---

## 시맨틱 노트

### gross / net 정의 (빌링)

`payment_history.status`는 결제 엔티티의 `markRefunded()`가 환불 시 `PAID` → `REFUNDED`로 **덮어써요** (별도 플래그가 아니라 상태 자체가 뒤집힘). 그래서:

- **`gross`(총매출) = `status IN ('PAID', 'REFUNDED')`인 건의 합** — 환불 여부와 무관하게 "한 번이라도 수금된 금액"의 총합이에요. `status='PAID'`로만 집계하면 환불된 결제가 gross에서도 빠져버리고, 이어서 `net = gross - refunded`로 또 한 번 차감돼 **이중차감** 버그가 생겨요 — 이 버그를 피하려고 `REFUNDED`도 gross에 포함시켜요.
- **`net`(순매출) = `gross - refunded`**
- 대시보드(§4)·앱 metrics(§5)·billing(§8)·revenue 시계열(§10) 전부 이 시맨틱을 따라요.

### DAU / MAU — 실데이터 기반 (`user_activity_days`)

DAU/MAU는 더미가 아니라 **진짜 활동 기록**이에요. 백엔드에 앱 스키마별 `user_activity_days(user_id, activity_date)` 테이블이 있고, **인증된 API 요청을 가로채는 인터셉터**가 `(user_id, 오늘)`을 upsert(`ON CONFLICT DO NOTHING`)해요. "오늘"은 앱 서버의 로컬 시계가 아니라 **DB의 `CURRENT_DATE`**로 결정돼요 — 앱 서버와 DB 서버의 시계/타임존이 어긋나도 기록 시점과 집계 쿼리(`WHERE activity_date = CURRENT_DATE`)의 날짜 기준이 항상 일치하게 하기 위해서예요.

- **DAU** = 날짜별 distinct user 수
- **MAU** = 최근 30일 distinct user 수
- **추적 시작일 이전 구간은 데이터가 없어요** — 신규 지표라 코호트가 안 쌓인 경우와 마찬가지로, DAU 시계열 차트는 활동 추적을 시작한 날짜 이후 구간만 존재해요(`src/mocks/fixtures.ts`의 `TRACKING_START` 상수가 이 상태를 mock에서도 재현).
- `template-flutter`는 이 활동 신호를 위해 별도 코드 수정이 필요 없어요 — 부팅 시 device 등록/토큰 refresh가 이미 인증 API를 치기 때문에 그 요청 자체가 활동 신호가 돼요. (포그라운드 복귀 수준의 정밀 ping은 `POST /api/apps/{slug}/users/me/activity`로 별도 확장돼 있어요 — auth_kit 쪽 계약은 `template-flutter`의 `docs/api-contract/user-profile.md` 참고.)

### 리텐션 D1/D7 코호트

`ops`(§11)의 `retentionD1`/`retentionD7`는 코호트 리텐션 %예요.

- **코호트 정의**: 가입일 기준 상대일 — D1 코호트는 가입일이 `[-15, -2]`일 구간, D7 코호트는 `[-21, -8]`일 구간(오늘 기준). 즉 "정확히 지금으로부터 N일 전 가입자"가 아니라, "가입 후 N일째가 이미 지난" 유저들의 집합이에요.
- **생존 판정**: 코호트에 속한 유저가 `user_activity_days`에 `가입일 + N일` 행이 있으면 "생존"으로 카운트.
- **값**: 코호트 크기 대비 생존 비율 %, 소수 1자리. **코호트 크기가 0이면 `null`**(정상 케이스 — 신설 지표라 아직 데이터가 안 쌓인 상태를 뜻해요. `retentionD1`이 `null`이면 `retentionD7`도 항상 `null`이에요 — D7은 D1보다 더 늦게 채워지니까요).
- 화면(`AnalyticsPage`)은 `null`을 "— (데이터 수집 중)"으로 표시해요 — 에러가 아니라 정상 대기 상태로 다뤄야 해요.

### Mock ↔ 실서버 토글

`.env`의 `VITE_USE_MOCK`이 유일한 스위치예요.

- `true`(기본): `src/mocks/browser.ts`의 MSW 서비스워커가 브라우저에서 `/api/admin/*`를 가로채요. `BASE_URL`도 강제로 빈 문자열이라 항상 same-origin으로 요청이 나가고 MSW가 반드시 매치해요.
- `false`: MSW 미기동, `VITE_API_BASE`(또는 Vite dev proxy의 `VITE_PROXY_TARGET`)로 실제 백엔드에 요청해요.
- `./factory local start`는 `GET /api/admin/health`를 직접 curl로 프로브해서 이 값을 **자동** 결정해요 — 백엔드가 떠 있으면 실서버 모드, 없으면 mock 모드로 폴백하고 안내 메시지를 출력해요.
- 두 모드가 스키마 드리프트 없이 동일하게 동작하는 건 `src/lib/types.ts`를 mock/실서버가 공유하기 때문이에요. 다만 위에서 짚은 에러 코드 불일치(`ATH_001` vs `ADMIN_001`, analytics `slug` 누락 시 mock=400/실서버=500)처럼 **응답 shape은 같아도 예외 경로의 세부 동작은 다를 수 있어요** — 에러 케이스는 mock만 보고 100% 신뢰하지 마세요.

---

## 관련 문서

- [`README.md`](./README.md) — 계약 문서 전체 구성 · 쌍 운영 규칙
- [`../guide/screens.md`](../guide/screens.md) — 화면별 엔드포인트 사용처
- [`짝 백엔드: template-spring`](https://github.com/storkspear/template-spring)
- 설계 스펙 원문: `template-spring` 저장소의 `docs/superpowers/specs/2026-07-06-admin-module-design.md`
