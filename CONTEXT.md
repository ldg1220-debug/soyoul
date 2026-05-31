# 소율's 학교 — Claude Code 인계 문서

> 이 파일을 `kids-edu-app.html`과 같은 폴더에 두고,  
> Claude Code 시작 시 "CONTEXT.md 읽고 이어서 작업해줘"라고 하면 돼.

---

## 📁 파일 정보

| 항목 | 내용 |
|------|------|
| 파일명 | `kids-edu-app.html` |
| 크기 | 207KB / 4,189줄 |
| 구조 | 단일 HTML 파일 (CSS + JS + HTML 전부 포함) |
| 외부 의존 | Google Fonts(Jua), Claude API(퀴즈), OpenAI TTS(음성), Pollinations.ai(삽화) |

---

## 🏗️ 화면 구조

```
홈 (연령 선택)
├── 공부하기
│   ├── 한글 퀴즈 (연령별 4단계: k / early / mid / high)
│   ├── 수학 퀴즈 (기본 계산 + 서술형)
│   └── 영어 퀴즈 (ABC / word / phrase / grammar / reading)
└── 놀이터
    ├── 그림그리기 (캔버스 + Undo + 지우개)
    ├── 노래 (계이름 피아노)
    ├── 기억력 카드 (15초 제한)
    ├── 같은 그림 찾기
    └── 동화 (AI 삽화 + TTS 낭독)
```

---

## ✅ 구현 완료 기능

### 1. 메뉴 이동 시 완전 초기화 (Clean Switch)
- **`cleanSweep()`** — 모든 화면 전환 시 호출하는 공통 초기화 함수
  - `window.speechSynthesis.cancel()` — TTS 즉시 중단
  - `storyAutoTimer`, `storyAutoAnimFrame`, `storyReadTimeout`, `memTicker` 전부 제거
  - `#ai-loading` 스피너 숨김
  - `.game-panel`, `.play-panel` 전부 비활성화
- 호출 위치: `goHome()`, `switchHomeTab()`, `openPlay()`, `closePlay()` 첫 줄

### 2. 동화 삽화 시스템
- **`prefetchAllPages(story)`** — `openStory()` 실행 즉시 전 페이지를 `Promise.all`로 동시 생성 (250ms 간격 분산)
- **`loadIllustForPage(story, pageIdx)`** — 캐시 → localStorage → Pollinations.ai 생성 (3회 재시도)
- **`blobToBase64(url)`** — fetch → Blob → FileReader → Base64 변환
- **`saveIllustToStorage(key, dataUrl)`** — `localStorage` 키: `illust_b64_{storyId}_{pageIdx}`
- **`renderIllustration(story, pageIdx)`** — 캐시 히트 즉시 표시, 없으면 로딩 UI
- **`showIllustLoading(el)`** — "🎨 소율이를 위한 그림을 그리는 중이에요..." + CSS 스피너
- **`showIllustImg(el, src, title, pageIdx)`** — opacity 0→1 페이드인
- 이모티콘(`page.img`) 렌더링 완전 제거
- 프롬프트 스타일: `warm soft watercolor storybook illustration, gentle pastel colors, cozy children picture book art, golden dreamy lighting, cute expressive adorable characters, no text no letters no numbers no watermark no emoji no symbols`

### 3. 그림 그리기 업그레이드
- **상태 변수**: `drawUndoStack[]`, `drawEraserMode`
- **`_saveDrawSnap()`** — mousedown/touchstart 시 `getImageData` 스냅샷 (최대 20개)
- **`undoCanvas()`** — `drawUndoStack.pop()` → `putImageData` 복원, 스택 비면 음성 안내
- **`toggleEraser()`** — 지우개 on/off 토글
- **`_syncEraserBtn()`** — `#btn-eraser` 버튼 색상 동기화 (빨강/회색)
- `setColor()` 호출 시 지우개 모드 자동 해제
- HTML 버튼: `#btn-undo` (↩ 뒤로가기), `#btn-eraser` (🩹 지우개), 🗑️ 전체지우기, 💾 저장

### 4. 기억력 카드
- `MEM_PEEK_SEC = {k:15, early:15, mid:15, high:15}` — 전 연령 15초

### 5. 퀴즈 데이터 & 셔플
- **총 145문항**: `FALLBACK` 115개 + `FALLBACK_MATH_WORD` 30개
- **`_shuffleArr(arr)`** — Fisher-Yates 셔플
- **`_getFbPool(poolKey, src)`** — 소진 시 자동 재셔플 캐시 풀
- **`useFallback(subj)`** — 셔플 풀에서 순서대로 출제, 중복 최소화
- FALLBACK 구조: `{ hangul: {k,early,mid,high}, math: {k,early,mid,high}, english: {abc,word,phrase,grammar,reading} }`

---

## 🔑 주요 전역 변수

```js
// 앱 상태
let selectedAge         // 'k' | 'early' | 'mid' | 'high'
let currentSubject      // 'hangul' | 'math' | 'english'
let currentHomeTab      // 'study' | 'play'
let score, streak       // 점수, 연속 정답

// 동화
let currentStory        // STORIES 배열의 현재 항목
let storyPageIdx        // 현재 페이지 인덱스
let storyReading        // TTS 낭독 중 여부
let storyAutoTimer      // 자동 넘김 setTimeout ID
let storyAutoAnimFrame  // 진행바 requestAnimationFrame ID
let storyReadTimeout    // 낭독 시작 setTimeout ID
let illustCache = {}    // 메모리 캐시 { 'storyId_pageIdx': base64DataUrl }

// 그림 그리기
let drawCtx, drawColor, drawSizeVal, isDrawing, lastX, lastY
let drawUndoStack = []  // 스냅샷 배열 (max 20)
let drawEraserMode = false

// 기억력 카드
let memTicker           // setInterval ID

// TTS / 오디오
let openaiApiKey        // OpenAI API 키
let engLevel            // 영어 레벨 'abc'|'word'|'phrase'|'grammar'|'reading'
let mathMode            // 'basic' | 'word'
```

---

## 🎨 CSS 디자인 토큰 (`:root` 변수)

```css
--sky:     하늘색 (버튼 등)
--sun:     노란색 (강조)
--grass:   초록색 (저장 버튼 등)
--coral:   산호색 (포인트)
--purple:  보라색
--navy:    남색 (텍스트)
--card-bg: 카드 배경색
--shadow:  그림자
--radius:  테두리 반경
```

---

## 🗂️ 주요 함수 목록 (95개)

### 네비게이션
`cleanSweep` `goHome` `switchHomeTab` `openPlay` `closePlay` `selectAge`

### 동화/삽화
`openStory` `closeStoryReader` `renderStoryPage` `storyPage`  
`autoReadStory` `toggleStoryRead` `stopStoryRead`  
`buildIllustPrompt` `getIllustUrl` `loadIllustForPage` `prefetchAllPages`  
`renderIllustration` `showIllustLoading` `showIllustImg`  
`saveIllustToStorage` `loadIllustFromStorage` `blobToBase64` `restoreIllustCache`

### 그림 그리기
`initDraw` `drawLine` `setColor` `setSize`  
`_saveDrawSnap` `undoCanvas` `toggleEraser` `_syncEraserBtn`  
`clearCanvas` `saveDrawing` `updateDrawRef` `nextDrawSubject`

### 퀴즈/공부
`startSubject` `generateAIQuestion` `loadAIQuestion`  
`useFallback` `_shuffleArr` `_getFbPool`  
`renderQuestion` `checkAnswer`  
`renderAbcDetail` `showAbcGrid`

### TTS/오디오
`speak` `speakNatural` `speakBrowser` `speakOpenAI`  
`cancelAllAudio` `loadVoices` `selectBestVoices`  
`initSong` `stopAllSongs` `speakLetter`

### 게임
`initMemory` `initPuzzle` `renderPuzzle` `movePuzzle`

### 유틸
`spawnConfetti` `startAutoBar` `stopAutoBar`

---

## 📌 작업 시 주의사항

1. **이모티콘 렌더링 금지** — `page.img`를 화면에 직접 출력하는 코드 추가 금지
2. **메뉴 전환 함수** 수정 시 `cleanSweep()` 첫 줄 호출 유지
3. **퀴즈 추가** 시 `FALLBACK` 구조 (`question`, `choices`, `answer`, `readAloud`, `explanation` 키) 준수
4. **그림 그리기** 수정 시 `drawUndoStack` / `drawEraserMode` 상태 동기화 유지
5. **localStorage 키 prefix**: 삽화 = `illust_b64_`, 설정 = 별도 키
6. 단일 파일 구조 유지 (CSS/JS 분리 금지)
