---
layout: post
title: "[Day 12] 백엔드 입문 — Express 서버 만들기 & Groq API로 AI 챗봇 연결"
date: 2026-07-14
categories: [TIL, Node, Backend]
tags: [Node, Express, REST, CRUD, 미들웨어, CORS, GroqAPI, dotenv, HSL]
---

웹개발 수업 12일차. 지금까지는 화면(프론트)을 만들었다면, 오늘은 **반대편 — 서버(백엔드)**를 처음 만들었다. Node·Express로 CRUD API 서버를 직접 세우고, 오후에는 Groq AI API를 연결해 나만의 챗봇까지. 7일차에 json-server로 "빌려 쓰던" 서버를 오늘 직접 만든 셈이다.

## 1. 수업 내용 요약 ① — Node · Express

### 1-1. Node.js — 브라우저 밖에서 JS 실행

Node.js = 크롬의 **V8 엔진**을 떼어내 서버·터미널에서도 JS를 돌리는 런타임. 비유: **브라우저 JS = 홀, Node = 주방.**

**왜 CRUD는 서버에서 하나? (핵심 질문)**
- 브라우저 JS는 파일·서버·DB 접근 불가 (보안)
- 브라우저 코드는 누구나 F12로 열람 → 거기에 DB 비밀번호·API 키를 두면 다 털림
- → ① **보안**(키·검증은 서버만) ② **공유**(여러 사람이 같은 데이터) 때문에 진짜 데이터 작업은 서버에서

```bash
node -v          # 버전 확인
npm init -y      # package.json 생성
node index.js    # 실행
```

### 1-2. Express — 서버를 쉽게 만드는 프레임워크

순수 Node의 `http` 모듈로도 서버는 만들 수 있지만, 주소마다 `if (req.url === ...)` 분기가 늘어난다. Express는 그걸 **`app.get('/주소')` 한 줄**로:

```js
// 순수 Node — 라우트마다 else if 한 덩이…
// Express — 주소마다 한 줄씩 나란히
app.get('/', (req, res) => res.send('Home'))
app.get('/about', (req, res) => res.send('About'))
app.listen(3000)
```

**용어 한 줄 정리:** **Node.js**(런타임) 위에서 **HTTP**(프로토콜)로 통신하는 백엔드 서버를 **Express**(프레임워크)로 쉽게 만든다.

### 1-3. 최소 서버 구조 — 모든 서버의 4박자

```js
const express = require('express')   // 1. 꺼내고
const app = express()                // 2. 만들고 (서버 본체)
app.get('/', (req, res) => {         // 3. 규칙 주고 (라우트)
  res.send('Hello, Server!')
})
app.listen(3000)                     // 4. 문 연다 (없으면 안 켜짐)
```

- **req** = 브라우저가 보낸 요청 / **res** = 서버가 돌려주는 응답
- 3000 = **포트 번호** (컴퓨터 안의 문 번호) — 7일차에 json-server가 쓰던 그 3000!

### 1-4. GET 두 가지 — 고정 주소 vs 변하는 주소

```js
// ① 고정 주소 → 전체 목록
app.get('/api/users', (req, res) => {
  res.json([{ id: 1, name: '지니' }, { id: 2, name: '철수' }])
})

// ② :id 변하는 주소 → 특정 1개 (/api/users/3 → 영희)
app.get('/api/users/:id', (req, res) => {
  const user = users.find(u => u.id === Number(req.params.id))
  if (!user) return res.status(404).json({ error: '없는 유저' })
  res.json(user)
})
```

- `:id` 자리 = 변하는 값, `req.params.id`로 꺼냄 — **단, 문자열이라 `Number()` 변환** (JS 5일차 타입 감각이 여기서!)
- 못 찾으면 404 응답 — React 프로젝트에서 fetch할 때 받던 그 404를 이제 **내가 보내는 입장**

### 1-5. POST — 보내고 받기, 그리고 LAN 실습

```js
app.use(express.json())              // 必 — 보낸 JSON을 req.body로 풀어줌

app.post('/api/chat', (req, res) => {
  const { message } = req.body       // 받기
  res.json({ ok: true, 받은문장: message })   // 답장 (영수증)
})
```

**왕복 구조 = 모든 웹앱의 기본 단위:** 프론트가 body로 보냄 → 서버가 `req.body`로 꺼냄 → `res.json` 답장 → 프론트가 `.then`으로 받음. **Claude 웹에서 프롬프트 보내는 것도 완전히 같은 POST 왕복**이라는 게 인상적이었다.

**🤝 옆자리 실습** — 짝과 서버/클라이언트 역할을 나눠, `ipconfig`로 IP 확인하고 방화벽 3000 포트를 열어 **다른 컴퓨터의 서버로** 메시지를 보냈다. localhost가 아닌 진짜 네트워크 통신 첫 경험!

### 1-6. PUT · DELETE — CRUD 완성

```js
app.put('/api/users/:id', (req, res) => {     // 수정 — 찾아서 교체
  const u = users.find(u => u.id === Number(req.params.id))
  if (!u) return res.status(404).json({ error: '없는 유저' })
  u.name = req.body.name
  res.json(u)
})

app.delete('/api/users/:id', (req, res) => {  // 삭제 — 그 id만 빼고 새 배열
  users = users.filter(u => u.id !== Number(req.params.id))
  res.json({ ok: true, 남은: users })
})
```

7일차에 json-server 상대로 **날리던** GET/POST/PUT/DELETE 요청을, 오늘은 전부 **받는 쪽**에서 구현했다. `find`(찾기)·`filter`(빼기) — 배열 메서드가 서버에서도 주인공.

### 1-7. 미들웨어 · CORS — 건물 입구의 검색대 ★

**미들웨어** = 모든 요청이 라우트에 도착하기 **전에** 거쳐가는 공통 처리 칸. `app.use`로 등록하며, **라우트보다 위에** 둔다.

```js
app.use(cors())            // 검색대1 — 외부 손님(다른 포트) 출입 허용
app.use(express.json())    // 검색대2 — 가져온 짐(body) 풀어서 정리
app.use((req, res, next) => {
  console.log(req.method, req.url)   // 검색대3 — 방문 장부 기록
  next()                             // 다음으로 통과! (안 부르면 여기서 멈춤)
})
```

**CORS의 정체를 드디어 이해했다.** 브라우저는 출처(프로토콜+주소+포트)가 다른 요청을 기본 차단한다 — 화면(5173) → 서버(3000)는 포트가 달라서 막히는 것. 11일차 프로젝트에서 Apple 피드에 막혀 **프록시로 우회했던 그 CORS**를, 오늘은 서버 주인 입장에서 `cors()`로 **열어주는** 법을 배웠다. 같은 문제의 양쪽 면을 다 본 셈. (연습은 전체 허용, 실무는 `cors({ origin: '내주소' })`로 내 사이트만.)

### 1-8. CRUD 최종 실습 — 우리 반 배치도 만들기 ★

강사 서버(`Authorization: HANCOM` 헤더 필수)에서 우리 반 20명 데이터를 받아오는데, **데이터가 오염되어 있다** — 잘못된 이름, 중복, 누락, 불필요한 항목. 할 일:

1. fetch(GET)로 현재 서버 데이터 조회 → 정상 명단과 비교
2. **PUT으로 수정, DELETE로 삭제, POST로 추가** — 오염 정정
3. 정리된 20명을 HTML 그리드 배치도(4×5)로 표시, 이름 클릭 → prompt → PUT 수정까지

오전에 배운 CRUD 4종이 전부 실전 투입되는 과제. "데이터는 늘 오염되어 있다"는 11일차 교훈(공개 API 가공)이 하루 만에 재등장했다.

## 2. 수업 내용 요약 ② — Groq API로 AI 연결

### 2-1. 키 발급 — .env와 .gitignore

Groq = 무료(카드 등록 없이)로 AI를 코드에서 부르는 통로. console.groq.com에서 `gsk_`로 시작하는 API 키를 발급받는다 (생성 직후 한 번만 보임!).

```
# .env — 서버에만 두는 비밀 파일
GROQ_API_KEY=gsk_여기에-본인-키

# .gitignore — 깃에 안 올릴 목록
.env
node_modules/
```

**키 = 출입증.** 깃·브라우저에 노출되면 남이 내 몫을 대신 써버린다. 실수로 깃허브에 올렸다면? 그 키는 버린 셈 — 즉시 Revoke하고 새로 발급.

### 2-2. 기본 호출 — 주소 · 헤더 · 내용 3칸

AI에 말 거는 법 = 택배 보내기: **어디로(주소) · 누구인지(키) · 뭐라고(messages).**

```js
require('dotenv').config()
const key = process.env.GROQ_API_KEY

const groqRes = await fetch('https://api.groq.com/openai/v1/chat/completions', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': 'Bearer ' + key        // "나 사용 권한 있어요" 증명
  },
  body: JSON.stringify({
    model: 'llama-3.1-8b-instant',           // 쓸 AI 모델 (무료·빠름)
    messages: [{ role: 'user', content: '한 문장으로 자기소개 해줘' }]
  })
})
const data = await groqRes.json()
console.log(data.choices?.[0]?.message?.content || data)
```

**마지막 줄의 새 문법 두 개:**
- **`?.` (Optional Chaining)** — "혹시 없으면 에러 내지 말고 undefined만 줘". `data.choices[0]`처럼 그냥 접근하면 choices가 없을 때 프로그램이 죽지만, `?.`는 안 죽고 넘어간다.
- **`||` (OR)** — 왼쪽 값이 있으면 그거, 없으면 오른쪽. 그래서 정상이면 AI 답이, 이상하면 data 전체가 찍혀 디버깅 가능.

⚠️ 만난 에러: `require`(CommonJS)와 파일 맨 바깥의 `await`(ES Module)를 섞으면 `ERR_AMBIGUOUS_MODULE_SYNTAX` — **`async` 함수로 감싸서** 해결.

### 2-3. 프론트 챗봇 — 키를 숨기는 프록시 구조

화면에 키를 넣으면 F12로 누구나 훔쳐본다 → **서버(심부름꾼)가 대신 호출**하고, 키는 서버 금고(.env)에만.

```
화면(client) ──프롬프트──▶ 내 서버 /api/chat ──키+질문──▶ Groq
          ◀──답변────              ◀──AI 응답──
```

서버는 오전에 배운 Express 그대로 (`cors()` + `express.json()` + `app.post`), 화면은 6일차에 배운 DOM 조작 그대로 (`getElementById`, `addEventListener`, fetch). **오전 백엔드 + 기존 프론트 지식이 합쳐져 AI 챗봇이 됐다.**

**헷갈림 주의 — 이름이 같은 `.json()` 두 개, 방향은 정반대:**

| 위치 | 코드 | 하는 일 |
|------|------|---------|
| 서버 | `res.json(객체)` | 객체 → 문자열 (내부에서 stringify) |
| 클라이언트 | `res.json()` | 문자열 → 객체 (파싱) |

인터넷으로는 **글자만** 오갈 수 있어서, 보낼 땐 문자열로 바꾸고 받으면 객체로 되돌린다.

**`.then()` vs `.catch()`** — then은 "성공하면 뭐 할래", catch는 "실패하면 뭐 할래". 단, fetch는 404·500도 "응답은 온 것"이라 then으로 간다. catch가 잡는 건 **서버가 아예 꺼졌거나 네트워크가 끊긴 진짜 실패** — 그래서 "❌ 서버 안 켜짐?" 안내가 catch에 있다.

## 3. 어려웠던 것 — 심화 학습

### HSL 컬러 — 배치도에 20가지 색 입히기

CRUD 배치도 실습에서 학생 20명의 좌석마다 다른 색을 입히려고 HSL을 활용했다.

**HSL = hsl(Hue, Saturation, Lightness)** 세 값으로 색을 지정하는 방식:

| 값 | 범위 | 뜻 |
|-----|------|-----|
| **Hue (색상)** | 0~360도 | 색상환에서의 각도 — 0=빨강, 120=초록, 240=파랑, 360=다시 빨강(한 바퀴) |
| **Saturation (채도)** | 0~100% | 색의 선명함 — 0%=회색, 100%=원색 |
| **Lightness (명도)** | 0~100% | 밝기 — 0%=검정, 50%=원색 그대로, 100%=흰색 |

**실습 코드의 핵심:**

```js
const hue = i * (360 / 20)               // 색상환을 20등분 (18도 간격)
el.style.background = `hsl(${hue}, 70%, 50%)`
```

`hsl(${hue}, 70%, 50%)` = "Hue만 0~360도 사이를 20등분해서 돌아가며 쓰고, 채도 70%·명도 50%는 고정". 그래서 색상환을 따라 **빨강→노랑→초록→파랑→보라로 자연스럽게 이어지는 20가지 색**이 나온다.

**보색 계산:**

```js
const compHue = (hue + 180) % 360   // 색상환 정반대(180도) = 보색
```

hue가 0(빨강)이면 compHue는 180(청록). `% 360`은 한 바퀴를 넘어가면 다시 0부터 세는 처리 — 배경색과 글자색을 보색으로 두면 가독성이 확보된다.

**왜 RGB가 아니라 HSL인가:** RGB(`rgb(255,0,0)`)는 빨强·초록·파랑의 "혼합량"이라 색을 균등하게 나누기 어렵다. HSL은 **"어떤 색인지(Hue)"와 "얼마나 밝은지(Lightness)"를 분리**해서 다루기 때문에, 지금처럼 색상만 균등하게 나누는 작업에 훨씬 편하다.

> 색 표현의 성장기: 3일차 named color(이름) → hex 코드(#F0E68C) → 9일차 textToHex(글자→색 함수) → 오늘 **HSL(각도로 색을 계산)**. 이제 색을 "고르는" 게 아니라 "계산"한다.

## 4. 과제 및 다음에 할 것

### ✅ 오늘의 과제
- [x] 깃허브 업로드
- [x] 블로그 작성 (지금 이 글!)

### 📌 다음에 할 것
- 배치도 실습 마무리 점검 — 오염 데이터 정정 결과 재확인
- Groq 챗봇 응용 아이디어 스케치 (추천 챗봇 · AI 학습도우미 · 여행플래너)
- React 프론트 + Express 백엔드 + AI API를 하나로 합치는 풀스택 구성 예습
- `.env`가 든 폴더 깃 상태 확인 — .gitignore 제대로 먹었는지 `git status`로 검증

---

**오늘의 한 줄 정리:** 지금까지는 요청을 보내는 쪽이었고, 오늘부터 받는 쪽도 만들 수 있다. 그리고 API 키는 무조건 서버 금고(.env)에 — 화면에 둔 비밀은 비밀이 아니다. 🔐
