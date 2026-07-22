# 삼신노트 카드뉴스 — 템플릿형 웹 에디터 (v3)

Claude Code로 작업하는 사람을 위한 프로젝트 가이드. 이 파일을 먼저 읽고, 디자인 토큰은 `design.md`, 과거 실패·수정 이력은 `lesson.md`를 참고하세요.

## 프로젝트 개요

병원·약국 SNS용 **카드뉴스(1080 × 1350px, 4:5)** 를 브라우저에서 직접 편집하고 PNG로 내보내는 **단일 HTML 정적 웹 에디터**입니다. 빌드 도구·번들러·백엔드가 없습니다. `index.html` 하나가 전체 앱입니다.

- 진입점: **`index.html`** (GitHub Pages가 서빙하는 파일, v3 템플릿형 에디터)
- 구버전 보존: `legacy/index.html`(블록엔진 글래스 에디터), `legacy/index2.html`(콘텐츠 변형본), `legacy/삼신노트_퍼고베리스_편집형.html`(최초 원본)
- 외부 의존성: Pretendard 웹폰트, `html2canvas`(PNG 캡처) — 모두 CDN

**v2(블록엔진)와 v3는 다른 앱입니다.** v3는 절대좌표 드래그 블록이 아니라 **고정 템플릿 + contenteditable 필드** 모델입니다. 자유 배치 플로팅 레이어는 Phase 2 예정이며 상태 모델에 `floats[]`로 자리만 예약되어 있습니다. lesson.md의 L1–L17은 legacy 기준 교훈이지만 일반 원칙(L1/2/3/4/7/14/16)은 v3에도 적용됩니다.

## 실행 방법

```bash
python -m http.server 8731
# 브라우저에서 http://localhost:8731/index.html
```

`file://`로 열어도 동작하지만 폰트/캡처/IndexedDB 제약이 있을 수 있어 로컬 서버 권장.

## 핵심 아키텍처 (v3)

### 1. 문서 모델 — schema 2

```js
DOC = {                      // 현재 열려 있는 문서 (전역, 문서 전환 시 재할당)
  schema: 2,
  meta:  { id, title, createdAt, updatedAt },
  cover: { brand, handle, q, sub },              // 표지 (HTML 문자열)
  cards: [ { eyebrow, body } ],                  // 본문 카드 (HTML 문자열)
  images:[ { src, fit, zoom, px, py, overlay } ],// 렌더 순서(표지+본문)별
  theme: { global:{ '--var':'#hex' }, cards:{ '2':{ '--var':'#hex' } } },
  floats:[ [] ],                                 // Phase 2 예약 (자유 배치 요소)
}
```

- 내장 시드 `APP_DATA`(`/*__DATA_START__*/…/*__DATA_END__*/` 마커 안)는 새 문서 템플릿. **모든 로더(부팅/가져오기/복원)는 `migrateDoc()`을 경유**해 v1(구 0722 다운로드 파일, flat theme)도 흡수한다.
- "HTML 파일 받기"는 마커 블록을 현재 `DOC`의 JSON으로 치환한 자기 복제 HTML을 다운로드한다. 그 파일을 "HTML 가져오기"로 다시 읽을 수 있다(왕복).

### 2. run 기반 서식 엔진 (부분 선택 서식의 근본 해결)

서식 적용 시점에만 문단(가장 안쪽 DIV)의 인라인 콘텐츠를 **run 배열**(동일 스타일 텍스트 구간)로 파싱 → 순수 배열 연산으로 변환 → 최소 HTML로 재직렬화. 타이핑은 일반 contenteditable(한글 IME 무영향).

```js
// run: { text|{br}|{atom}, fs, fw, it, un, st, color, hl, ls, chain }
// color 토큰(gold/iv/mute/red/green)은 클래스로 직렬화 → 테마에 반응
```

**불변 규칙 (어기면 과거 버그 재발):**
- **line-height는 절대 스팬에 쓰지 않는다.** 문단 블록에 unitless로만. (`serializeRuns`가 원천 차단)
- 미지의 클래스/속성 요소·**맨 span**(`.chk` 플렉스 래퍼)은 구조 컨테이너로 원형 보존(`chain`). 절대 벗기지 말 것.
- 선택과 "경계만 닿은" 문단·스팬은 건드리지 않는다 (`parasInRange`의 `end > start` 필터).
- 진입점은 `applyInline(patch)`(인라인)·`applyBlockStyle(fn)`(문단: 정렬·줄간격·자간)뿐. DOM Range를 직접 수술하지 말 것.
- cover 필드처럼 호스트 직속에 블록 스타일을 걸어야 하면 `<div data-line>`으로 감싼 뒤 적용 (writeBack이 innerHTML만 저장하므로).

### 3. 테마 — 2계층 CSS 변수

- **에디터 UI는 `--ui-*` 전용** (`--ui-bg/surface/ink/mute/accent/border/danger`). **JS가 절대 변경하지 않는다.** 셸(툴바·패널·토스트)에 캔버스 테마 변수를 쓰지 말 것.
- 캔버스 테마 변수(`--page/--ink/--ivory/--body-ink/--mute/--sub-ink/--gold/--gold-soft/--red/--green`)는 `:root` 기본값 + **JS가 `.stage`(전역)/`.card`(카드별) 인라인으로 오버라이드**. `documentElement`에 쓰면 안 된다.
- 카드별 테마·카드 배경색 = `DOC.theme.cards[i]` 오버라이드 (배경색은 `--ink`).
- 이미지 오버레이 그라데이션은 카드 유효 `--ink`에서 **JS로 rgba 계산**(`applyOverlayColor`) — html2canvas가 `color-mix()`를 못 그리므로 CSS 신문법 금지.

### 4. 저장소 — IndexedDB (`samsin_cardnews`)

- `docs` { id, title, createdAt, updatedAt, thumb, data } / `snapshots` { snapId, docId, ts, label, data }
- **모든 뮤테이터는 `markDirty()`를 호출**한다 → 800ms 디바운스 자동 저장. 새 기능 추가 시 상태를 바꾸는 곳마다 잊지 말 것.
- localStorage에는 마지막 문서 id 포인터(`samsin_last_doc`)만. **이미지 dataURL을 localStorage에 넣지 말 것**(L7 쿼터 사고).
- 스냅샷: 변경 후 5분 경과·전체 내보내기·HTML 받기 시점·수동. 복원 전 "복원 전 자동 백업" 생성. 문서당 30개(공간 60% 초과 시 15개) 유지.
- 업로드 이미지는 `compressImage()`(긴 변 2160px, JPEG q0.85, 알파 시 PNG) 경유.

### 5. UI 패널 규칙

- 플로팅 패널(이모지·테마·색 팝오버·버전)은 **`registerPanel()`로 등록 → `togglePanel()/closePanels()`로만 표시 제어** (L16: 배타 표시 한 곳 관리). 새 패널도 반드시 등록할 것.
- 서식 바 버튼은 `mousedown preventDefault`로 선택 유지 (L4). input 요소는 예외.
- 패널 상단 위치는 `--tbH`(툴바)·`--tbH2`(툴바+서식 바) 기준.

## 코드 구조 (index.html `<script>` 섹션 — 번호 주석 기준)

```
[0]  카드 콘텐츠 + 이미지 상태 — 데이터 시드 (APP_DATA, __DATA_START__ 마커)
[1]  문서 정규화 · 마이그레이션 (migrateDoc / newDocFromTemplate / DOC)
[2]  렌더링 (renderDeck — 재호출 가능)
[3]  편집 반영·편집 모드 (writeBack / editToggle)
[4]  공용 UI (토스트 · syncBars · 패널 관리자 · savedRange 선택 기억)
[5]  서식 엔진 (parseRuns/patchRange/serializeRuns/applyInline/applyBlockStyle)
[6]  실행 취소/다시 실행 (undoStack · commitTyping · IME 가드)
[7]  PPT식 서식 바 (selectionState/updateFmtState · 팝오버 · 단축키)
[8]  이모지 (데이터·패널·삽입)
[9]  색상 테마 (THEME_VARS/PRESETS · 스코프 · applyThemeAll/applyOverlayColor)
[10] 네비게이션 (show/idx)
[11] 카드 이미지 (applyCardImg/loadNat/컨트롤/compressImage)
[12] 미리보기 스케일 (fit)
[13] PNG 캡처 (captureCard(i, scale)/exportOne/exportAll)
[14] HTML 파일로 받기 (buildHtmlBlob/downloadBlob)
[15] 저장소 (openDb/dbGet·Put·Del/markDirty/saveDoc/boot 지원)
[16] 썸네일 (genThumb)
[17] 버전 스냅샷 (snapshotDoc/pruneSnapshots)
[18] 내 작업 패널 (renderHome + 문서 CRUD)
[19] 버전 패널 (renderVerList/복원)
[20] 텍스트로 만들기 (splitCardBlocks/textToDoc/autoFitCard — 붙여넣기 자동 배치)
[21] 부팅 (boot — 저장소에서 문서 복원 후 renderDeck)
```

## 작업 규칙

- **단일 파일 유지.** 별도 번들/프레임워크 도입 금지. 기능 추가는 섹션 번호 주석 구조를 따른다.
- 위 "핵심 아키텍처"의 불변 규칙 4가지 그룹(서식 엔진 / 테마 / 저장소 / 패널)을 지킨다.
- **PNG 캡처 호환 검증.** 새 요소가 캡처(1080×1350, transform 없음) 경로에서 깨지지 않는지, 편집용 UI가 캡처에 찍히지 않는지 확인 (L1/L2).
- **원본 비주얼 보존.** 슬라이드 타이포 CSS(`.cv-title`, `.qtitle`, `.body` 등)는 기존 값 유지. 토큰은 `design.md`.
- 사용자가 수정을 요청했거나 같은 실수가 반복되면 **`lesson.md`에 항목(L22…)을 추가**한다.

## 변경 후 검증 체크리스트

로컬 서버로 띄워 (`http://localhost:8731/index.html`):

1. 콘솔 에러 0
2. **서식**: 레거시 스팬 포함 줄에서 중간 몇 글자만 크기 변경 → 선택 밖 서식 유지 / `.chk` 두 개 가로지른 선택 정상 / 적용·해제 반복해도 HTML 길이 안정(devtools로 중첩 스팬·스팬 line-height 없음 확인) / Ctrl+Z 복원
3. **테마**: 다크 프리셋 적용 → 툴바·패널 색 불변 / 카드별 오버라이드 → 다른 카드 무영향
4. **저장**: 편집 후 1초 내 새로고침 → 복원 / 내 작업에서 문서 전환 / 버전 복원 시 자동 백업 생성
5. **내보내기**: PNG 2160×2700에 편집 UI 미포함 / HTML 받기 → 가져오기 왕복
6. 배포 시: push → Actions green → Pages 루트에 신 에디터, `/legacy/index.html`에 구 에디터

## 배포

`main` 푸시 시 `.github/workflows/deploy.yml`이 저장소 루트 전체를 GitHub Pages로 배포. Settings → Pages → Source = **GitHub Actions**. 자세한 내용은 `README.md`.
