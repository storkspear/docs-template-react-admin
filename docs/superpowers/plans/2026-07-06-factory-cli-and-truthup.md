# factory CLI + 데이터 진실화 구현 플랜 (template-react-admin)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** ① factory CLI(install/start/test — 백엔드 프로브 후 실서버/mock 자동 선택) ② mock 데이터 진실화(스펙 §G — DAU 시계열 시작일·failures24h 정의·로그인 shape) ③ 실서버 연동 배관(Vite proxy — CORS 회피). 스펙: `template-spring/docs/superpowers/specs/2026-07-06-admin-module-design.md`

**Architecture:** dev 는 Vite proxy(`/api` → `localhost:8080`)로 same-origin 처리 (백엔드 CORS 수정 불필요). factory start 가 `GET /api/admin/health` 프로브 → 성공 시 `VITE_USE_MOCK=false`, 실패 시 mock + 안내. mock fixtures 는 백엔드 계약의 레퍼런스 역할이므로 실제 조회 시맨틱과 정합하게 수정.

**Tech Stack:** bash(factory) · Vite 8 proxy · MSW

## Global Constraints

- 커밋 트레일러(Co-Authored-By) 금지 · push 는 사용자 신호 시에만
- `npm run build` 그린 후에만 커밋
- 계약(types.ts) 필드명 변경 금지 — DAU/MAU 는 백엔드가 진짜로 구현하므로 이름 유지 (스펙 §G/H)

---

### Task 1: .env.example + Vite proxy + mock 토글 정리

**Files:**
- Create: `.env.example`
- Modify: `vite.config.ts` (server.proxy 추가)

**Interfaces:**
- Produces: env 계약 — `VITE_USE_MOCK`(기본 true), `VITE_PROXY_TARGET`(기본 `http://localhost:8080`). `VITE_API_BASE` 는 빈 값 유지(프록시로 same-origin) — `src/api/client.ts` 수정 불필요

- [ ] **Step 1: `.env.example` 생성**

```bash
# mock 모드 (기본 true) — factory start 가 백엔드 프로브 결과로 자동 결정
VITE_USE_MOCK=true

# 실서버 모드에서 Vite dev proxy 가 /api 를 전달할 백엔드 주소
VITE_PROXY_TARGET=http://localhost:8080

# 프록시를 안 쓰고 절대 URL 로 직접 칠 때만 설정 (CORS 는 백엔드 책임이 됨)
# VITE_API_BASE=
```

- [ ] **Step 2: `vite.config.ts` 에 proxy 추가** — 기존 defineConfig 에 server 블록 병합 (기존 내용은 유지):

```ts
import { defineConfig, loadEnv } from 'vite'

export default defineConfig(({ mode }) => {
  const env = loadEnv(mode, process.cwd(), '')
  return {
    // ...기존 plugins 등 유지...
    server: {
      proxy: {
        // 실서버 모드: /api → 백엔드 (same-origin 이 되어 CORS 불필요).
        // mock 모드에선 MSW 서비스워커가 먼저 가로채므로 이 프록시는 무해.
        '/api': {
          target: env.VITE_PROXY_TARGET || 'http://localhost:8080',
          changeOrigin: true,
        },
      },
    },
  }
})
```

- [ ] **Step 3: 빌드 확인 + 커밋**

Run: `npm run build` → 그린. mock 모드 동작 확인: `npm run dev` 후 `curl -s http://localhost:5173 -o /dev/null -w "%{http_code}"` → 200

```bash
git add .env.example vite.config.ts
git commit -m "feat(admin): .env.example + Vite /api proxy (실서버 same-origin)"
```

---

### Task 2: factory CLI (install / start / test — 백엔드 프로브)

**Files:**
- Create: `factory` (실행 bash, `chmod +x`)
- Modify: `README.md` (빠른 시작에 factory 사용법 추가)

**Interfaces:**
- Produces: `./factory install [--symlink=<alias>]`, `<repo> start`, `<repo> test [--no-backend]`
- 프로브 계약: `GET {VITE_PROXY_TARGET}/api/admin/health` — 2xx 면 백엔드 살아있음 (template-spring 플랜 Task 3 의 public 엔드포인트)

- [ ] **Step 1: `factory` 스크립트 작성**

```bash
#!/usr/bin/env bash
# factory — template-react-admin 통합 dispatcher (template-flutter/spring 과 동일 UX)
set -euo pipefail

REPO_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
REPO_NAME="$(basename "$REPO_DIR")"

# .env 로드 (없으면 기본값)
[ -f "$REPO_DIR/.env" ] && set -a && source "$REPO_DIR/.env" && set +a
PROXY_TARGET="${VITE_PROXY_TARGET:-http://localhost:8080}"

usage() {
  cat <<EOF
사용법: $REPO_NAME <command>

  install [--symlink=<alias>]  ~/.local/bin 에 symlink 등록
  start                        백엔드 프로브 후 dev 서버 기동 (성공: 실서버 / 실패: mock)
  start --mock                 프로브 없이 mock 강제
  test [--no-backend]          build + (기본) 백엔드 ping
EOF
}

probe_backend() {
  curl -sf -m 3 -o /dev/null "$PROXY_TARGET/api/admin/health"
}

cmd="${1:-}"; shift || true
case "$cmd" in
  install)
    alias_name="$REPO_NAME"
    for a in "$@"; do [[ "$a" == --symlink=* ]] && alias_name="${a#--symlink=}"; done
    mkdir -p "$HOME/.local/bin"
    ln -sf "$REPO_DIR/factory" "$HOME/.local/bin/$alias_name"
    echo "✓ $alias_name → $REPO_DIR/factory"
    ;;
  start)
    if [[ "${1:-}" == "--mock" ]]; then
      echo "→ mock 모드 (강제)"
      VITE_USE_MOCK=true npm --prefix "$REPO_DIR" run dev
    elif probe_backend; then
      echo "✓ 백엔드 연결 확인 ($PROXY_TARGET) — 실서버 모드로 시작합니다."
      VITE_USE_MOCK=false npm --prefix "$REPO_DIR" run dev
    else
      echo "✗ 백엔드 미연결 ($PROXY_TARGET/api/admin/health)"
      echo "  백엔드 서버가 필요하면 template-spring 을 먼저 실행하세요."
      echo "→ mock 데이터로 실행합니다. (헤더의 MOCK 배지로 상태 표시)"
      VITE_USE_MOCK=true npm --prefix "$REPO_DIR" run dev
    fi
    ;;
  test)
    npm --prefix "$REPO_DIR" run build
    if [[ "${1:-}" != "--no-backend" ]]; then
      if probe_backend; then echo "✓ backend ping OK"; else echo "✗ backend ping FAIL (--no-backend 로 생략 가능)"; exit 1; fi
    fi
    echo "✓ test 통과"
    ;;
  *) usage; exit 1 ;;
esac
```

- [ ] **Step 2: 실행 검증**

```bash
chmod +x factory
./factory install
./factory test --no-backend
```
Expected: symlink 생성 메시지 + build 그린 + "✓ test 통과". `./factory start` 는 백엔드 없을 때 "→ mock 데이터로 실행합니다." 출력 후 dev 서버 기동 (Ctrl-C 종료).

- [ ] **Step 3: README 갱신 + 커밋** — README "빠른 시작" 에 `./factory install` → `<repo> start` 흐름 3줄 추가.

```bash
git add factory README.md
git commit -m "feat(admin): factory CLI — 백엔드 프로브 후 실서버/mock 자동 선택"
```

---

### Task 3: mock 진실화 (스펙 §G)

**Files:**
- Modify: `src/mocks/fixtures.ts`
- Modify: `src/mocks/handlers.ts` (필요 시 — 로그인 응답 shape 확인)

**Interfaces:**
- 진실 계약 (백엔드 플랜과 동일 시맨틱):
  - 로그인 응답 `admin` = `{userId, email, role: 'superadmin', appSlug: 'admin'}`
  - `dau` 시계열: **추적 시작일(TRACKING_START, 오늘-14일) 이전 포인트 없음**
  - dashboard `failures24h` = 해당 슬러그 audit fixtures 중 `result==='FAILURE'` && 24h 이내 건수

- [ ] **Step 1: 현재 fixtures 확인** — `src/mocks/fixtures.ts` 를 열어 ① login 이 반환하는 admin 객체 필드 ② `analytics(metric, interval)` 의 dau 분기 ③ dashboard perSlug 의 failures24h 값 생성부를 찾는다.

- [ ] **Step 2: dau 시계열 시작일 반영** — analytics dau 분기를 다음 시맨틱으로 교체 (기존 값 생성 로직은 유지하되 시작일 필터만 추가):

```ts
/** 활동 추적 시작일 — 백엔드 user_activity_days 도입 시점을 모사 (이전 날짜 포인트 없음). */
const TRACKING_START = dayjs().subtract(14, 'day').startOf('day')

// dau 분기: 기존 30일 시리즈 생성 후
points = points.filter((p) => !dayjs(p.ts).isBefore(TRACKING_START))
```

- [ ] **Step 3: failures24h 정합** — dashboard fixtures 의 perSlug.failures24h 를 임의 숫자 대신 audit fixtures 파생으로:

```ts
const failures24h = (slug: string) =>
  auditLogs.filter(
    (l) => l.slug === slug && l.result === 'FAILURE' && dayjs(l.occurredAt).isAfter(dayjs().subtract(24, 'hour')),
  ).length
```
(totals.failures24h = perSlug 합)

- [ ] **Step 4: 로그인 shape 검증** — login 응답의 admin 이 `{userId, email, role, appSlug}` 와 다르면 (`id` 로 돼 있는 등) `role: 'superadmin'`, `appSlug: 'admin'` 으로 맞춘다. types.ts 의 `AdminAccount` 가 이미 이 4필드이므로 **types.ts 는 수정 금지**.

- [ ] **Step 5: 빌드 + mock 화면 확인 + 커밋**

Run: `npm run build` → 그린. `npm run dev` 로 대시보드·분석 페이지가 정상 렌더(분석 dau 차트가 14일 구간만)인지 확인.

```bash
git add src/mocks
git commit -m "fix(admin): mock 진실화 — dau 추적시작일 + failures24h 정의 정합"
```

---

### Task 4: 분석 차트 "데이터 시작일" 표기

**Files:**
- Modify: `src/pages/AnalyticsPage.tsx` (AnalyticsChart)

- [ ] **Step 1: AnalyticsChart 에 캡션 추가** — Card 에 extra 로 첫 포인트 날짜 표시 (시리즈가 요청 구간보다 늦게 시작할 때 사용자가 "왜 앞이 비었지" 하지 않게):

```tsx
function AnalyticsChart({ slug, def }: { slug: string; def: ChartDef }) {
  const { data, isLoading } = useQuery({
    queryKey: ['analytics', slug, def.metric],
    queryFn: () => getAnalytics(def.metric, { interval: 'day' }),
    enabled: !!slug,
  })
  const series = data?.points.map((p) => ({ label: date(p.ts), value: p.value })) ?? []
  const firstTs = data?.points[0]?.ts
  return (
    <Card
      size="small"
      title={def.label}
      loading={isLoading}
      extra={
        firstTs ? (
          <Typography.Text type="secondary" style={{ fontSize: 12 }}>
            {date(firstTs)}부터
          </Typography.Text>
        ) : undefined
      }
    >
      <MiniBarChart data={series} color={def.color} format={def.fmt} />
    </Card>
  )
}
```
(`Typography` 는 이미 import 되어 있음.)

- [ ] **Step 2: 빌드 + 커밋**

Run: `npm run build` → 그린.
```bash
git add src/pages/AnalyticsPage.tsx
git commit -m "feat(admin): 분석 차트 데이터 시작일 캡션"
```

---

### Task 5: 통합 검증 (백엔드 실서버 연결)

전제: template-spring 플랜 전체 완료 + 로컬 기동 (`ADMIN_EMAIL/ADMIN_PASSWORD` env 시드 포함).

- [ ] **Step 1**: template-spring 기동 → `curl -s http://localhost:8080/api/admin/health` → `{"data":{"status":"UP"},...}`
- [ ] **Step 2**: `./factory start` → "✓ 백엔드 연결 확인 — 실서버 모드" 출력 확인, 헤더에 MOCK 배지 **없음** 확인
- [ ] **Step 3**: 로그인 (ADMIN_EMAIL 계정) → 대시보드·앱·사용자·분석·감사로그 각 화면이 실데이터로 렌더 (Chrome 연결 시 스크린샷 검증)
- [ ] **Step 4**: 백엔드 끄고 `./factory start` → "→ mock 데이터로 실행합니다" + MOCK 배지 확인
- [ ] **Step 5**: 권한 경계 (스펙 §F) — 앱 유저 토큰으로 `/api/admin/apps` curl → **403** / superadmin 토큰으로 `/api/apps/{slug}/*` → **403** 확인
- [ ] **Step 6**: 발견 이슈를 fix 커밋으로 처리 후 종합 보고
