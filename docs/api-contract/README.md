# API Contract

`template-react-admin`과 짝 백엔드 [`template-spring`](https://github.com/storkspear/template-spring)의 admin 모듈(`core/core-admin-impl`, `/api/admin/*`) 간 **1:1 계약** 정리예요. 응답 스키마·에러 코드가 양쪽에서 **완전 동일**해요.

> **왜 1:1?** `template-flutter` ↔ `template-spring`과 같은 이유예요 — 같은 개발자가 프론트와 백엔드를 함께 운영하는 앱 공장 전제라, Mapper 층을 두지 않고 DTO 필드명을 그대로 맞춰요. 백엔드 DTO(`AdminAccountResponse` 등)의 Javadoc에도 "필드명은 React `types.ts` 계약과 1:1(변경 금지)"라고 명시돼 있어요.

---

## 문서 구성

| 파일 | 내용 |
|------|------|
| [`admin-api.md`](./admin-api.md) | 엔드포인트 전체(백엔드 매핑 전부 + mock 전용 발송 — 개수·목록은 그 문서 서두의 실측 기준) — 요청 파라미터 · 응답 shape · 에러 코드 · 시맨틱 노트(gross/net, DAU/MAU, 리텐션) |

이 템플릿은 `template-flutter`처럼 도메인별로 문서를 쪼개지 않아요. Admin 콘솔은 **단일 계약 표면**(`/api/admin/*` 하나)이라 `admin-api.md` 한 파일에 전부 담겨 있어요.

> **각주(후속 사이클 안내)**: 게시물 본문 계약(markdown + `![alt](attachment://{id})` 참조 — `admin-api.md` § "본문 계약")은 관리자 콘솔 전용이 아니라, 후속 사이클에서 `template-flutter` 앱의 게시물 렌더에도 동일하게 적용될 예정이에요.

---

## 진실의 출처

| 계층 | 파일 |
|------|------|
| 프론트 DTO (계약) | `src/lib/types.ts` |
| 프론트 API 클라이언트 | `src/api/client.ts` |
| Mock 구현 (개발용) | `src/mocks/handlers.ts`, `src/mocks/fixtures.ts` |
| 백엔드 컨트롤러 | `template-spring`의 `core/core-admin-impl/.../controller/` |
| 백엔드 에러 코드 | `AdminError.java`(`ADMIN_001`~`ADMIN_023` — 현행 전체 표는 [`admin-api.md`](./admin-api.md) "에러 코드" 절), `CommonError.java`(`CMN_004`/`CMN_005`/`CMN_006`) |
| 설계 스펙 원문 | `template-spring` 저장소의 `docs/superpowers/specs/2026-07-06-admin-module-design.md` |

`src/lib/types.ts`의 인터페이스가 **프론트 쪽 계약의 진실**이에요 — mock과 실서버가 같은 타입을 공유해서 스키마 드리프트가 안 나요(`CLAUDE.md` §3 참고).

---

## 쌍 운영 규칙

### 서버 변경 시

1. 백엔드의 `AdminError.java`·컨트롤러 응답 DTO 수정
2. 프론트의 `src/lib/types.ts`·`src/api/client.ts` **동시** 수정, `src/mocks/handlers.ts`/`fixtures.ts`도 같이 갱신(mock이 실 계약을 그대로 반영해야 하므로)
3. 두 레포에 **같은 커밋 메시지**로 PR
4. `npm run build`(`tsc -b`)가 타입 불일치를 잡아줘요. 런타임 계약 검증(mock ↔ 실서버 응답 shape 비교)은 아직 자동화가 없어요 — `VITE_USE_MOCK=false`로 붙여서 수동 확인하세요.

### 프론트 변경 시

- 프론트 단독으로 계약을 바꾸지 **마세요**. 백엔드에 요청 먼저.
- 필드 추가·삭제는 반드시 백엔드 리드.

---

## 계약 변경 시 확인 포인트

- [ ] 양쪽 레포의 DTO/타입 파일 수정 확인 (`types.ts` ↔ 백엔드 `dto` 패키지)
- [ ] `src/mocks/fixtures.ts`가 새 필드를 반영하는지(안 하면 mock 모드에서만 조용히 undefined)
- [ ] 기존 화면 호환 (하위 호환 필드 추가 우선)
- [ ] `npm run build` 그린
- [ ] 문서 업데이트 (이 폴더)

---

## 관련 문서

- [`admin-api.md`](./admin-api.md) — 엔드포인트 전체 계약
- [`../guide/screens.md`](../guide/screens.md) — 화면이 각 엔드포인트를 어떻게 쓰는지
- [`짝 백엔드: template-spring`](https://github.com/storkspear/template-spring)
- 설계 스펙: `template-spring` 저장소의 `docs/superpowers/specs/2026-07-06-admin-module-design.md` (링크는 해당 브랜치가 `main`에 병합된 뒤 추가 예정)
