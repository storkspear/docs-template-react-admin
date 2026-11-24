# 홈서버 배포

관리자 콘솔을 Mac mini의 nginx와 Cloudflare Tunnel로 제공하는 절차예요. 원본 저장소의 `tools/deploy.sh`가 빌드, 파일 전송, nginx 설정, DNS·Tunnel 등록과 배포 확인을 처리해요.

## 환경과 준비 사항

| 대상 | 설정 파일 | 관리자 주소 기본값 | 백엔드 주소 기본값 | Mock 기본값 |
|---|---|---|---|---|
| dev | `.env` | `dev-admin.<domain>` | `dev-server.<domain>` | `true` |
| prod | `.env.prod` | `admin.<domain>` | `server.<domain>` | `false` |

개발 환경은 `.env.example`, 운영 환경은 `.env.prod.example`을 참고해 설정해요. `BASE_DOMAIN`, `DEPLOY_HOST`, `DEPLOY_SSH_USER`, `CLOUDFLARE_API_TOKEN`이 필요해요. `ADMIN_SUBDOMAIN`, `BACKEND_SUBDOMAIN`, `VITE_USE_MOCK`으로 기본값을 바꿀 수 있어요. 서브도메인을 바꿨다면 dev와 prod가 같은 주소를 가리키지 않는지 확인해주세요.

설정 파일에는 비밀 값이 들어가므로 Git에 커밋하지 않아요. 배포 명령은 대상 파일을 읽지만 실행한 셸에서 export한 변수도 상속받을 수 있으므로, 다른 환경의 설정이 남아 있지 않은 셸에서 실행해요.

로컬에는 Node.js, npm, bash, curl, jq, rsync와 SSH가 필요해요. 서버에는 nginx와 사용할 Cloudflare Tunnel이 준비돼 있어야 해요. 여러 터널이 있으면 `CLOUDFLARE_TUNNEL_ID`로 대상을 지정해요.

### SSH와 nginx 권한

- `SSH_KEY`에 배포 키 경로를 지정하거나 `SSH_PRIVATE_KEY`에 키 본문을 설정할 수 있어요. 둘 다 없으면 기존 SSH 설정을 사용해요. 여러 키를 차례로 시도하는 환경에서는 서버의 인증 시도 횟수 제한에 걸릴 수 있어요.
- 배포 계정에는 nginx 설정 디렉터리 쓰기 권한과 `nginx -t`, `nginx -s reload` 실행 권한이 필요해요.
- `NGINX_SUDO=1`은 nginx 명령에만 sudo를 붙여요. 설정 파일 쓰기 권한은 별도로 준비해야 해요. 서비스 실행 계정이나 소유권은 기존 홈서버 운영 방식에 맞춰 설정해요.

홈서버 자체의 구조와 운영은 [홈서버 런북](https://github.com/storkspear/docs-infra-homeserver/blob/main/docs/runbook.md)을 참고해요. 이 링크는 저장소 접근 권한이 필요할 수 있어요.

## 배포하기

```bash
./factory dev deploy
# 운영 환경에 배포할 때
./factory prod deploy
```

스크립트는 다음 순서로 실행돼요.

1. 대상 환경의 설정과 권한을 확인해요.
2. Vite로 빌드하고 rsync로 산출물을 전송해요.
3. SPA fallback과 `/api` 프록시를 포함한 nginx server block을 만들어요. API는 같은 출처로 요청하고, nginx가 백엔드 호스트에 맞춰 전달해요.
4. Cloudflare DNS와 Tunnel ingress를 등록하거나 갱신해요.
5. 배포 주소의 응답을 검사해요.

Mock·실서버를 전환하려면 대상 설정 파일의 `VITE_USE_MOCK`을 바꾸고 다시 배포해요. `VITE_API_BASE`도 빌드에 반영되며, 생략하면 같은 출처의 `/api`를 사용해요. 배포 스크립트가 이 두 Vite 값을 셸 환경에 설정해 로컬 `.env`의 값이 운영 빌드에 섞이는 것을 막아요.

## 응답 헤더와 캐시

생성한 nginx 설정은 CSP, `X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy`, `Permissions-Policy` 헤더를 오류 응답에도 붙여요. `index.html`과 MSW 워커는 `no-store`, 해시가 붙은 `/assets/`는 1년 캐시와 `immutable`을 사용해요.

CSP 기본값은 `tools/deploy.sh`의 `DEFAULT_CSP`에 있어요. 자기 출처의 스크립트, jsDelivr 글꼴, HTTPS 미디어, YouTube·draw.io 프레임 등을 허용해요. 다른 출처가 필요하면 허용 범위를 확인한 뒤 `NGINX_CSP`로 정책 전체를 지정해요.

## 성공 판정과 실패 후 확인

SPA HTML 확인은 기본 6회, 5초 간격으로 재시도해요. 횟수는 `HEALTH_TRIES`로 바꿀 수 있어요. CSP 헤더와 실서버 모드의 `/api/admin/health` 확인은 각각 한 번 실행해요. Mock 모드에서 백엔드가 없는 경우는 경고로 처리해요.

검증이 실패하면 파일 전송과 설정 반영이 이미 끝났더라도 종료 코드는 0이 아니에요. 출력된 실패 단계를 확인하고 배포 주소, nginx 설정, Tunnel 연결을 점검해주세요. 이 검사는 로그인·권한·데이터베이스 기능 전체를 검사하지 않으므로 배포 후에는 필요한 화면도 확인해요.

## 배포 정리

```bash
./factory dev clear
# 운영 배포를 정리할 때만
./factory prod clear
```

명령은 선택한 주소의 DNS, Tunnel ingress, nginx 설정과 배포 디렉터리를 삭제해요. 실행 전에 환경 파일의 주소와 호스트를 확인해주세요. 두 환경에 같은 주소나 경로를 설정하면 구분해서 정리할 수 없어요.

일부 삭제나 nginx 재적용이 실패해도 가능한 나머지 정리를 진행한 뒤 실패 코드로 끝나요. 이미 없는 대상은 정리된 것으로 처리해요. 사전 조회가 실패하면 잘못된 대상을 지우지 않도록 중단해요. 실패 후에는 로그에 남은 항목을 확인하고 원인을 해결한 뒤 다시 실행해요.

## 관련 문서

- [파생 레포 가이드](./derived-repo.md)
- [API 계약](../api-contract/README.md)
- [배포 스크립트](https://github.com/storkspear/template-react-admin/blob/main/tools/deploy.sh)
