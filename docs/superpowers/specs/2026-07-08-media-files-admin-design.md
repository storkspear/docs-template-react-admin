# 미디어 파일 관리 화면 (이미지·영상·오디오) — 설계

> 유형: Design spec · 대상: `template-react-admin` (정본) → `admin-console` 동기화 · 작성일: 2026-07-08
> 개정: v2 — 4-렌즈 spec 리뷰(실현성·일관성·완결성) 반영

## 1. 배경 · 문제

관리자 `/files`(`FilesPage.tsx`)는 앱 스토리지(`<slug>-uploads` 버킷) 파일을 **조회·검역(숨김)·복원·영구삭제**하는 모더레이션 도구다. 현 한계:

1. **미리보기가 비어 보인다.** 개발 기본이 목 모드(`VITE_USE_MOCK=true`)인데 목 이미지 URL이 1×1 **투명** PNG(`PIXEL_PNG`)라 40×40 빈 칸으로 렌더 → "미리보기 안 됨"으로 체감. (코드 버그 아님, 목 플레이스홀더 한계. 실 백엔드 presigned URL이면 실제 썸네일.)
2. **미디어 타입 인식이 없다.** `IMAGE_EXT`(png/jpg/gif/webp)만 이미지로 보고 40px `<img>`, 나머지는 파일 아이콘. 영상·오디오는 미리보기·재생 개념 없음.
3. **타입 필터·그리드·오디오 재생이 없다.** 앱+prefix 필터만 있어 시각 스캔(썸네일 그리드)·오디오 청취가 불가.

## 2. 목표 · 비목표

### 이번 목표 (뷰 전용)
- 파일을 **이미지/영상/오디오/기타**로 분류 + **타입 탭 필터**
- **이미지·영상**은 **그리드 썸네일**, **오디오**는 **인라인 재생**(한 번에 하나)
- **미리보기가 실제로 보이게**: 목 fixtures 실사 샘플 + 실 모드(presigned) 동일 동작
- 기존 **모더레이션(검역/복원/삭제)** 유지

### 비목표 (별도 후속)
- **업로드 기능**(백엔드 admin 업로드 엔드포인트 + UI) — [§12]
- **백엔드 `contentType` 필드** — 확장자 분류로 이번 요구 충분
- **고급 라이트박스**(줌/회전/슬라이드쇼) — 이번 '확대'는 단순 모달까지만(§5)
- 서버 페이지네이션 방식 변경(현 `max` 절단 + `truncated` 유지, 단 max 상향 §5)

## 3. 타입 분류 (확장자 기반, 프론트 전용)

`AdminFile`에 `contentType`이 없어 파일 `key` 확장자로 분류. 신규 `src/lib/mediaKind.ts`:

```ts
export type MediaKind = 'image' | 'video' | 'audio' | 'other'
const IMAGE = /\.(png|jpe?g|gif|webp|avif|bmp|svg)$/i
const VIDEO = /\.(mp4|webm|mov|m4v|ogv)$/i
const AUDIO = /\.(mp3|m4a|aac|wav|ogg|oga|flac)$/i
export function mediaKind(key: string): MediaKind { /* IMAGE→'image' … 아니면 'other' */ }
```
확장자 없는 키는 `'other'`. 기존 `IMAGE_EXT` 상수는 이 유틸로 대체(중복 제거).

> **한계 명시**: 분류가 클라이언트 확장자 기반이라 (a) 확장자 없는/오기된 키는 오분류, (b) 타입별 정확한 총량은 알 수 없다(§5의 절단 이슈와 연결). 정밀도가 필요하면 후속에서 백엔드 `contentType`.

## 4. 메뉴 · 탭 구조 (단일 페이지 + 타입 탭)

`/files` 라우트 **하나** 유지, 페이지 상단 타입 탭: `전체 | 이미지 | 영상 | 오디오`.

- **컴포넌트**: 기존 **`src/components/ViewTabs.tsx` 재사용**(SendPage·AppDetailPage·ComponentsPage가 뷰 전환에 쓰는 정본 패턴 — `Segmented` 아님). 각 탭 라벨에 **로드된 건수**를 붙인다: `이미지 3`, 절단 시 `이미지 3+`(§5). ViewTabs가 label 문자열만 받으므로 컴포넌트 변경 없이 라벨 임베드로 처리.
- **deep-link (`?type=`) — 신규 2-way URL 동기화 패턴**:
  - ⚠️ 코드베이스의 기존 `?slug=`은 **read-once**(마운트 시 초기 상태 시드만, `setSearchParams` 미사용)다. `?type=`은 탭 클릭마다 URL을 되쓰는 **새 패턴**임을 명시.
  - 되쓸 때 **함수형 업데이터로 기존 파라미터 보존**해 `?slug=`가 유실되지 않게: `setSearchParams(prev => { const n = new URLSearchParams(prev); v ? n.set('type', v) : n.delete('type'); return n })`.
  - 전체 = `type` 파라미터 없음.
- **nav 메뉴는 "파일" 1개 그대로**(`nav.tsx` 단일 소스 무변경).
- **prefix × type 상호작용**: 서버에서 `slug`+`prefix`로 fetch → **클라에서 `mediaKind`로 AND 필터**. 건수 뱃지·`truncated`·빈 상태는 모두 **현재 prefix 범위 내** 기준. prefix 변경 시 탭 뱃지 재계산.

## 5. 타입별 뷰 모드

전체 파일을 **max 상향**(이 화면은 `getAppFiles(slug, { prefix, max: 1000 })`로 요청)해 클라 분류 정확도를 높인다. 그래도 절단될 수 있으므로 아래 절단 규칙을 지킨다.

| 탭 | 뷰 | 미리보기 | '확대/재생' | 액션 |
|---|---|---|---|---|
| 전체 | 테이블(현행 `AdminMiniGrid`, 모바일은 기존 `MobileCardList` 폴백 유지) | `MediaPreview` 셀 | — | 상태별(§아래) |
| 이미지 | **그리드**(카드) | `<img>` 썸네일 | 카드 클릭 → **Modal에 풀사이즈 `<img>`**(줌/슬라이드쇼 없음) | 상태별 |
| 영상 | **그리드**(카드) | `<video src={url + '#t=0.1'} muted preload="metadata">` (첫 프레임 강제 시킹) + ▶ 오버레이 | ▶ 클릭 → **카드 내 인라인 `<video controls>` 재생** | 상태별 |
| 오디오 | **리스트** | 파일명 + 인라인 ▶/⏸ + **경과시간**(진행바는 후속) | 인라인 재생(§6) | 상태별 |

**액션(모든 탭 공통, `quarantined` 상태 조건부)** — 현행 테이블 동작을 그리드/리스트에도 적용:
- `quarantined === false` → **[검역]**(Popconfirm)
- `quarantined === true` → **[복원]** + **[삭제]**(danger, 영구)
- ⚠️ 타입 탭에 검역 파일이 보이므로 **복원 액션을 반드시 각 탭 카드/행에도** 노출(전체 탭 전용으로 두지 않음 — 리뷰 지적: 검역 표시하면서 복원 경로만 막는 건 UX 함정).

**영상 썸네일(핵심 수정)**: `preload="metadata"`만으론 다수 브라우저(Firefox/Safari)가 첫 프레임을 안 그려 검은 박스가 된다(고치려던 '빈 칸'을 영상에서 재현). → **src에 `#t=0.1` 미디어 프래그먼트**를 붙여 브라우저가 0.1s로 seek하며 프레임을 디코드/페인트하게 한다(크로스브라우저, fragment라 서버 미전송 → presigned 쿼리스트링과 안전). §11 테스트로 검증.

**절단(truncated) 규칙**:
- 건수 뱃지는 **'로드된 집합' 기준**. `truncated`면 `3+`로 불완전 표기.
- **`truncated` 경고를 전체 탭뿐 아니라 모든 타입 탭에서 노출**(각 탭이 절단된 부분집합을 보므로). 문구 예: "일부만 표시됐어요 — prefix로 좁혀주세요. 이 타입에 더 있을 수 있어요."

**빈 상태**: 타입 0건 탭은 **비활성/숨김 없이 계속 노출**하고 Ant `<Empty>`로 안내(예: "이 앱에 오디오 파일이 없어요"). `truncated`로 비어 보이는 경우와 진짜 0건을 문구로 구분.

**모바일(useIsMobile)**: 이미지/영상 그리드 **모바일 2열 / 데스크톱 4~6열**(CSS grid `auto-fill minmax` 또는 useIsMobile 분기). 오디오 리스트는 풀폭 행. ViewTabs는 좁은 화면에서 가로 스크롤/축약. 전체 탭은 기존 모바일 카드 폴백 유지.

## 6. 오디오 플레이어 (한 번에 하나)

페이지 레벨 훅 `src/lib/useAudioController.ts` — **단일 `<audio>` 인스턴스**(ref)를 공유:

- 상태: `currentKey: string | null`, `isPlaying: boolean`. (경과/총 시간은 **재생 중인 행만** 표시하도록 지역 상태/ref로 격리 — 전역 state로 두면 `timeupdate`(~4회/초)마다 리스트 전체 리렌더.)
- `toggle(key, url)`: 같은 key 재생 중이면 pause; 아니면 `audio.src=url` 교체 후 `play()`. **`play()` 반환 promise를 await/catch**해 src 교체 직후 발생하는 `AbortError`('play() interrupted by new load')를 무시(빠른 탭/트랙 전환 대비).
- `stop()`: pause + currentKey 리셋.
- **필수 네이티브 리스너**:
  - `ended` → `currentKey=null, isPlaying=false` (없으면 클립 종료 후 버튼이 ⏸로 고착).
  - `error` → "URL이 만료됐어요(≈10분), 새로고침 해주세요" 메시지 + 목록 `invalidateQueries`.
  - `timeupdate` → 재생 행의 경과시간만 갱신(위 격리 원칙).
- 각 행 ▶ 버튼: `currentKey===row.key && isPlaying`이면 ⏸.

## 7. `MediaPreview` 컴포넌트 (현 `PreviewCell` 대체)

타입 인식 미리보기(전체 탭 셀 + 그리드 카드 공용):
- `image` → `<img src={url}>` (`object-fit: cover`)
- `video` → `<video src={url + '#t=0.1'} muted preload="metadata">` + ▶ 오버레이
- `audio` → 오디오 아이콘(`SoundOutlined`)
- `other` → 파일 아이콘(`FileOutlined`)
- 로드 실패(`onError`) → 대체 아이콘 폴백(빈 칸 방지).

## 8. 목 fixtures 개선 (미리보기 실제로 보이게)

`src/mocks/fixtures.ts` + `src/mocks/assets/`:

- `PIXEL_PNG`(투명) → **눈에 보이는 실사 샘플**. 번들 부담 최소화 위해 **초소형 실 파일**을 `src/mocks/assets/`에 두고 Vite import(URL 해석):
  - **이미지**: 작은 컬러 이미지(≤30KB). 초경량 대안으로 **컬러 SVG data-URI**도 허용(영상/오디오와 달리 이미지는 data-URI로 충분).
  - **영상**: 1~2초 컬러 mp4(≤150KB).
  - **오디오**: 1~2초 톤/무음 mp3 or m4a(≤100KB).
- **샘플 확보 방법(명시)** — CC0 자체 생성. 예 ffmpeg:
  - `ffmpeg -f lavfi -i color=c=teal:s=320x240:d=2 -pix_fmt yuv420p sample-clip.mp4`
  - `ffmpeg -f lavfi -i sine=frequency=440:duration=2 sample-voice.mp3`
  - 이미지: 단색 320×240 png/jpg(자체 생성).
- `buildFiles()`에 **영상·오디오 key 추가**: `uploads/clip.mp4`, `uploads/voice.m4a`, `uploads/song.mp3` 등 + 기존 이미지 url을 실사 샘플로 교체. (타입 필터/그리드/플레이어 목 데모 가능)

## 9. 실 모드 호환 (백엔드 무변경)

- 실 백엔드는 `AdminFile.url`에 **presigned GET URL**(~10분) 반환 → `<img>/<video>/<audio>` 표시에 **CORS 불필요**(canvas/fetch 아님).
- **`AdminFile` 모델 무변경** — mock↔real 스키마 동일 원칙. `contentType`은 후속.
- ⚠️ **목 모드 미디어 Range 검증 필요(가정 금지)**: MSW SW가 미핸들 요청을 passthrough하는데, `<video>/<audio>`는 Range(`bytes=`) 요청을 보내고 SW가 Vite의 206 Partial Content를 그대로 되돌려야 재생/시크가 된다. Chromium+짧은 클립은 대체로 되나 **Safari 시크 등 러프 엣지**가 알려져 있다. → 구현 시 **dev(Vite)와 build/preview(dist) 양쪽에서 실제 브라우저로 오디오 재생·시크·영상 프레임 로드 1회 검증**. 문제 시 미디어 요청만 SW 우회.

## 10. 컴포넌트 · 파일 영향

| 파일 | 변경 |
|---|---|
| `src/lib/mediaKind.ts` | **신규** — 확장자→`MediaKind` |
| `src/lib/useAudioController.ts` | **신규** — 단일 재생 컨트롤러(ended/error/AbortError 처리) |
| `src/pages/FilesPage.tsx` | `ViewTabs` 타입 탭 + `?type=` 2-way 동기화 + prefix×type AND 필터 + 타입별 조건부 뷰 |
| `src/components/MediaGrid.tsx` | **신규** — 이미지/영상 카드 그리드(썸네일+메타+상태별 액션+모바일 열 수) |
| `src/components/MediaPreview.tsx` | **신규** — 타입 인식 미리보기(현 `PreviewCell` 대체) |
| `src/mocks/fixtures.ts` + `src/mocks/assets/*` | 실사 샘플 미디어 + 영상/오디오 fixture |
| `src/components/ViewTabs.tsx` | 재사용(변경 없음, 라벨에 건수 임베드) |
| `src/lib/types.ts`, `src/nav.tsx` | 변경 없음 |
| `src/api/client.ts` | 변경 없음(검역/복원/삭제·getAppFiles 재사용, max 인자만 전달) |

## 11. 테스트

- `mediaKind` 유닛: 확장자별 분류(경계·대소문자·확장자 없음).
- 타입 탭 필터: 각 탭에 해당 종류만(전체=모두). **prefix 변경 시 탭 뱃지 재계산**.
- **절단**: `truncated`일 때 뱃지 `N+` 표기 + 모든 탭에 경고 노출.
- **검역 파일**: 타입 탭에서 quarantined 카드에 **복원 액션 노출**.
- 오디오 단일 재생: A 재생 중 B 재생 → A 정지·B 재생; 클립 `ended` 시 버튼 ▶로 복귀; 빠른 전환 시 AbortError로 안 죽음.
- 그리드 렌더: 이미지/영상 카드가 목 샘플로 실제 미디어 표시(빈 칸 아님). **영상 `#t=0.1` 첫 프레임 렌더** 확인.
- `MediaPreview` 로드 실패 → 대체 아이콘 폴백.
- 빈 상태: 타입 0건 탭 `<Empty>` 문구.
- **회귀**: 검역/복원/삭제, `truncated` 경고, 앱/prefix 필터, 전체 탭 모바일 카드 폴백.
- **수동 검증**(§9): 목 dev + build/preview 브라우저에서 오디오 재생·시크, 영상 프레임 로드.

## 12. 스코프 밖 · 후속 (기록)

- **업로드 기능** — admin 업로드(백엔드 presigned PUT/multipart + 인증·크기·타입 검증 + 프론트 UI). 별도 spec.
- **백엔드 `contentType`** — 확장자 의존·타입별 총량 부정확 한계 해소.
- 오디오 **시킹 진행바**, 그리드/리스트 토글, 고급 라이트박스(줌/슬라이드쇼).

## 13. 정본 · 전파

- **정본**: `template-react-admin` 구현. **전파**: `admin-console`(파생 레포) 동기화 필요 — 사용자가 5173에서 admin-console을 돌리므로, 실제 확인은 admin-console 동기화 후 또는 template-react-admin dev 서버로.

## 14. v3 개정 — 반응형 모달 플레이어 (유튜브식) + 최종 리뷰 픽스

> 사용자 요구: "영상은 어디서 누르든 새 탭 말고 **크게 뜨는 팝업**으로 재생, 브라우저 크기에 따라 리사이징(유튜브처럼)." 아래는 §5의 "확대/재생" 동작을 **대체(supersede)** 한다.

### 14.1 `MediaModal` — 신규 컴포넌트
- `src/components/MediaModal.tsx`. props: `{ open: boolean; onClose: () => void; kind: 'image' | 'video'; url: string }`.
- Ant `Modal`(`open`, `onCancel=onClose`, `footer={null}`, `centered`, `width="auto"`, `styles={{ body: { padding: 0, lineHeight: 0 } }}`, `destroyOnClose`)로 구현 → **Esc·바깥(mask) 클릭·X 로 닫힘**(AntD 기본). `destroyOnClose`로 닫을 때 `<video>` 재생 중지.
- **반응형**(브라우저 크기에 맞춰 리사이징, 비율 유지):
  - image → `<img src={url} style={{ maxWidth:'90vw', maxHeight:'85vh', width:'auto', height:'auto', objectFit:'contain', display:'block' }} />`
  - video → `<video src={url} controls autoPlay style={{ maxWidth:'90vw', maxHeight:'85vh', width:'auto', height:'auto', display:'block' }} />` (네이티브 컨트롤 = 재생/시크/볼륨 + **전체화면 버튼**)
- 줌/슬라이드쇼는 여전히 비목표(§2) — 단순 반응형 뷰어까지만.

### 14.2 클릭 동작 통일 (그리드 + 테이블 둘 다)
- **이미지·영상 탭 그리드**(`MediaGrid`): 카드 클릭 → `MediaModal` 오픈. **카드 내 인라인 `<video controls>` 재생 제거**(§5의 인라인 재생 대체). 썸네일은 그대로(이미지 `<img>`, 영상 `#t=0.1`+▶).
- **전체 탭 테이블**(`FilesPage`): 미리보기 셀(`MediaPreview`)이 **image/video면 클릭 가능** → `MediaModal` 오픈. **미디어 파일은 파일명 링크의 `target="_blank"`(새 탭) 대신 모달로** 열리게 한다(파일명 링크는 image/video일 때 preventDefault + 모달, 그 외(pdf 등)는 기존 새 탭 유지). → **영상이 새 탭으로 열리는 동작 제거**.
- 오디오는 §6대로 리스트 인라인 재생(모달 아님).

### 14.3 최종 리뷰 픽스 (같이 반영)
- 🔴 **오디오 재생이 모더레이션 중 끊김**: `FilesPage`의 `onExpired`(invalidate) 콜백이 매 렌더 새 참조 → `useAudioController` effect(deps에 onExpired) 재실행 → cleanup의 `audio.pause()`가 재생을 끊는다. → **`onExpired`를 `useCallback`으로 안정화**(또는 컨트롤러 effect가 onExpired 참조를 ref에 보관해 deps에서 제거). 검역/복원/삭제 중에도 재생 유지.
- 🟡 **`?type` 비정상 값 → 백지**: `activeType`가 all/image/video/audio가 아니면(오타·`other`) `'all'`로 폴백(정규화). 렌더 분기 항상 매치.
- 🟡 **MediaGrid 썸네일 폴백**: 그리드 `<img>`/`<video>`에도 `MediaPreview`처럼 `onError`(+ 필요 시 stall) → 대체 아이콘. 깨진 이미지/검은 박스 방지.
- 🟡 **빈 상태 truncated 구분**: `truncated`로 0건일 때와 진짜 0건을 `<Empty>` 문구로 구분(예: 절단이면 "prefix로 좁혀 더 찾아보세요", 진짜 0건이면 "이 타입 파일이 없어요").
- 🟡 **샘플 mp4 인라인 회피**: `sample-clip.mp4`가 Vite 기본 `assetsInlineLimit`(4KB) 미만이면 data-URI로 인라인돼 `#t=0.1` 시킹이 불안정 → **파일을 4KB 초과로 재인코딩**(예: 길이/해상도↑)하거나 `vite.config`의 `assetsInlineLimit`을 미디어에 0 적용해 **파일로 방출**되게 한다.

### 14.4 컴포넌트 영향 (추가)
- `src/components/MediaModal.tsx` **신규**.
- `src/components/MediaGrid.tsx` 수정(인라인 재생 → MediaModal), `src/pages/FilesPage.tsx` 수정(테이블 미디어 클릭 → MediaModal, onExpired useCallback, type 정규화), `src/components/MediaPreview.tsx`(클릭 핸들 전달 or 셀에서 onClick 래핑).
