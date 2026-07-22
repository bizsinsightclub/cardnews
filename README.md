# 삼신노트 카드뉴스 — 템플릿형 웹 에디터 (v3)

병원·약국 SNS용 **카드뉴스(1080 × 1350px)** 를 브라우저에서 편집하고 PNG로 내보내는 단일 HTML 웹 에디터입니다. 빌드 과정 없이 `index.html` 하나로 동작합니다.

## ✨ 기능
- ✏️ **PPT식 서식 툴바** — 크기 입력·가▲▼ 스테퍼, 굵게/기울임/밑줄/취소선, 형광펜·글자색(테마 스와치 + 커스텀), 정렬·줄간격·자간, 빠른 스타일 프리셋 6종
- 🧠 **run 기반 서식 엔진** — 부분 선택 서식이 주변 서식을 깨뜨리지 않음, 문단 가로지르는 선택 지원, Ctrl+Z/Y 실행 취소(한글 IME 안전)
- 🎨 **색상 테마** — 프리셋 6종 + 역할별 색(배경/글자/포인트/기호), **🌍 전체 적용 / 🎴 이 카드만** 스코프, 테마가 에디터 UI를 침범하지 않음
- 🖼️ 카드별 배경 이미지(확대·가로·세로·오버레이, 업로드 시 자동 압축) + 😊 이모지 검색 삽입
- 💾 **실시간 자동 저장 (IndexedDB, 이미지 포함)** — 새로고침해도 유지
- 📁 **내 작업** — 여러 카드뉴스를 썸네일 목록으로 저장·전환·복제·이름 변경
- 🕘 **버전 기록** — 5분 주기·내보내기 시점 자동 스냅샷, 언제든 과거 시점으로 복원(복원 전 자동 백업)
- 📥 카드별 / 전체 **PNG 저장** (1080×1350, 2x = 2160×2700)
- 📄 **HTML 파일 받기/가져오기** — 데이터가 구워진 자기 완결 HTML로 백업·이동

## 🚀 로컬 실행
```bash
python -m http.server 8731
# http://localhost:8731/index.html 접속
```
> `file://`로 바로 열어도 대부분 동작하지만, 폰트·PNG 캡처·저장소 안정성을 위해 로컬 서버를 권장합니다.

## 🌐 GitHub Pages 배포 (자동)
1. 이 폴더를 GitHub 저장소로 push.
2. 저장소 **Settings → Pages → Build and deployment → Source** 를 **GitHub Actions** 로 설정.
3. `main` 브랜치에 push하면 `.github/workflows/deploy.yml` 이 자동으로 사이트를 배포합니다.
4. 배포 URL: `https://<사용자명>.github.io/<저장소명>/` (구형 에디터: `…/legacy/index.html`)

## 📁 구조
```
index.html        v3 템플릿형 에디터 (배포 진입점)
legacy/           구버전 보존 (블록엔진 글래스 에디터 index.html · index2.html · 최초 원본)
CLAUDE.md         Claude Code 작업 가이드 (아키텍처·작업 규칙)
design.md         디자인 토큰/UI 명세 (--ui-* 셸 / 캔버스 테마 2계층)
lesson.md         실패·수정 이력 (L1–L17 legacy · L18+ v3)
.github/workflows/deploy.yml   Pages 자동배포 (저장소 루트 전체)
```

## 🛠 기술
순수 HTML/CSS/JS 단일 파일. 저장은 IndexedDB(문서·스냅샷) + localStorage(마지막 문서 포인터). 외부 의존성은 CDN 2개 — `html2canvas`(PNG 캡처), Pretendard 웹폰트.
