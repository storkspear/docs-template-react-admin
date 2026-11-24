# API Contract

`template-react-admin`과 [`template-spring`](https://github.com/storkspear/template-spring)의 관리자 API(`/api/admin/*`) 사이의 계약을 정리했어요. DTO 필드명을 양쪽에서 맞춰 사용하고, 별도 변환 계층을 두지 않는 방식이에요.

프론트의 TypeScript 타입은 Java 서버 응답을 실행 중 검증하지 않아요. 공통 API의 필드와 오류 코드를 함께 관리하되, Mock 전용 기능과 서버에서만 적용하는 권한·잠금 정책은 구분해서 확인해야 해요.

## 문서 구성

[관리자 API](./admin-api.md)에 엔드포인트별 요청·응답, 권한, 오류와 집계 기준을 모았어요. 영수증과 발송처럼 Mock에서만 제공하는 기능도 그 문서에 표시해요.

게시물 편집기는 HTML 본문과 `attachment://{id}` 참조를 사용해요. 기존 Markdown·일반 텍스트도 읽을 수 있어요. 저장 형식과 첨부 참조 규칙은 [본문 계약](./admin-api.md#본문-계약-html편집기-attachment)을 확인해주세요.

## 진실의 출처

| 확인할 내용 | 원본 소스 |
|---|---|
| 프론트 타입 | `src/lib/types.ts` |
| 요청 구성 | `src/api/client.ts` |
| 개발용 Mock | `src/mocks/handlers.ts`, `src/mocks/fixtures.ts` |
| 서버 요청·응답 | `template-spring/core/core-admin-impl/.../controller/` |
| 서버 오류 | `AdminError.java`, `CommonError.java` |

설명과 구현이 다르면 해당 동작을 처리하는 소스를 기준으로 문서를 고쳐요. 내부 설계 문서는 Git에 포함되지 않으므로 저장소를 클론하는 것만으로 받을 수는 없어요.

## 쌍 운영 규칙

### 서버 변경 시

1. 서버의 요청·응답 DTO, 오류 코드와 권한 조건을 수정해요.
2. 프론트의 `types.ts`, `client.ts`, Mock과 사용하는 화면을 함께 맞춰요.
3. 두 저장소에 같은 커밋 메시지로 PR을 만들어 관련 변경을 찾을 수 있게 해요.
4. `npm run build`와 `npm test`를 실행하고, 실서버 모드에서도 바뀐 요청의 응답과 오류를 확인해요.

빌드는 프론트 내부의 타입 관계를, Mock 기반 테스트는 화면과 클라이언트 동작을 검사해요. Mock과 실서버 응답을 자동으로 전수 비교하는 검사는 없으므로 실서버 확인을 대신하지 못해요.

### 프론트 변경 시

프론트에서 필요한 필드가 달라지면 서버 계약부터 함께 검토해요. 화면만 먼저 바꿔 아직 없는 응답을 기대하지 않도록 해요.

## 계약 변경 시 확인 포인트

- [ ] 서버 DTO와 프론트 타입의 필드·null 허용 여부가 맞아요.
- [ ] Mock 데이터와 처리 코드가 변경을 반영해요.
- [ ] 기존 화면과 권한별 동작이 유지돼요.
- [ ] 빌드·테스트와 실서버의 성공·오류 응답을 확인했어요.
- [ ] API 문서에 지원 범위와 변경 내용을 반영했어요.

## 관련 문서

- [관리자 API](./admin-api.md)
- [화면 가이드](../guide/screens.md)
- [Mock·실서버 실행](../guide/derived-repo.md#3-mock--실서버-전환)
- [백엔드 소스](https://github.com/storkspear/template-spring)
