<a id="template-react-admin--docs"></a>

# template-react-admin — Docs

`template-react-admin`으로 관리자 콘솔을 만들고 운영하는 개발자를 위한 문서예요. 처음 시작할 때는 파생 레포 가이드를, 화면을 수정할 때는 화면 가이드를, 백엔드를 연결할 때는 API 계약을 읽어주세요.

이 콘솔은 [`template-spring`](https://github.com/storkspear/template-spring)의 `/api/admin/*`를 호출해요. 프론트와 백엔드를 함께 관리하면서 요청·응답 필드와 권한을 맞추는 구조예요.

## 시작하기

- [파생 레포 가이드](./guide/derived-repo.md): 설치, 브랜딩, 메뉴 구성, Mock·실서버 실행, 화면 추가
- [원본 README](../README.md): 빠른 시작과 소스 구조
- [작업 지침](../CLAUDE.md): 설정을 관리하는 파일과 개발 규칙

원본 저장소 링크는 해당 저장소에 접근할 수 있는 계정이 필요해요.

<a id="api-계약-api-contract"></a>

## API 계약 ([`api-contract/`](./api-contract/))

- [계약 관리](./api-contract/README.md): 변경할 파일과 검증 순서
- [관리자 API](./api-contract/admin-api.md): 요청·응답, 권한, 오류, 집계 기준, Mock과 실서버의 차이

<a id="화면-가이드-guidescreensmd"></a>

## 화면 가이드 ([`guide/screens.md`](./guide/screens.md))

[화면 가이드](./guide/screens.md)에서 화면별 기능과 API, 권한, 모바일 동작을 확인할 수 있어요.

<a id="파생-레포-guidederived-repomd"></a>

## 파생 레포 ([`guide/derived-repo.md`](./guide/derived-repo.md))

[파생 레포 가이드](./guide/derived-repo.md)는 새 저장소를 만든 뒤 필요한 설정을 순서대로 설명해요. 홈서버 배포는 [배포 가이드](./guide/deployment.md)에서 다뤄요.

<a id="레퍼런스-referencereference"></a>
<a id="레퍼런스-reference"></a>

## 레퍼런스 ([`reference/`](./reference/environment.md))

[개발 환경](./reference/environment.md)에 기술 스택 그림과 버전표, 라이브러리를 옮겨 쓸 때 확인할 사항을 모았어요. 설치 버전의 기준은 원본 저장소의 `package.json`과 lock 파일이에요.

## 문서 구조

| 문서 | 담당 내용 |
|---|---|
| `api-contract/` | 서버 계약과 변경 절차 |
| `guide/derived-repo.md` | 파생 앱 설정과 개발 |
| `guide/deployment.md` | 홈서버 배포와 정리 |
| `guide/screens.md` | 화면별 동작 |
| `reference/environment.md` | 개발 환경과 기술 스택 |

문장을 고칠 때는 [공통 글쓰기 기준](https://storkspear.github.io/docs-template-spring/#reference/STYLE_GUIDE.md)을 따라요. 설명과 코드가 다르면 현재 코드를 확인해 문서를 고치고, 설계 원칙을 바꿔야 하는 문제는 별도로 검토해요.
