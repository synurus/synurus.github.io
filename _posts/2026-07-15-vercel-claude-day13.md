---
layout: post
title: "[Day 13] 배포 자동화 & AI로 웹 만들기 — Vercel, 그리고 system prompt"
date: 2026-07-15
categories: [TIL, 배포, AI]
tags: [Vercel, 배포, CI, ClaudeCode, PlanMode, design토큰, GroqAPI, systemprompt]
---

웹개발 수업 13일차. 오전에는 **Vercel 배포** — 드래그 한 번부터 "push = 배포" 자동화까지. 오후에는 **Claude Code로 웹사이트를 설계부터 배포까지** 만드는 워크플로를 배웠다. 심화에서는 어제 만든 Groq 챗봇이 답변에 영어·한자를 섞는 문제를 **system prompt**로 잡았다.

## 1. 수업 내용 요약 ① — Vercel 배포

8일차엔 Netlify로 첫 배포를 해봤는데, 오늘은 Vercel로 배포의 **여러 단계**를 배웠다. 늘 보던 수업 사이트 주소(hancom-nine.vercel.app)가 바로 이걸로 배포된 거였다!

### 1-1. Vercel Drop — 끌어다 놓기만

Git·CLI·설치 전부 없이 **vercel.com/drop**에 폴더나 .zip을 드래그하면 즉시 URL 발급. 프로토타입·일회성 배포에 최적.

⚠️ **한계가 핵심:** 드롭할 때마다 **새 프로젝트·새 URL**이 생긴다 (기존 사이트 갱신 불가 — Netlify Drop과 다른 점). 같은 URL로 업데이트하려면 CLI(`vercel --prod`)나 Git 연결로 갈아타야 한다. Drop으로 시작했어도 대시보드 → Settings → Git에서 나중에 레포를 연결할 수 있다.

### 1-2. CLI 배포 — 터미널 4줄

```bash
npm i -g vercel     # CLI 설치 (최초 1회)
vercel login        # 브라우저 인증 (1회)
vercel              # preview 배포 — 임시 URL (실서비스 영향 없음)
vercel --prod       # production 배포 — 실서비스 URL 갱신
```

- **preview vs production 구분**이 포인트 — `vercel`은 매번 새 임시 URL이라 실험용, `--prod`가 진짜 배포
- ⚠️ **반드시 프로젝트 폴더 안에서 실행** — vercel은 "현재 디렉토리"를 배포 대상으로 잡아서, 밖에서 실행하면 엉뚱한 폴더가 올라간다 (`cd my-app` 후 `package.json` 보이는 위치에서!)

### 1-3. GitHub 연동 — push = 배포 (실무 표준)

레포를 Vercel에 연결하면 **`git push`할 때마다 자동 재배포**. 이제 배포 명령 자체가 사라진다.

```bash
git push   # ← 이게 곧 배포 명령
```

연결법: 프로젝트 대시보드 → Settings → Git → Connect Git Repository. 주의할 것들:

- **production 브랜치는 기본 `main`** — 작업 브랜치가 다르면 push해도 배포 안 됨 (Settings → Environments → Production에서 변경, 또는 main에 머지)
- `.gitignore`에 `node_modules · dist · .vercel` 확인 (`.vercel/` 폴더는 연결 정보라 깃에 올리면 안 됨)
- 코드 수정분 반영은 **push 먼저** — 대시보드의 Redeploy는 그 커밋 그대로 재빌드일 뿐

### 1-4. Claude 연동 — "푸시해줘" 한마디로 배포

GitHub ↔ Vercel이 연결된 상태에선 Claude Code에게 코드 수정 후 **"깃허브에 푸시해줘"** 한마디면 끝:

```
① Claude에게 코드 수정 요청
② "깃허브 푸시해줘"
③ Claude가 git add · commit · push 실행
④ Vercel이 push 감지 → 자동 재배포 → 1~2분 후 URL 반영
```

핵심: **배포 명령을 따로 안 한다.** push = 배포라는 파이프라인 덕분. (⚠️ 커밋 author 이메일이 Vercel 계정과 달라 배포가 안 되면 `git config user.email` 확인.)

> 11일차 프로젝트 때 Netlify에서 `_redirects`까지 손수 만졌던 걸 떠올리면, 배포가 "이벤트"에서 "일상"으로 바뀌는 과정을 일주일 사이에 다 경험한 셈이다.

## 2. 수업 내용 요약 ② — Claude Code로 웹사이트 만들기

HTML·CSS·JS를 한 줄도 직접 안 짜고, 대화로 **설계 → 생성 → 리뷰 → 배포**까지 가는 워크플로.

### 2-1. 워크플로 3단계 — Plan Mode부터

**① Plan Mode 설계 (Shift+Tab)** — 바로 코딩시키지 않고 Plan Mode로 요구사항을 정리하면, Claude가 구현 플랜을 먼저 제시하고 승인 후에 코드를 만든다. 큰 작업일수록 효과적. (4일차에 폴더 자동화 스크립트 만들 때 썼던 그 플랜 모드!)

```
> 포트폴리오 랜딩 페이지 만들어줘
  - 히어로 섹션 + 소개 + 프로젝트 카드 3개 + 연락처 푸터
  - 단일 HTML, CSS·JS 인라인, 모바일 반응형
```

**② 자연어로 부분 수정** — "히어로 배경을 다크 그라디언트로", "카드에 hover 애니메이션 추가", "모바일에서 햄버거 메뉴로" — 콕 집어 지시.

**③ 스크린샷·리뷰 반복** — 브라우저 결과를 스크린샷으로 첨부해 수정 지시 → 반복 개선.

### 2-2. frontend-design 스킬 — "예쁘게"라고 말하지 않기

Claude의 공식 UI 디자인 스킬. 새 UI를 부탁하면 자동으로 켜지고, 코드를 짜기 **전에** 4가지를 먼저 정한다:

1. **색** — 4~6개 named hex 팔레트
2. **타이포** — 디스플레이 + 본문 + 유틸 서체
3. **레이아웃** — ASCII 와이어프레임
4. **시그니처** — 이 페이지만의 기억에 남을 요소 **하나**

그리고 그 계획이 "어디서나 나올 법한 기본값"은 아닌지 스스로 비평한 뒤 코드를 쓴다. 막막한 "예쁘게 만들어줘" 대신 **시그니처 하나를 정하고 나머지를 절제**하는 접근이 인상적이었다.

브리프를 구체적으로 줄수록 결과가 좋다:

```
> 커피 원두 구독 서비스 랜딩 페이지 만들어줘
  - 대상: 원두 마니아, 페이지 목적: 구독 전환
  - tokens.css: 팔레트·글씨·간격을 :root 변수로 (색·치수의 단일 출처)
  - styles.css: var(--token)만 사용, 색 코드 직접 박지 말 것
  - index.html: 시맨틱 구조만
```

**토큰 시스템 = CSS 변수 철학의 완성형.** 색·간격을 `:root` 한곳에 모으면(단일 출처) 한 곳만 고쳐도 사이트 전체가 바뀐다 — 5일차 CSS 변수부터 다크모드까지 계속 쓰던 그 원리가 "디자인 시스템"이라는 이름으로 정리됐다.

### 2-3. design.md — 디자인을 글로 적은 설계도

**design.md** = 한 브랜드의 색·글씨·간격·부품을 정리한 디자인 명세서. 이 설계도만 건네면 Claude가 같은 무드의 사이트를 통째로 조립한다.

- [getdesign.md](https://getdesign.md/) — design.md 형식 명세 갤러리
- [getdesign.kr](https://getdesign.kr/) — 토스·배민·클래스101 등 한국 서비스 14개의 디자인 시스템을 같은 포맷으로 정리한 한국판

실습은 BMW M 스타일 설계도로: 새까만 #000 캔버스, 파랑·남색·빨강 **M 삼색 띠** 포인트, 제목 굵은 대문자(700)·본문 얇게(300), 모서리 직각(radius 0) — 이 명세를 주고 `bmw-tokens.css + bmw-styles.css + bmw.html` 3종을 뽑았다. **디자인이 코드가 아니라 "문서"로 관리될 수 있다**는 게 오늘의 발견.

## 3. 어려웠던 것 — 심화 학습

### system prompt — 챗봇에게 "대화 내내 지킬 규칙" 심기

어제 만든 Groq 챗봇이 답변에 영어·한자를 섞어 쓰는 문제가 있었다. 해결책: **messages 배열 맨 앞에 `role: 'system'` 메시지 추가.**

```js
body: JSON.stringify({
  model: 'llama-3.1-8b-instant',
  messages: [
    // system으로 한국어 응답 강제
    { role: 'system', content: '반드시 한국어로만 답변하라. 영어를 제외한 다른 언어나 문자를 절대 섞지 마라.' },
    { role: 'user', content: req.body.prompt }
  ]
})
```

**정리한 내용:**

- OpenAI 호환 API(Groq도 동일 스펙)에서는 messages 배열의 **첫 항목**으로 `role: 'system'`을 넣으면, 모델이 **대화 내내 그 지침을 우선 규칙**으로 따른다. user 메시지가 "질문"이라면 system 메시지는 "역할·규칙 설정".
- **"하지 마라"만으로 부족하면 더 구체적으로** — "출력에 한글(가-힣)과 기본 문장부호만 사용하라"처럼 허용 범위를 명시하면 한자 혼입이 더 줄어든다. 금지보다 허용 목록이 강력하다.
- **작은 모델의 한계와 이중 방어** — llama-3.1-8b-instant 같은 작은 모델은 system 지시를 가끔 무시한다. 완전히 막고 싶으면 응답을 받은 뒤 **정규식으로 한자 포함 여부를 검사해 재요청**하는 방법도 있다:

```js
if (/[\u4e00-\u9fff]/.test(reply)) {   // 한자(CJK) 유니코드 범위 검사
  // 한자가 섞였으면 재요청
}
```

> `[\u4e00-\u9fff]` = 한자 블록의 유니코드 범위. 12일차 배치도에서 한글 검사에 쓴 `/[가-힣]/`과 같은 원리다 — 문자 범위를 정규식으로 지정하는 패턴이 벌써 두 번째 등장.

**느낀 점:** 프롬프트로 지시(1차 방어) + 코드로 검증(2차 방어). AI의 출력도 결국 "검증해야 할 외부 데이터"로 다루는 게 안전하다 — 서버에서 입력값 검증하던 것과 같은 자세다.

## 4. 과제 및 다음에 할 것

### ✅ 오늘의 과제
- [x] 깃허브 업로드
- [x] 블로그 작성 (지금 이 글!)

### 📌 다음에 할 것
- K-POP 차트 앱을 Vercel + GitHub 연동으로 재배포해보기 — push = 배포 파이프라인 직접 구축
- getdesign.kr에서 마음에 드는 design.md 하나 골라 내 블로그 스타일에 응용해보기
- 챗봇에 system prompt 페르소나 실험 — "픽셀 게임 NPC처럼 말하는 챗봇" 같은 것
- 지금까지의 흐름 정리: 프론트(React) + 백엔드(Express) + AI(Groq) + 배포(Vercel) — 풀스택 조각이 다 모였다!

---

**오늘의 한 줄 정리:** 배포는 push 한 번으로, 디자인은 문서(design.md) 한 장으로, AI의 말투는 system 한 줄로 — 오늘 배운 건 전부 "한 곳에서 통제하는 법"이었다.
