# template-react-admin — Docs

이 폴더의 문서는 `template-react-admin`을 기반으로 **파생 레포를 만들고 운영하는 개발자**를 위한 가이드예요. 레포 최상위의 [`README.md`](../README.md)와 [`CLAUDE.md`](../CLAUDE.md)가 이미 빠른 시작·단일 소스·컨벤션을 충실히 담고 있어서, 여기 `docs/`는 그걸 반복하지 않고 **백엔드 계약**과 **화면별 상세**처럼 더 깊게 들어가야 하는 내용만 다뤄요.

> **짝이 되는 백엔드**: 이 프론트엔드는 [`template-spring`](https://github.com/storkspear/template-spring)의 admin 모듈(`/api/admin/*`)을 소비해요. `template-flutter` ↔ `template-spring`과 같은 "앱 공장" 3종 세트 중 하나로, 같은 솔로 개발자가 세 레포를 함께 운영하는 걸 전제해요.

---

## 시작하기

레포를 처음 만났다면 이 순서로 읽어주세요.

- [`README.md`](../README.md) — 스택 소개 · 빠른 시작(`npm install`/`factory` CLI) · Mock 로그인 · 화면↔엔드포인트 매핑 · 폴더 구조
- [`CLAUDE.md`](../CLAUDE.md) — Claude Code 에이전트용 요약이지만 사람이 읽어도 좋은 5분 요약: 단일 소스 표(`appConfig.ts`/`palette.ts`/`nav.tsx` 등) · 자주 하는 작업 · 컨벤션

이 두 문서에 이미 있는 내용(스택·명령어·단일 소스 표)은 여기서 다시 안 다뤄요.

---

## API 계약 ([`api-contract/`](./api-contract/))

`template-spring`과의 1:1 계약이에요. 응답 스키마·에러 코드가 어긋나면 mock과 실서버 양쪽이 동시에 안 맞아요.

- [`api-contract/README.md`](./api-contract/README.md) — 계약 문서 구성 · 쌍 운영 규칙 · 진실의 출처
- [`api-contract/admin-api.md`](./api-contract/admin-api.md) — 엔드포인트 전체(백엔드 매핑 전부 + mock 전용 발송 — 정확한 개수·목록은 그 문서 서두의 실측 기준): 요청 파라미터 · 응답 shape · 에러 코드(`ADMIN_001`~`ADMIN_025`, `CMN_001`/`CMN_004`/`CMN_005`/`CMN_006`) · 시맨틱 노트(gross/net, DAU/MAU, 리텐션 D1/D7, mock↔실서버 토글)

## 화면 가이드 ([`guide/screens.md`](./guide/screens.md))

전 화면(정확한 목록·개수는 `guide/screens.md` 서두 기준) 각각이 무엇을 보여주고 어떤 API를 쓰는지, 모바일에서 어떻게 달라지는지(무한스크롤·필터 모달) 정리했어요.

- [`guide/screens.md`](./guide/screens.md)

## 파생 레포 ([`guide/derived-repo.md`](./guide/derived-repo.md))

"Use this template" 직후부터 실서버 연결까지, `CLAUDE.md` §8 체크리스트를 단계별 상세 + 코드 예시로 풀어썼어요.

- [`guide/derived-repo.md`](./guide/derived-repo.md) — factory CLI 흐름 · 브랜딩(appConfig/palette/favicon) · 메뉴 구성(nav.tsx) · Mock→실서버 전환 · 화면 추가 패턴(그리드형/폼형)

## 레퍼런스 ([`reference/`](./reference/))

기술 스택 인벤토리예요. 루트 README 의 스택 표를 docs 뷰어에서도 볼 수 있게 버전표와 다이어그램으로 정리했어요.

- [`reference/environment.md`](./reference/environment.md) — 프레임워크·라이브러리 버전표 (정본은 `package.json`)
- [`reference/environment-ui.md`](./reference/environment-ui.md) — 기술 스택 한눈 다이어그램 (docs 뷰어에서 렌더링)

---

## 문서 구조

```
docs/
├── README.md                  # 이 파일 — 목차
├── api-contract/
│   ├── README.md               # 계약 구성 · 쌍 운영 규칙
│   └── admin-api.md            # 엔드포인트 전체 상세 계약
├── guide/
│   ├── screens.md               # 화면 가이드 (전 화면)
│   └── derived-repo.md          # 파생 레포 체크리스트 심화판
└── reference/
    ├── environment.md            # 기술 스택 버전표
    └── environment-ui.md         # 기술 스택 다이어그램
```
