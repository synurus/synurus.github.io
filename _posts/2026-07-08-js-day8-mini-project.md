---
layout: post
title: "[Day 8] 첫 미니 프로젝트 완성 & 배포 — 흑기사 픽셀아트 유튜브 소개 사이트 🚀"
date: 2026-07-08
categories: [TIL, JavaScript, Project]
tags: [미니프로젝트, HTML, CSS, JavaScript, Netlify, localStorage, class, IntersectionObserver]
---

웹개발 수업 8일차. HTML·CSS·JS 과정의 최종 관문인 **미니 프로젝트를 완성하고 배포**했다. 3주간 배운 걸 전부 한 페이지에 녹인 첫 작품 — 형식도 평소 TIL 대신 프로젝트 회고로 남긴다.

## 🎮 완성작: 흑기사(BLACK_KNIGHT) 픽셀아트 유튜브 소개 사이트

**👉 배포 링크: [https://lhw-mini-pr-js.netlify.app/](https://lhw-mini-pr-js.netlify.app/)**

포트나이트 유튜버·스트리머 "흑기사" 채널을 **픽셀아트 세계**로 소개하는 스크롤 사이트다. 흰 배경 + 검은 톤 픽셀아트에, 반짝이는 별과 픽셀 꽃이 깔린 5개의 방(섹션)을 스크롤로 탐험하는 콘셉트. y-n10.com의 스크롤 구성에서 영감을 받았다.

## 1. 기획 — 코딩보다 기획이 먼저

수업에서 배운 대로 **기획서부터** 썼고, 압축 버전을 `index.html` 맨 위 주석으로 박아뒀다 (만들다 헷갈릴 때 보는 나침반 — 실제로 여러 번 봤다).

| 항목 | 내용 |
|------|------|
| 한 줄 소개 | 유튜브 채널을 픽셀아트 세계로 소개하는 스크롤 사이트 |
| 목표 | ① 채널 홍보·구독 유도 ② 흑백 픽셀 브랜딩 ③ 배운 것 전부 녹이기 |
| 구성 | 타이틀 인트로 + 5섹션 (채널/유튜브/포트나이트/스킨 갤러리/방명록) |
| 디자인 장치 | Galmuri 픽셀폰트, 섹션별 꽃 색, 별똥별, 커스텀 픽셀 커서, 다크모드 |

섹션마다 바닥 꽃 색을 다르게(🔴🔵🟢🟡🟣) 정한 게 결과적으로 사이트의 아이덴티티가 됐다.

## 2. 구현 하이라이트 — 배운 것이 어디에 쓰였나

### ⭐ 픽셀 별·별똥별·꽃 동적 생성

HTML에 별을 120개 손으로 적는 대신, JS 반복문으로 랜덤 위치·색·크기의 별을 **동적 생성**했다.

```js
for (let i = 0; i < count; i++) {
  const star = document.createElement('div');
  star.style.left = rand(0, 100) + 'vw';
  star.style.color = pick(STAR_COLORS);   // 배열에서 랜덤 뽑기
  field.appendChild(star);
}
```

배운 것 총동원: `createElement`/`appendChild`(STEP 11 반복문), `Math.random`, 함수로 유틸 분리(`rand`, `pick`). 꽃은 섹션의 `data-flower` 속성을 읽어 색을 CSS 변수(`--c`)로 주입했다 — **JS가 CSS 변수를 조작**할 수 있다는 걸 여기서 처음 써봤다.

### ⭐ 메인 기능: 스킨 갤러리 — class + 3D 뒤집기 + 필터

캐릭터 6명(케데헌 3 + 오버워치 3)을 **클래스로 정의**하고 `new`로 인스턴스 배열을 만들었다 (JS STEP 15).

```js
class Character {
  constructor(id, name, group, videoUrl, desc, img) { ... }
}
const characters = [
  new Character('rumi', '루미', '케데헌', 'https://...', '카리스마 리더 보컬', 'assets/Rumi_Portrait.png'),
  /* ... 6명 */
];
```

- **카테고리 필터** — `characters.filter(c => c.group === filter)` 한 줄 (STEP 13 배열 메서드)
- **3D 뒤집기** — hover 시 `rotateY(180deg)` + `preserve-3d` + `backface-visibility: hidden` (CSS 카드 뒤집기 그대로 응용). 뒷면에 캐릭터 설명이 나온다.
- **클릭 → 영상** — `window.open(char.videoUrl, '_blank')`, 본 스킨은 localStorage에 기록해서 표시

데이터(클래스 배열)와 화면(render 함수)을 분리해두니 필터 기능이 공짜로 생기는 느낌이었다. **render 패턴의 위력을 체감.**

### ⭐ 방명록 — 배열 메서드 총출동

닉네임·별점·메시지 폼(HTML STEP 15·24)에 배운 배열 메서드가 전부 들어갔다:

| 기능 | 메서드 |
|------|--------|
| 새 글 추가 | `push` |
| 목록 렌더 | `map(...).join('')` |
| 최신순 정렬 | `[...entries].sort((a, b) => b.ts - a.ts)` |
| 평균 별점 | `reduce((acc, e) => acc + e.rating, 0)` |
| 삭제 | `filter(e => e.id !== 삭제할id)` |

글자수 카운터는 `input` 이벤트, 저장은 `localStorage` + `JSON.stringify/parse`. 새로고침해도 방명록이 남아있는 걸 보니 뿌듯했다.

### 분위기·편의 기능들

- **밤하늘 다크모드** — `body.night` 클래스 토글로 CSS 변수를 통째로 반전. 5일차 미니 프로젝트에서 배운 ":root 한 곳만 바꾸면 전체가 바뀐다"를 다크모드에 응용한 것. 선택은 localStorage에 저장.
- **스크롤 진행 바 + 맨 위로 버튼** — scroll 이벤트로 진행률 계산
- **커스텀 픽셀 커서** — SVG를 data URI로 만들어 `cursor: url(...)`에 지정
- **`@media` 반응형** — 768px/480px 두 단계로 모바일 대응

## 3. 막혔던 것 & 수업 밖에서 배운 것

만들다 보니 수업 범위를 넘는 문제들을 만났고, 그때마다 찾아 배웠다.

**① 배경음악 자동재생이 안 된다 (브라우저 autoplay 정책)**

`<audio autoplay>`를 넣어도 소리가 안 났다. 알고 보니 브라우저는 **사용자 상호작용 전에 소리 있는 자동재생을 차단**한다. 해결책: 일단 `muted` 상태로 자동재생을 시작하고(음소거면 허용됨), 사용자의 첫 클릭·스크롤에서 음소거를 푸는 방식.

```js
bgm.muted = true;              // 음소거면 자동재생 허용
bgm.play().catch(() => {});
// 첫 상호작용(pointerdown/keydown/scroll)에서 muted = false
```

**② 유튜브 임베드가 로컬에서 안 열린다** — `file://`로 직접 열면 임베드·음악이 막힌다(오류 153). 반드시 **Live Server 같은 로컬 서버로 실행**해야 한다는 것도 이번에 확실히 알았다.

**③ IntersectionObserver** — 섹션이 화면에 들어올 때 등장 애니메이션을 주고 싶었는데, scroll 이벤트로 위치를 매번 계산하는 것보다 `IntersectionObserver`가 표준이라고 해서 적용했다. "요소가 화면과 교차하면 알려줘"라고 등록만 하면 된다.

**④ Web Animations API** — 별똥별은 CSS `@keyframes` 대신 `element.animate([...], {duration, easing})`로 JS에서 직접 랜덤 경로 애니메이션을 만들었다. 매번 다른 방향으로 떨어져야 해서 JS 쪽이 맞았다.

**⑤ escapeHtml — 방명록의 보안 구멍** — 방명록 입력값을 `innerHTML`로 그대로 넣으면 누가 `<script>`를 입력했을 때 코드가 실행될 수 있다(XSS). `<` → `&lt;` 식으로 특수문자를 바꿔주는 `escapeHtml` 함수를 거쳐 출력했다. HTML 1일차에 배운 엔티티가 보안에서 다시 등장!

**⑥ prefers-reduced-motion** — 애니메이션 멀미가 있는 사용자를 위해 OS 설정을 감지해 움직임을 줄이는 미디어 쿼리도 넣었다 (5일차 심화에서 본 "@media는 화면 크기만이 아니다"의 실전 적용).

## 4. 심화 — Claude Code에 Google AI Studio API 키 적용

프로젝트 내내 Claude Code를 조수로 썼는데, 토큰 사용량을 분산하기 위해 Google AI Studio API 키를 연결했다.

```bash
npm install -g @google/gemini-cli
```

**함께 알게 된 것 — CLAUDE.md 파일:**

- `C:\Users\(유저이름)\.claude` 폴더에 `CLAUDE.md` 파일을 추가할 수 있다
- **Claude Code가 실행 시 가장 먼저 읽는 문서**로, 실행 규칙 등을 명시해둘 수 있다
- 예를 들어 "답변은 한국어로", "코드 수정 전 계획부터 보여줘" 같은 나만의 규칙을 적어두면 매번 말하지 않아도 적용된다

도구를 쓰는 걸 넘어 **도구를 내 방식대로 설정**하기 시작한 단계.

## 5. 과제 및 회고

### ✅ 오늘의 과제
- [x] 깃허브 업로드
- [x] 블로그 작성 (지금 이 글!)

### 📌 다음에 할 것
- 확장 아이디어 도전: 스킨 클릭 시 새 탭 대신 **페이지 내 픽셀 모달**로 영상 재생, 스킨 이름 검색창(`includes`), 버튼 효과음
- 스킨 영상 자리표시자 링크를 실제 영상으로 교체
- 다음 과목 진도 대비 복습 — 특히 이번에 처음 쓴 IntersectionObserver, Web Animations API 문서 다시 읽기

### 회고

- **좋았던 점** — 기획서를 먼저 쓰고 "작게 시작 → 기능 하나씩"을 지킨 덕에 길을 잃지 않았다. 별 하나 띄우기 → 별 120개 → 별똥별, 이런 식으로.
- **어려웠던 점** — 수업 예제는 "되는 코드"였는데, 프로젝트는 autoplay 정책·XSS처럼 **현실의 제약**을 처음 만나는 과정이었다. 그런데 그게 제일 많이 배운 부분.
- **뿌듯한 점** — 3주 전엔 `<h1>안녕하세요</h1>` 한 줄이었는데, 지금은 기획하고 만들고 배포한 URL이 있다.

---

**오늘의 한 줄 정리:** 수업은 재료를 주고, 프로젝트는 요리를 시킨다. 그리고 배포 링크가 생기는 순간, 코드는 "내 것"이 된다. 🚀
