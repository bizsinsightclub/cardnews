# 삼신노트 카드뉴스 — 글래스 웹 에디터

Claude Code로 작업하는 사람을 위한 프로젝트 가이드. 이 파일을 먼저 읽고, 디자인은 `design.md`, 과거 실패·수정 이력은 `lesson.md`를 참고하세요.

## 프로젝트 개요

병원·약국 SNS용 **카드뉴스(1080 × 1350px, 4:5)** 슬라이드를 브라우저에서 직접 편집하고 PNG로 내보내는 **단일 HTML 정적 웹 에디터**입니다. 빌드 도구·번들러·백엔드가 없습니다. `index.html` 하나가 전체 앱입니다.

- 진입점: **`index.html`** (GitHub Pages가 서빙하는 파일)
- 원본 보존본: `삼신노트_퍼고베리스_편집형.html` (개선 전 버전, 참고용)
- 외부 의존성: Pretendard/IBM Plex Mono/Nanum Myeongjo 웹폰트, `html2canvas`(PNG 캡처) — 모두 CDN

## 실행 방법

정적 파일이라 그냥 열면 되지만, `file://`에서는 일부 브라우저가 폰트/캡처를 제한할 수 있으므로 로컬 서버 권장:

```bash
python -m http.server 8731
# 브라우저에서 http://localhost:8731/index.html
```

## 핵심 인터랙션 모델 (v2 — 중요)

- **모든 블록은 절대좌표(absolute).** flow/free 이원 모델은 폐기됨. 최초 1회 `bakeLayout()`가 원본 레이아웃을 측정해 `x,y,w`를 굳히고, 이후 모든 배치는 좌표 기반. (이전 버전의 "이동 시 위계 깨짐·클릭 불가" 버그 원인 → `lesson.md` L8 참고)
- **클릭 = 선택/이동, 더블클릭 = 편집.** 비편집 블록은 `contenteditable=false` + `user-select:none`(드래그=이동), 더블클릭 시에만 편집 진입, 바깥 클릭 시 종료. (`lesson.md` L9)
- **한 장씩(페이지형) 뷰.** `cur` 인덱스의 슬라이드만 렌더. 상단 페이저·하단 필름스트립으로 이동.
- **브랜드/핸들(페이지번호)/진행점(dots)도 일반 블록** → 이동·삭제·편집 가능.

## 기능 (요구사항 대응)

| 기능 | 구현 위치(섹션 주석) | 메모 |
|---|---|---|
| 텍스트 자유 드래그 이동 | `3. 선택/편집/이동/크기` | 절대좌표 직접 갱신. snap 정렬선(중앙540·안전62/1018·상하62/1288) |
| 폰트 크기 편집 | 블록바 `A−/A+`, 선택툴바 `A−/A+` | 블록 단위 = `fontPx`, 인라인 선택 = span 래핑 |
| 도형 안 텍스트 넣기/빼기 | `6. 도형 추가` + 더블클릭 편집 | 좌측 독 `◇도형`으로 추가, 블록바 🗑 삭제, 내부는 더블클릭 편집 |
| 선택 서식 툴바(드래그 시 위에 뜸) | `4. 선택 서식 툴바` | 편집 중 글자 선택 시 출현. B·I·U·하이라이트·글자색·크기(인라인)·**자간/줄간격(박스 전체)**. `execCommand` + `adjustBoxStyle` |
| 자간/줄간격 | 선택툴바 `AV±` / `≡±` | 블록 단위. `b.letterSpacing`(em)·`b.lineHeight`(비율) 모델 저장·복원. 줄간격은 인라인 불가하여 블록 적용 |
| 그림판식 hex 팔레트 | `5. 컬러 팔레트` | 스와치 + `<input type=color>` + hex. 배경/하이라이트/글자색/도형배경 공용 |
| 슬라이드 배경색 | 독 `🎨 배경` | `sl.bg` (null=기본 그라데이션) |
| 이미지 편집(플로팅 패널) | `7. 이미지 패널` | 독 `🖼 이미지` → 옆에 뜨는 패널에서 줌·가로·세로·채움·제거 |
| 슬라이드 추가/복제/삭제 | 독 `＋/⧉/🗑` | |
| 페이지 이동 | `9. 페이저/필름스트립/독` | ‹ n/N › + 번호 칩 |
| 자동 저장 | `1. 저장소` localStorage | 키 `samsin_cardnews_v2`, 250ms 디바운스. 독 `↺ 초기화`로 리셋 |
| PNG 저장 | `8. PNG 저장` | html2canvas, 1080×1350 고정, scale 2. 전체저장은 페이지 순회 |

## 코드 구조 (index.html `<script>` 내 섹션)

순서대로 번호 주석이 달려 있습니다. 수정 시 해당 섹션만 보세요.

```
0.  DEFAULT / SHAPES / SWATCHES / TYPE_FONT       — 데이터·상수
1.  저장소 + 레이아웃 베이크 (preState/load/save/bakeLayout)
2.  렌더 (현재 슬라이드 한 장: render/buildBlock)
3.  선택·편집·이동·크기 (selectBlock/enterEdit/exitEdit/startDrag/wireBlock)
4.  선택 서식 툴바 (updateFmt/execCommand/stepFontSize)
5.  컬러 팔레트 (openPalette)
6.  도형 추가 / 텍스트 추가
7.  이미지 패널 (syncImagePanel/applyImg)
8.  PNG 저장 (savePng/saveAll)
9.  페이저·필름스트립·좌측 독 (setIndex/dock click 라우팅/슬라이드 CRUD)
10. 시작 (init: 폰트 로드 후 bake → render)
```

### 상태(state) 모델 (v2)

```js
state = { _baked:true, _guides:true, slides:[ {
  id, name, cover, bg:null|'#hex', image:null|{src,fit,zoom,px,py},
  blocks:[ { id, type, html, x, y, w, fontPx:null, bg:null } ]   // 전부 절대좌표
} ] }
// cur = 현재 보는 슬라이드 인덱스(전역)
```

- `type`은 그대로 CSS 클래스명: `brand`,`handle`,`dots`,`hcover`,`qhead`,`h2`,`qsub`,`bd`,`eyebrow`,`ratio`,`eq`,`vsrow`,`checks`,`disc`,`brandline`.
- 베이크 전 임시 상태(`preState`)는 `_brand/_handle/_dots/_content`를 들고 있다가, `bakeLayout()`이 측정 후 단일 `blocks[]`로 합치고 `_baked=true`로 표시한다. 저장된 state가 `_baked`면 베이크를 건너뛴다.
- 텍스트는 `html`(innerHTML)로 저장 → 새로고침 복원.

## 작업 규칙

- **단일 파일 유지.** 별도 번들/프레임워크 도입 금지. 기능 추가는 `index.html` 섹션 주석 구조를 따른다.
- **절대좌표 모델 유지.** 슬라이드 전체를 덮는 클릭 가로채기 레이어를 만들지 말 것(L8). 좌표 계산엔 항상 `/VIEW()` 보정(L3).
- **PNG 캡처 호환 검증.** `--view` 축소를 쓰므로 새 요소가 `body.exporting` 경로(transform none, 1080×1350)에서 깨지지 않는지 확인.
- **원본 비주얼 보존.** 슬라이드 컴포넌트 CSS(`.qhead`,`.vscol` 등)는 원본 값 유지. 디자인 토큰은 `design.md`.
- **변경 후 검증:** 로컬 서버로 띄워 ① 콘솔 에러 0 ② 베이크 후 단일 슬라이드 렌더 ③ 블록 이동→재클릭 가능 ④ 더블클릭 편집→재클릭 가능 ⑤ PNG 저장까지 확인.
- 사용자가 수정을 요청했거나 같은 실수가 반복되면 **`lesson.md`에 항목(L14…)을 추가**한다.

## 배포

`main` 푸시 시 `.github/workflows/deploy.yml`이 GitHub Pages로 자동 배포. 저장소 Settings → Pages → Source를 **GitHub Actions**로 설정하면 됨. 자세한 내용은 `README.md`.
