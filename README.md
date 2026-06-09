# 삼신노트 카드뉴스 — 글래스 웹 에디터

병원·약국 SNS용 **카드뉴스(1080 × 1350px)** 를 브라우저에서 편집하고 PNG로 내보내는 단일 HTML 웹 에디터입니다. 빌드 과정 없이 `index.html` 하나로 동작합니다.

## ✨ 기능
- 🖱️ **텍스트 자유 드래그 이동** (PPT 느낌, snap 정렬선)
- 🔠 **폰트 크기 조절** (블록 단위 / 드래그 선택 단위)
- 🟧 **도형 추가·삭제 + 내부 텍스트 편집** (비교카드·체크리스트·강조박스 등 9종)
- ✏️ **드래그하면 뜨는 서식 툴바** — 굵게·기울임·밑줄·하이라이트·글자색·크기
- 🎨 **그림판식 hex 컬러 팔레트** — 슬라이드 배경 / 하이라이트 / 글자색 / 도형 배경
- 🖼️ 슬라이드별 배경 이미지(줌·크롭·채움전환)
- ➕ 슬라이드 추가·복제·삭제
- 💾 **localStorage 자동 저장** (새로고침해도 유지, `↺ 초기화` 제공)
- 📥 슬라이드별 / 전체 **PNG 저장** (1080×1350, 2x)

## 🚀 로컬 실행
```bash
python -m http.server 8731
# http://localhost:8731/index.html 접속
```
> `file://`로 바로 열어도 대부분 동작하지만, 폰트·PNG 캡처 안정성을 위해 로컬 서버를 권장합니다.

## 🌐 GitHub Pages 배포 (자동)
1. 이 폴더를 GitHub 저장소로 push.
2. 저장소 **Settings → Pages → Build and deployment → Source** 를 **GitHub Actions** 로 설정.
3. `main` 브랜치에 push하면 `.github/workflows/deploy.yml` 이 자동으로 사이트를 배포합니다.
4. 배포 URL: `https://<사용자명>.github.io/<저장소명>/`

```bash
git init && git add . && git commit -m "init: 카드뉴스 글래스 에디터"
git branch -M main
git remote add origin https://github.com/<you>/<repo>.git
git push -u origin main
```

## 📁 구조
```
index.html        개선된 에디터 (배포 진입점)
삼신노트_퍼고베리스_편집형.html   원본(개선 전) 보존본
CLAUDE.md         Claude Code 작업 가이드
design.md         에디터 디자인 시스템/UI 명세
lesson.md         실패·수정 이력 (lessons learned)
.github/workflows/deploy.yml   Pages 자동배포
```

## 🛠 기술
순수 HTML/CSS/JS. 외부 의존성은 CDN 2개 — `html2canvas`(PNG 캡처), 웹폰트(Pretendard·IBM Plex Mono·Nanum Myeongjo).
