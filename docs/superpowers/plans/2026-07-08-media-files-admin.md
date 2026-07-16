# 미디어 파일 관리 화면 (이미지·영상·오디오) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** admin `/files` 화면을 이미지/영상/오디오 타입 탭 + 그리드 썸네일 + 오디오 인라인 플레이어로 확장하고, 목 미리보기가 실제로 보이게 한다(모더레이션·백엔드 무변경).

**Architecture:** `/files` 단일 라우트에 `ViewTabs` 타입 탭(`?type=` URL 동기화)을 얹고, 서버에서 `slug+prefix`로 받은 목록을 클라이언트 `mediaKind()`로 AND 필터한다. 전체=기존 ag-grid 테이블(미리보기 셀만 타입 인식으로 교체), 이미지/영상=카드 그리드, 오디오=단일 `<audio>` 컨트롤러 기반 인라인 재생 리스트. 목 fixtures에 실사 샘플 미디어를 넣어 미리보기를 실제로 렌더한다.

**Tech Stack:** React 19 + TypeScript + Vite + Ant Design + TanStack Query + ag-grid-community + MSW. 상태: `useSearchParams`(react-router). 스타일: 인라인 style + AntD 토큰.

## Global Constraints

- **검증 게이트(유닛 테스트 없음)**: 각 태스크 마지막에 `npm run build`(= `tsc -b && vite build`) **그린** + `npm run lint`(oxlint) **그린**. UI 변경 태스크는 추가로 `npm run dev` 목 모드에서 육안 확인. (이 레포엔 vitest/testing-library가 없고 품질 게이트가 build+lint임 — 사용자 결정.)
- **`AdminFile` 모델·`nav.tsx`·`api/client.ts` 시그니처 변경 금지** (mock↔real 스키마 동일 원칙). `getAppFiles`는 이미 `max` 인자를 받으므로 호출부에서 `max:1000`만 전달.
- **커밋**: Conventional Commits `type(admin): subject`, **Co-Authored-By 트레일러 금지**(husky 차단). 한 태스크 = 한 커밋.
- **정본**: `template-react-admin`. `admin-console` 전파는 이 계획 밖(별도).
- **presigned URL**: `AdminFile.url`은 ~10분 만료. 미디어 요소는 `onError`로 대체 처리.

---

### Task 1: `mediaKind` 분류 유틸

**Files:**
- Create: `src/lib/mediaKind.ts`

**Interfaces:**
- Produces: `type MediaKind = 'image'|'video'|'audio'|'other'`; `function mediaKind(key: string): MediaKind`

- [ ] **Step 1: 유틸 작성**

`src/lib/mediaKind.ts`:
```ts
export type MediaKind = 'image' | 'video' | 'audio' | 'other'

const IMAGE = /\.(png|jpe?g|gif|webp|avif|bmp|svg)$/i
const VIDEO = /\.(mp4|webm|mov|m4v|ogv)$/i
const AUDIO = /\.(mp3|m4a|aac|wav|ogg|oga|flac)$/i

/** 파일 key(경로)의 확장자로 미디어 종류 판별. 확장자 없으면 'other'. */
export function mediaKind(key: string): MediaKind {
  if (IMAGE.test(key)) return 'image'
  if (VIDEO.test(key)) return 'video'
  if (AUDIO.test(key)) return 'audio'
  return 'other'
}
```

- [ ] **Step 2: 빌드·린트 검증**

Run: `npm run build && npm run lint`
Expected: 둘 다 그린(에러 0). (수동 확인: `mediaKind('a/b.MP4')==='video'`, `mediaKind('x.txt')==='other'`, `mediaKind('noext')==='other'`를 임시 콘솔로 1회 확인 후 제거 가능.)

- [ ] **Step 3: 커밋**

```bash
git add src/lib/mediaKind.ts
git commit -m "feat(admin): 파일 key 확장자 기반 mediaKind 분류 유틸"
```

---

### Task 2: 목 fixtures — 실사 샘플 미디어 + 영상/오디오 파일

**Files:**
- Create: `src/mocks/assets/sample-clip.mp4`, `src/mocks/assets/sample-audio.mp3` (초소형 생성물)
- Modify: `src/mocks/fixtures.ts` (PIXEL_PNG 교체 영역 ~609, `buildFiles()` ~617-655)

**Interfaces:**
- Consumes: `mediaKind` 아님(무관). `AdminFile` 타입(기존).
- Produces: `buildFiles()`가 이미지/영상/오디오/기타를 모두 포함하고, 이미지/영상/오디오 url이 실제 로드되는 소스를 가진다.

- [ ] **Step 1: 초소형 샘플 미디어 생성 (ffmpeg)**

Run:
```bash
mkdir -p src/mocks/assets
ffmpeg -y -f lavfi -i color=c=teal:s=320x240:d=2 -pix_fmt yuv420p -movflags +faststart src/mocks/assets/sample-clip.mp4
ffmpeg -y -f lavfi -i sine=frequency=440:duration=2 -q:a 9 src/mocks/assets/sample-audio.mp3
```
Expected: 두 파일 생성(각 <150KB/<100KB 목표). ffmpeg 없으면 CC0 초소형 mp4/mp3를 같은 경로/이름으로 준비. 이미지는 파일 없이 컬러 data-URI 사용(Step 2).

- [ ] **Step 2: fixtures import + 샘플 URL 교체**

`src/mocks/fixtures.ts` 상단 import 추가:
```ts
import sampleClip from './assets/sample-clip.mp4'
import sampleAudio from './assets/sample-audio.mp3'
```
`PIXEL_PNG`(투명) 정의를 **눈에 보이는 컬러 이미지 data-URI**로 교체(파일 불필요, 40×40에서 보이는 단색):
```ts
/** 눈에 보이는 샘플 썸네일 — 컬러 PNG data-uri(1x1 solid, cover로 확대). */
const SAMPLE_IMG =
  'data:image/svg+xml;utf8,' +
  encodeURIComponent(
    '<svg xmlns="http://www.w3.org/2000/svg" width="64" height="64"><rect width="64" height="64" fill="#4f46e5"/></svg>',
  )
```

- [ ] **Step 3: `buildFiles()`에 영상/오디오 추가 + 이미지 url 교체**

`src/mocks/fixtures.ts`의 `buildFiles()` 반환 배열을 아래로 교체(이미지 3개 url→`SAMPLE_IMG`, 영상·오디오 추가, PDF·검역 유지):
```ts
function buildFiles(): AdminFile[] {
  return [
    { key: 'avatars/user-1.png', size: 84_213, lastModified: iso(2), url: SAMPLE_IMG, quarantined: false },
    { key: 'avatars/user-2.jpg', size: 152_004, lastModified: iso(5), url: SAMPLE_IMG, quarantined: false },
    { key: 'uploads/banner.webp', size: 342_880, lastModified: iso(1), url: SAMPLE_IMG, quarantined: false },
    { key: 'uploads/promo-clip.mp4', size: 1_920_000, lastModified: iso(1), url: sampleClip, quarantined: false },
    { key: 'uploads/intro-voice.m4a', size: 220_400, lastModified: iso(2), url: sampleAudio, quarantined: false },
    { key: 'uploads/theme-song.mp3', size: 640_120, lastModified: iso(4), url: sampleAudio, quarantined: false },
    { key: 'docs/monthly-report.pdf', size: 1_048_576, lastModified: iso(10), url: 'https://example-signed.local/docs/monthly-report.pdf?sig=mock', quarantined: false },
    { key: 'quarantine/uploads/suspicious.png', size: 45_120, lastModified: iso(3), url: SAMPLE_IMG, quarantined: true },
  ]
}
```

- [ ] **Step 4: 타입 선언 (미디어 import)**

`src/vite-env.d.ts`에 mp4/mp3 모듈 선언이 없으면 추가(Vite 기본 asset import 타입):
```ts
declare module '*.mp4' { const src: string; export default src }
declare module '*.mp3' { const src: string; export default src }
```

- [ ] **Step 5: 빌드·린트·목 확인**

Run: `npm run build && npm run lint`
Expected: 그린. 이어서 `npm run dev` → `/files`(현행 테이블) 이미지 셀이 **파란 사각형으로 보이면** 성공(투명 픽셀 아님). 영상/오디오 행은 아직 파일 아이콘(다음 태스크에서 미리보기).

- [ ] **Step 6: 커밋**

```bash
git add src/mocks/assets src/mocks/fixtures.ts src/vite-env.d.ts
git commit -m "feat(admin): 파일 목 fixtures에 실사 샘플 미디어(이미지/영상/오디오) 추가"
```

---

### Task 3: `MediaPreview` — 타입 인식 미리보기 (현 `PreviewCell` 대체)

**Files:**
- Create: `src/components/MediaPreview.tsx`
- Modify: `src/pages/FilesPage.tsx` (기존 `PreviewCell`(15-30행) 제거, 컬럼 `cellRenderer`를 `MediaPreview`로)

**Interfaces:**
- Consumes: `mediaKind` (Task 1)
- Produces: `<MediaPreview fileKey={string} url={string} size?={number} />`

- [ ] **Step 1: 컴포넌트 작성**

`src/components/MediaPreview.tsx`:
```tsx
import { useState } from 'react'
import type { CSSProperties } from 'react'
import { FileOutlined, SoundOutlined } from '@ant-design/icons'
import { mediaKind } from '../lib/mediaKind'

interface MediaPreviewProps {
  fileKey: string
  url: string
  /** 정사각 한 변 px. 기본 40(테이블 셀). 그리드 카드는 더 크게. */
  size?: number
}

/** 파일 종류별 썸네일: 이미지=img, 영상=video 첫 프레임(#t=0.1)+▶, 오디오/기타=아이콘. 로드 실패 시 아이콘 폴백. */
export default function MediaPreview({ fileKey, url, size = 40 }: MediaPreviewProps) {
  const [failed, setFailed] = useState(false)
  const kind = mediaKind(fileKey)
  const box: CSSProperties = { width: size, height: size, borderRadius: 4, objectFit: 'cover', display: 'block' }
  const icon: CSSProperties = { fontSize: Math.round(size * 0.5), color: 'rgba(0,0,0,0.45)' }

  if (!failed && kind === 'image') {
    return <img src={url} alt="" style={box} onError={() => setFailed(true)} />
  }
  if (!failed && kind === 'video') {
    return (
      <div style={{ position: 'relative', width: size, height: size }}>
        <video src={`${url}#t=0.1`} muted preload="metadata" style={box} onError={() => setFailed(true)} />
        <span style={{ position: 'absolute', inset: 0, display: 'flex', alignItems: 'center', justifyContent: 'center', color: '#fff', fontSize: Math.round(size * 0.35), textShadow: '0 0 3px rgba(0,0,0,0.6)' }}>▶</span>
      </div>
    )
  }
  if (kind === 'audio') return <SoundOutlined style={icon} />
  return <FileOutlined style={icon} />
}
```

- [ ] **Step 2: FilesPage에서 PreviewCell 교체**

`src/pages/FilesPage.tsx`:
- 상단 `const IMAGE_EXT = …` 와 `function PreviewCell(…) {…}` (15-30행) **삭제**. `FileOutlined` import가 다른 데서 안 쓰이면 제거.
- `import MediaPreview from '../components/MediaPreview'` 추가.
- 컬럼 정의(199행) 교체:
```tsx
{ colId: 'preview', headerName: '미리보기', width: 72, sortable: false,
  cellRenderer: (p: ICellRendererParams<AdminFile>) =>
    p.data ? <MediaPreview fileKey={p.data.key} url={p.data.url} /> : null },
```

- [ ] **Step 3: 빌드·린트·목 확인**

Run: `npm run build && npm run lint`
Expected: 그린. `npm run dev` → `/files` 테이블에서 이미지=파란 썸네일, 영상=첫 프레임+▶, 오디오=🔊 아이콘. (크로스브라우저: Chrome/Firefox/Safari 중 최소 2개에서 영상 첫 프레임 확인.)

- [ ] **Step 4: 커밋**

```bash
git add src/components/MediaPreview.tsx src/pages/FilesPage.tsx
git commit -m "feat(admin): 타입 인식 MediaPreview 컴포넌트로 파일 미리보기 교체"
```

---

### Task 4: `useAudioController` — 한 번에 하나 재생 훅

**Files:**
- Create: `src/lib/useAudioController.ts`

**Interfaces:**
- Produces:
  ```ts
  interface AudioController {
    currentKey: string | null
    isPlaying: boolean
    audio: HTMLAudioElement | null   // 재생 행이 timeupdate 구독용
    toggle: (key: string, url: string) => void
    stop: () => void
  }
  function useAudioController(onExpired?: () => void): AudioController
  ```

- [ ] **Step 1: 훅 작성**

`src/lib/useAudioController.ts`:
```ts
import { useCallback, useEffect, useRef, useState } from 'react'

export interface AudioController {
  currentKey: string | null
  isPlaying: boolean
  audio: HTMLAudioElement | null
  toggle: (key: string, url: string) => void
  stop: () => void
}

/**
 * 리스트에서 오디오를 한 번에 하나만 재생. 단일 <audio> 인스턴스를 공유해
 * 새 파일 재생 시 이전을 자동 정지한다. ended → 버튼 복귀, error(만료) → onExpired.
 * 경과시간은 전역 state로 두지 않는다(리스트 전체 리렌더 방지) — 재생 행이 audio를 직접 구독.
 */
export function useAudioController(onExpired?: () => void): AudioController {
  const ref = useRef<HTMLAudioElement | null>(null)
  const [currentKey, setCurrentKey] = useState<string | null>(null)
  const [isPlaying, setIsPlaying] = useState(false)

  const getAudio = useCallback(() => {
    if (!ref.current) ref.current = new Audio()
    return ref.current
  }, [])

  useEffect(() => {
    const audio = getAudio()
    const onEnded = () => { setCurrentKey(null); setIsPlaying(false) }
    const onErr = () => { setCurrentKey(null); setIsPlaying(false); onExpired?.() }
    audio.addEventListener('ended', onEnded)
    audio.addEventListener('error', onErr)
    return () => {
      audio.removeEventListener('ended', onEnded)
      audio.removeEventListener('error', onErr)
      audio.pause()
    }
  }, [getAudio, onExpired])

  const toggle = useCallback(
    (key: string, url: string) => {
      const audio = getAudio()
      if (currentKey === key && isPlaying) {
        audio.pause()
        setIsPlaying(false)
        return
      }
      if (currentKey !== key) {
        audio.src = url
        setCurrentKey(key)
      }
      audio
        .play()
        .then(() => setIsPlaying(true))
        .catch((e: unknown) => {
          // src 교체 직후 빠른 전환 시 play() interrupted(AbortError)는 무시
          if ((e as Error)?.name !== 'AbortError') { setCurrentKey(null); setIsPlaying(false) }
        })
    },
    [getAudio, currentKey, isPlaying],
  )

  const stop = useCallback(() => {
    getAudio().pause()
    setCurrentKey(null)
    setIsPlaying(false)
  }, [getAudio])

  return { currentKey, isPlaying, audio: ref.current, toggle, stop }
}
```

- [ ] **Step 2: 빌드·린트 검증**

Run: `npm run build && npm run lint`
Expected: 그린. (실제 재생 검증은 Task 6에서 오디오 리스트 연결 후.)

- [ ] **Step 3: 커밋**

```bash
git add src/lib/useAudioController.ts
git commit -m "feat(admin): 한 번에 하나 재생하는 useAudioController 훅"
```

---

### Task 5: `MediaGrid` — 이미지/영상 카드 그리드

**Files:**
- Create: `src/components/MediaGrid.tsx`

**Interfaces:**
- Consumes: `mediaKind` (Task 1), `MediaPreview` 아님(그리드는 자체 렌더), `AdminFile`(types), `fileSize`/`dateTime`(lib/format), `useIsMobile`(lib)
- Produces:
  ```ts
  interface MediaGridProps {
    files: AdminFile[]
    onQuarantine: (key: string) => void
    onRestore: (key: string) => void
    onDelete: (key: string) => void
    busy?: boolean
  }
  ```

- [ ] **Step 1: 컴포넌트 작성**

`src/components/MediaGrid.tsx`:
```tsx
import { useState } from 'react'
import { Card, Tag, Button, Popconfirm, Space, Typography, Modal, Empty } from 'antd'
import type { AdminFile } from '../lib/types'
import { mediaKind } from '../lib/mediaKind'
import { fileSize, dateTime } from '../lib/format'
import { useIsMobile } from '../lib/useIsMobile'

interface MediaGridProps {
  files: AdminFile[]
  onQuarantine: (key: string) => void
  onRestore: (key: string) => void
  onDelete: (key: string) => void
  busy?: boolean
}

/** 이미지/영상 카드 그리드. 이미지 클릭=풀사이즈 모달, 영상 ▶=카드 내 인라인 재생. 검역 상태별 액션. */
export default function MediaGrid({ files, onQuarantine, onRestore, onDelete, busy }: MediaGridProps) {
  const isMobile = useIsMobile()
  const [zoomUrl, setZoomUrl] = useState<string | null>(null)
  const [playingKey, setPlayingKey] = useState<string | null>(null)

  if (files.length === 0) return <Empty description="이 타입의 파일이 없어요" />

  const cols = isMobile ? 2 : 4
  return (
    <>
      <div style={{ display: 'grid', gridTemplateColumns: `repeat(${cols}, 1fr)`, gap: 12 }}>
        {files.map((f) => {
          const kind = mediaKind(f.key)
          const name = f.key.replace(/^quarantine\//, '')
          return (
            <Card key={f.key} size="small" styles={{ body: { padding: 8 } }}>
              <div
                style={{ position: 'relative', width: '100%', aspectRatio: '1 / 1', background: '#000', borderRadius: 4, overflow: 'hidden', cursor: 'pointer' }}
                onClick={() => {
                  if (kind === 'image') setZoomUrl(f.url)
                  else if (kind === 'video') setPlayingKey(f.key)
                }}
              >
                {kind === 'image' && <img src={f.url} alt="" style={{ width: '100%', height: '100%', objectFit: 'cover' }} />}
                {kind === 'video' && playingKey === f.key && (
                  // eslint-disable-next-line jsx-a11y/media-has-caption
                  <video src={f.url} controls autoPlay style={{ width: '100%', height: '100%' }} />
                )}
                {kind === 'video' && playingKey !== f.key && (
                  <>
                    <video src={`${f.url}#t=0.1`} muted preload="metadata" style={{ width: '100%', height: '100%', objectFit: 'cover' }} />
                    <span style={{ position: 'absolute', inset: 0, display: 'flex', alignItems: 'center', justifyContent: 'center', color: '#fff', fontSize: 28, textShadow: '0 0 4px rgba(0,0,0,0.7)' }}>▶</span>
                  </>
                )}
              </div>
              <Typography.Text ellipsis style={{ display: 'block', marginTop: 6, fontSize: 12 }} title={name}>
                {name} {f.quarantined && <Tag color="orange">검역</Tag>}
              </Typography.Text>
              <Typography.Text type="secondary" style={{ fontSize: 11 }}>
                {fileSize(f.size)} · {dateTime(f.lastModified)}
              </Typography.Text>
              <Space size={4} style={{ marginTop: 6 }}>
                {!f.quarantined ? (
                  <Popconfirm title="사용자에게 안 보이게 숨길까요?" okText="검역" cancelText="취소" onConfirm={() => onQuarantine(f.key)}>
                    <Button size="small" disabled={busy}>검역</Button>
                  </Popconfirm>
                ) : (
                  <Button size="small" disabled={busy} onClick={() => onRestore(f.key)}>복원</Button>
                )}
                <Popconfirm title="영구 삭제 — 되돌릴 수 없어요" okText="삭제" cancelText="취소" okButtonProps={{ danger: true }} onConfirm={() => onDelete(f.key)}>
                  <Button size="small" danger disabled={busy}>삭제</Button>
                </Popconfirm>
              </Space>
            </Card>
          )
        })}
      </div>
      <Modal open={!!zoomUrl} footer={null} onCancel={() => setZoomUrl(null)} centered width="auto">
        {zoomUrl && <img src={zoomUrl} alt="" style={{ maxWidth: '80vw', maxHeight: '80vh', display: 'block' }} />}
      </Modal>
    </>
  )
}
```

- [ ] **Step 2: 빌드·린트 검증**

Run: `npm run build && npm run lint`
Expected: 그린. (연결은 Task 6.)

- [ ] **Step 3: 커밋**

```bash
git add src/components/MediaGrid.tsx
git commit -m "feat(admin): 이미지/영상 카드 그리드 MediaGrid(확대 모달·인라인 재생·상태별 액션)"
```

---

### Task 6: `FilesPage` 통합 — 타입 탭 + 조건부 뷰 + 오디오 리스트

**Files:**
- Modify: `src/pages/FilesPage.tsx` (탭·필터·조건부 뷰 전면 통합)

**Interfaces:**
- Consumes: `mediaKind`(T1), `MediaPreview`(T3, 이미 연결됨), `useAudioController`(T4), `MediaGrid`(T5), 기존 `ViewTabs`, `getAppFiles`/mutations(기존), `AdminMiniGrid`/`GridPage`(기존), `useIsMobile`.
- Produces: (터미널 화면 — 소비처 없음)

- [ ] **Step 1: import + 타입 탭 상태(?type= 2-way 동기화)**

`src/pages/FilesPage.tsx` import 추가:
```tsx
import { Segmented } from 'antd' // 사용 안 함 — 제거 대상 확인용. 실제로는 ViewTabs 사용:
import ViewTabs from '../components/ViewTabs'
import MediaGrid from '../components/MediaGrid'
import { mediaKind, type MediaKind } from '../lib/mediaKind'
import { useAudioController } from '../lib/useAudioController'
import { Button as AntButton } from 'antd' // 오디오 ▶/⏸ 용(이미 Button import 있으면 재사용)
```
> ⚠️ `Segmented`는 쓰지 않는다(기존 뷰 전환 정본은 `ViewTabs`). 위 줄은 넣지 말 것 — 명시용.

타입 상태를 `useSearchParams`로 2-way 동기화(기존 `?slug=`은 read-once였음 — `?type=`는 되쓰되 slug 보존):
```tsx
const [searchParams, setSearchParams] = useSearchParams()
const activeType = (searchParams.get('type') as MediaKind | 'all' | null) ?? 'all'
const setType = (t: 'all' | MediaKind) => {
  setSearchParams((prev) => {
    const n = new URLSearchParams(prev)
    if (t === 'all') n.delete('type')
    else n.set('type', t)
    return n
  })
}
```

- [ ] **Step 2: prefix→fetch + 클라 type AND 필터 + 건수/절단**

`getAppFiles` 호출에 `max: 1000` 추가하고, 로드된 파일을 타입별로 분류:
```tsx
const filesQuery = useQuery({
  queryKey: ['files', slug, filters.prefix],
  queryFn: () => getAppFiles(slug as string, { prefix: filters.prefix, max: 1000 }),
  placeholderData: keepPreviousData,
  enabled: !!slug,
})
const allFiles = filesQuery.data?.files ?? []
const truncated = filesQuery.data?.truncated ?? false
const counts = useMemo(() => {
  const c = { all: allFiles.length, image: 0, video: 0, audio: 0, other: 0 }
  for (const f of allFiles) c[mediaKind(f.key)]++
  return c
}, [allFiles])
const shownFiles = useMemo(
  () => (activeType === 'all' ? allFiles : allFiles.filter((f) => mediaKind(f.key) === activeType)),
  [allFiles, activeType],
)
// 건수 뱃지: 절단 시 N+ 표기
const badge = (n: number) => (truncated ? `${n}+` : `${n}`)
```

- [ ] **Step 3: 타입 탭 UI (ViewTabs, 건수 라벨) + 절단 경고(모든 탭)**

`GridPage` children 최상단에 탭 + 절단 경고를 넣는다:
```tsx
<div style={{ display: 'flex', flexWrap: 'wrap', gap: 8, alignItems: 'center' }}>
  <ViewTabs<'all' | MediaKind>
    value={activeType === 'other' ? 'all' : activeType}
    onChange={setType}
    options={[
      { label: `전체 ${badge(counts.all)}`, value: 'all' },
      { label: `이미지 ${badge(counts.image)}`, value: 'image' },
      { label: `영상 ${badge(counts.video)}`, value: 'video' },
      { label: `오디오 ${badge(counts.audio)}`, value: 'audio' },
    ]}
  />
</div>
{truncated && (
  <Alert type="warning" showIcon message="일부만 표시됐어요 — prefix로 좁혀주세요. 이 타입에 더 있을 수 있어요." />
)}
```
> `other`는 별도 탭 없이 '전체'에 포함(기타 파일도 전체 테이블에 노출). `activeType==='other'`는 발생하지 않지만 방어적으로 'all' 처리.

- [ ] **Step 4: 조건부 뷰 — 전체=테이블 / 이미지·영상=그리드 / 오디오=리스트**

`shownFiles`를 타입별로 렌더. 전체 탭은 기존 `AdminMiniGrid`(컬럼은 그대로, 단 `rowData={shownFiles}`), 이미지/영상은 `MediaGrid`, 오디오는 아래 리스트:
```tsx
{activeType === 'all' && (
  <AdminMiniGrid<AdminFile> columns={columns} rowData={shownFiles} loading={filesQuery.isLoading} getRowId={(p) => p.data.key} mobileTitleField="key" />
)}
{(activeType === 'image' || activeType === 'video') && (
  <MediaGrid
    files={shownFiles}
    busy={quarantineMutation.isPending || restoreMutation.isPending || deleteMutation.isPending}
    onQuarantine={(k) => quarantineMutation.mutate(k)}
    onRestore={(k) => restoreMutation.mutate(k)}
    onDelete={(k) => deleteMutation.mutate(k)}
  />
)}
{activeType === 'audio' && <AudioList files={shownFiles} onQuarantine={(k) => quarantineMutation.mutate(k)} onRestore={(k) => restoreMutation.mutate(k)} onDelete={(k) => deleteMutation.mutate(k)} onExpired={invalidate} />}
```

- [ ] **Step 5: `AudioList` 서브컴포넌트(같은 파일 하단) — 인라인 재생·경과시간**

`FilesPage.tsx` 하단(또는 `src/components/AudioList.tsx` 분리)에 추가:
```tsx
import { List, Button, Popconfirm, Space, Tag, Empty, Typography } from 'antd'
import { PlayCircleOutlined, PauseCircleOutlined } from '@ant-design/icons'
import { useAudioController } from '../lib/useAudioController'
import { useEffect, useState } from 'react'

function Elapsed({ audio, active }: { audio: HTMLAudioElement | null; active: boolean }) {
  const [t, setT] = useState(0)
  useEffect(() => {
    if (!audio || !active) return
    const on = () => setT(audio.currentTime)
    audio.addEventListener('timeupdate', on)
    return () => audio.removeEventListener('timeupdate', on)
  }, [audio, active])
  if (!active) return null
  const s = Math.floor(t)
  return <Typography.Text type="secondary" style={{ fontSize: 12 }}>{`${Math.floor(s / 60)}:${String(s % 60).padStart(2, '0')}`}</Typography.Text>
}

function AudioList({ files, onQuarantine, onRestore, onDelete, onExpired }: {
  files: AdminFile[]
  onQuarantine: (k: string) => void
  onRestore: (k: string) => void
  onDelete: (k: string) => void
  onExpired: () => void
}) {
  const ctl = useAudioController(onExpired)
  if (files.length === 0) return <Empty description="이 앱에 오디오 파일이 없어요" />
  return (
    <List
      dataSource={files}
      renderItem={(f) => {
        const active = ctl.currentKey === f.key && ctl.isPlaying
        const name = f.key.replace(/^quarantine\//, '')
        return (
          <List.Item
            actions={[
              !f.quarantined ? (
                <Popconfirm key="q" title="사용자에게 안 보이게 숨길까요?" okText="검역" cancelText="취소" onConfirm={() => onQuarantine(f.key)}>
                  <Button size="small">검역</Button>
                </Popconfirm>
              ) : (
                <Button key="r" size="small" onClick={() => onRestore(f.key)}>복원</Button>
              ),
              <Popconfirm key="d" title="영구 삭제 — 되돌릴 수 없어요" okText="삭제" cancelText="취소" okButtonProps={{ danger: true }} onConfirm={() => onDelete(f.key)}>
                <Button size="small" danger>삭제</Button>
              </Popconfirm>,
            ]}
          >
            <Space>
              <Button
                type="text"
                icon={active ? <PauseCircleOutlined /> : <PlayCircleOutlined />}
                onClick={() => ctl.toggle(f.key, f.url)}
              />
              <span>{name}</span>
              {f.quarantined && <Tag color="orange">검역</Tag>}
              <Elapsed audio={ctl.audio} active={active} />
            </Space>
          </List.Item>
        )
      }}
    />
  )
}
```

- [ ] **Step 6: 빌드·린트·목 확인 (핵심 검증)**

Run: `npm run build && npm run lint`
Expected: 그린. 이어서 `npm run dev` 목 모드에서:
- 탭 전환: 전체(테이블)/이미지·영상(그리드 썸네일)/오디오(리스트). URL이 `?type=video` 등으로 변하고, `?slug=`가 있으면 유지되는지 확인.
- 이미지 카드 클릭 → 풀사이즈 모달. 영상 ▶ → 카드 내 재생.
- 오디오 ▶ → 재생, 다른 오디오 ▶ → 이전 정지·새 재생, 클립 종료 시 버튼 ▶로 복귀, 경과시간 표시.
- 검역 파일이 각 탭에서 '복원' 버튼 노출.
- (오디오/영상 재생·시크가 목 SW 경유에서 되는지 Chrome+Safari 중 확인 — 안 되면 spec §9의 SW 우회.)

- [ ] **Step 7: 커밋**

```bash
git add src/pages/FilesPage.tsx src/components/AudioList.tsx
git commit -m "feat(admin): 파일 화면 타입 탭(이미지/영상/오디오) + 그리드/오디오 플레이어 통합"
```

---

## Self-Review (작성자 체크)

- **Spec 커버리지**: §3 mediaKind(T1) · §4 탭/?type=/prefix×type(T6 S1-3) · §5 뷰 모드·확대·절단·빈상태(T5,T6) · §6 오디오(T4,T6 S5) · §7 MediaPreview(T3) · §8 목 샘플(T2) · §9 실모드(모델 무변경, 수동 검증 T6 S6) · §11 검증(build+lint+수동, 게이트 결정 반영). ✅ 전 섹션 태스크 매핑됨.
- **Placeholder**: 없음(모든 코드 실물). ffmpeg 미보유 시 대안(CC0) 명시.
- **타입 일관성**: `mediaKind`/`MediaKind`(T1) → T3/T5/T6 동일 사용. `useAudioController` 반환 타입(T4) → T6 `AudioList` 소비 일치. `MediaGrid` props(T5) → T6 호출 일치.
- **주의**: T6 Step1의 `Segmented`/`AntButton` import 줄은 "넣지 말 것" 명시(혼동 방지). 실제 필요한 import만 사용.
