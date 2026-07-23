---
layout: post
title: "[Day 18] AI에 UI 입히기 — Streamlit·Gradio, 그리고 나만의 Claude Code 스킬 만들기"
date: 2026-07-23
categories: [TIL, Python, AI]
tags: [Streamlit, Gradio, Plotly, ClaudeCode, Skills, Hooks, CLAUDEmd, 자동화]
---

웹개발 수업 18일차. 터미널에서만 돌던 AI를 **웹 화면으로 꺼내는 날**이었다 — Streamlit으로 실시간 CCTV 대시보드를, Gradio로 번역기·YOLO 탐지기를 만들고 탭 하나로 합쳤다. 오후에는 Claude Code 로드맵을 스킬까지 훑었고, 심화로 **나만의 스킬**을 직접 만들었다.

## 1. 수업 내용 요약 ① — Streamlit (파이썬으로 만드는 대시보드)

HTML·CSS·JS 없이 파이썬만으로 웹 앱을 만드는 도구. `streamlit run 파일명.py` 한 줄이면 브라우저가 열린다.

```bash
pip install streamlit
streamlit run v20_01_streamlit_basic.py    # python이 아니라 streamlit run!
```

### 1-1. 핵심 아이디어 — 빈 액자에 사진 갈아끼우기

실시간 영상의 원리가 명쾌했다. **`st.empty()`로 빈 자리(placeholder)를 예약**해두고, 반복문 안에서 그 자리에 프레임을 계속 덮어쓰면 슬라이드 쇼처럼 움직인다.

```python
st.set_page_config(layout="wide")        # 반드시 코드 최상단
st.title("YOLO 실시간 CCTV 탐지")
frame_placeholder = st.empty()           # ← 빈 액자 예약

cap = cv2.VideoCapture("http://210.99.70.120:1935/live/cctv013.stream/playlist.m3u8")
model = YOLO("yolo11n.pt")

while cap.isOpened():
    success, frame = cap.read()
    if not success:
        st.warning("프레임 읽기 실패(Streamlit)")
        break
    annotated = model(frame)[0].plot()
    frame_placeholder.image(annotated, channels="BGR")   # 액자에 덮어쓰기
cap.release(); cv2.destroyAllWindows()
```

- **`channels="BGR"`** — OpenCV는 색 순서가 BGR이라 명시해야 색이 제대로 나온다. 16일차 `putText` 때 배운 그 BGR 함정이 여기서 또!
- `st.set_page_config()`는 **맨 위**가 아니면 동작하지 않는다

### 1-2. 컬럼 레이아웃 + Plotly 실시간 그래프

화면을 좌우로 쪼개 **왼쪽엔 영상, 오른쪽엔 탐지 개수 그래프**를 동시에 갱신했다.

```python
col1, col2 = st.columns(2)
with col1: frame_placeholder = st.empty()
with col2: chart_placeholder = st.empty()

@st.cache_resource            # ★ 무거운 모델은 딱 한 번만 로드
def load_model():
    return YOLO("yolo11n.pt")
model = load_model()

# 탐지 결과 → 이름 목록 → 개수 집계 → 막대 그래프
labels = [model.names[int(c)] for c in results[0].boxes.cls]
df_count = pd.DataFrame({"Object": labels}).value_counts().reset_index(name="Count")
fig = px.bar(df_count, x="Object", y="Count", color="Object", text="Count")

frame_placeholder.image(annotated, channels="BGR")
chart_placeholder.plotly_chart(fig, use_container_width=True, key=f"chart_{time.time()}")
```

**세 가지 포인트:**
- **`@st.cache_resource`** — Streamlit은 화면을 조작할 때마다 스크립트를 처음부터 다시 실행한다. 캐시를 안 걸면 매번 모델을 새로 읽어서 느려진다.
- **`key=f"chart_{time.time()}"`** — 매 프레임 고유 key를 줘서 위젯 갱신 충돌 방지. React에서 배운 `key` 개념과 같은 이유(각 요소를 구분하는 이름표)라는 게 재밌었다.
- `model.names[int(c)]` — 15일차에 뒤집어봤던 그 `model.names`가 여기선 정방향(인덱스→이름)으로 쓰인다.

## 2. 수업 내용 요약 ② — Gradio (함수 하나면 웹앱)

Streamlit이 "대시보드"라면 Gradio는 **"함수에 화면 씌우기"**에 특화. 심지어 `share=True` 한 줄로 72시간짜리 공개 URL까지 나온다.

```bash
pip install gradio deep-translator ultralytics
```

> ⚠️ **import하자마자 `numpy.core.multiarray failed to import`가 뜨면** gradio 잘못이 아니다. numpy 2부터 내부 모듈명이 `numpy.core` → `numpy._core`로 바뀌었는데, 옛 numpy로 빌드된 pandas·matplotlib이 옛 이름을 부르기 때문. gradio는 내부에서 pandas를 쓰다 터진 **피해자**다. 해결은 `pip install -U pandas matplotlib` (numpy를 1.x로 내리는 건 오히려 torch·ultralytics가 깨져서 비추천). 범인 찾기는 `python -c "import pandas; print('pandas OK')"`처럼 하나씩 import해보면 된다.

### 2-1. gr.Interface — 3줄이면 웹앱

```python
import gradio as gr

def say_hello(name):
    return "Hello, " + name

gr.Interface(fn=say_hello, inputs="text", outputs="text").launch(share=True)
```

`fn`(실행할 함수) · `inputs` · `outputs` 세 개만 정하면 화면은 Gradio가 알아서 그려준다. 번역기도 함수만 바꾸면 끝:

```python
def trans_en_to_ko(text):
    return GoogleTranslator(source="en", target="ko").translate(text)
```

YOLO 탐지기도 마찬가지 — 다만 **`gr.Image(type="numpy")`**로 받아야 한다 (type을 안 적으면 PIL 객체로 들어와 오류).

### 2-2. gr.Blocks — 앱 3개를 탭 하나로 ★

`gr.Interface`는 **앱 하나에 기능 하나**다. 세 앱을 같이 쓰려면 파이썬을 세 번 실행해야 한다. `gr.Blocks`는 화면을 직접 배치하는 방식이라 탭으로 묶을 수 있다.

```python
with gr.Blocks() as gr_web:
    gr.Markdown("# 내가 만든 AI 앱 모음")

    with gr.Tab("인사"):
        name_box = gr.Textbox(label="이름")
        hello_out = gr.Textbox(label="인사말")
        gr.Button("인사하기").click(say_hello, name_box, hello_out)

    with gr.Tab("번역"):
        en_box = gr.Textbox(label="영어")
        ko_box = gr.Textbox(label="한국어")
        gr.Button("번역하기").click(trans_en_to_ko, en_box, ko_box)

    with gr.Tab("YOLO 탐지"):
        img_in = gr.Image(type="numpy", label="사진 올리기")
        img_out = gr.Image(label="탐지 결과")
        gr.Button("탐지하기").click(det_image, img_in, img_out)

gr_web.launch(share=True)          # with 블록 "밖"에서 실행
```

| 비교 | gr.Interface | gr.Blocks |
|------|-------------|-----------|
| 화면 | Gradio가 알아서 | 내가 쓴 순서대로 배치 |
| 기능 | 1개 | 여러 개 (Tab·Row·Column) |
| 실행 시점 | 입력하면 자동 | `.click()` 등으로 내가 지정 |
| 칸 지정 | `"text"` 형식만 | `name_box`처럼 변수로 지목 |

**`.click(함수, 입력칸, 출력칸)`이 핵심 한 줄** — "버튼 누르면 → 이 함수 실행 → 이 칸에서 읽어 → 저 칸에 결과". 6일차 JS의 `addEventListener("click", ...)`와 완전히 같은 구조다. 실행 방식도 `.click()`(버튼) · `.change()`(입력 변할 때) · `.submit()`(엔터)로 나뉜다.

**파이썬 들여쓰기가 그대로 화면 구조가 된다**는 게 가장 인상적이었다. `with` 안에 `with`를 넣어 Tab 안에 Row를 겹치는 식. 14일차에 "들여쓰기는 법률"이라고 배웠는데, 여기선 아예 레이아웃이 된다.

**자주 하는 실수 4가지**도 미리 기록: ① 탭마다 칸 이름 중복(나중 것이 덮어씀) ② `.click()`을 with 밖에 씀 ③ 버튼만 만들고 `.click()` 안 붙임(눌러도 무반응) ④ `launch()`를 with 안에 씀.

## 3. 수업 내용 요약 ③ — Claude Code 로드맵 (01~19)

수업 내내 조수로 써온 Claude Code를 **제대로** 배우는 시간. 19개 스텝을 주제별로 묶으면:

### 3-1. 기본 — 설치·모델·명령어

- 설치: `npm install -g @anthropic-ai/claude-code` (Node.js 18+)
- **모델 이름은 시(詩) 형식에서** — Haiku(17음, 가장 짧은 시 → 가장 빠름) · Sonnet(14행 정형시 → 균형) · Opus(라틴어 '대작' → 가장 강력). 이름에 성격이 담겨 있었다니!
- 명령어: `/init`(CLAUDE.md 자동 생성) · `/clear` · `/compact`(대화 압축으로 토큰 절약) · `/usage`(사용량 확인) · `/doctor`(구조 진단) · `@파일`(특정 파일 지정) · `/model` · `!prefix`(즉시 실행)
- **모드 전환 = Shift+Tab** — `default`(매번 확인) / `auto`(자동 수행) / **`plan`(계획 먼저 작성 → 승인 후 구현)**. 4일차 폴더 자동화, 13일차 웹사이트 제작 때 쓴 그 플랜 모드의 정체가 여기 있었다.
- 자율권이 높을수록 속도↑ 위험↑ — *"큰 힘에는 큰 책임이 따른다"*

### 3-2. CLAUDE.md — AI에게 주는 업무 지시서

12일차에 "실행 시 먼저 읽는 문서"로 잠깐 배웠던 걸 오늘 제대로 정리했다.

- **200줄 이하 유지** — 모든 세션 시작 시 컨텍스트에 로드되므로, 길수록 토큰 소비↑ 준수율↓. 초과분은 `@import`나 `.claude/rules/`로 분리
- **포함할 것**: 빌드·테스트 명령어, 코드 스타일, 네이밍 규칙, 환경변수 주의사항
- **제외할 것**: 코드 보면 아는 것, 자주 바뀌는 정보, "깔끔한 코드 작성" 같은 당연한 말
- **"IMPORTANT" · "YOU MUST"** 같은 강조어가 준수율을 올린다
- 개인 설정은 `CLAUDE.local.md`로 분리해 .gitignore에 (13일차 `.env` 감각과 동일)

### 3-3. 프롬프트 잘 쓰는 법

원칙이 전부 **"막연함 → 구체"** 한 방향이었다:

| 원칙 | ❌ | ✅ |
|------|-----|-----|
| 위치 지정 | 버그 고쳐줘 | auth.js에서 비밀번호 찾기 버튼 클릭 시 오류 수정해줘 |
| 성공 기준 | 이메일 확인 기능 만들어줘 | ...abc@test.com은 통과, 글자만 쓰면 오류. 완성 후 직접 검사해줘 |
| 기존 방식 참조 | 새 위젯 추가해줘 | 기존 ProductCard 방식 그대로 달력 위젯도 만들어줘 |
| 금지사항 명시 | 코드 정리해줘 | ...새 도구 추가 말고, 동작하는 기능은 건드리지 말 것 |
| 단계별 분해 | 로그인 다 만들어줘 | 현황 파악 → 방법 설명 → 코드 작성 순서로, 단계마다 확인 |

구조화 템플릿도 배웠다: **`[ROLE] [GOAL] [CONSTRAINTS] [CONTEXT] [QUESTION]`**. 그리고 **질문 5선지 기법**(객관식 4 + 주관식 1, 추천에 ★) — 작업 전에 Claude가 먼저 물어보게 해서 방향을 맞추는 방식인데, CLAUDE.md에 등록하면 매번 요청 안 해도 자동 적용된다.

### 3-4. 모델 선택 & Effort 레벨

**Effort = 사고 깊이**를 조절하는 값. `low`(오타 수정) → `medium`(일반 코드) → `high`(복잡한 버그·설계) → `xhigh` → `max`. 깊을수록 결과가 좋지만 토큰↑ 비용↑. 모델과 **독립적으로** 조절 가능하다는 게 포인트.

### 3-5. 스킬 (Skills) — 나만의 슬래시 명령어

**요리 레시피 비유**가 딱이었다:

- 🍳 레시피 저장 = 스킬 파일 작성 (`.claude/skills/이름.md`)
- 🗣️ "레시피대로 해줘" = `/이름` 입력
- 👨‍🍳 요리사가 레시피대로 = Claude가 그 내용대로 작업

자주 쓰는 작업을 파일에 저장해두면 짧은 명령어 하나로 재현된다. → 여기가 오늘 심화의 출발점!

## 4. 어려웠던 것 — 심화 학습

### 나만의 Claude Code 스킬 만들기 🛠️

수업에서 스킬 개념을 배우자마자, **직접 만들어봤다.**

**① 먼저 남의 스킬 써보기 — Grill-Me, Caveman**

바로 만들기 전에 공개된 스킬 두 개(**Grill-Me** · **Caveman**)를 설치해서 써봤다. 스킬이 어떻게 호출되고 Claude의 행동이 어떻게 달라지는지 감을 잡는 게 목적. 남이 만든 걸 뜯어보는 게 가장 빠른 학습법이라는 건 3일차에 F12로 남의 사이트를 뜯어보던 때와 같다.

**② 내 불편함 찾기**

Claude Code로 작업하다 보면 `.md` 파일을 자주 만든다 — 계획 문서, 기획서, 정리 노트. 그런데 **만들고 나면 터미널에 경로만 뜬다.** 확인하려면 매번 탐색기를 열어 그 폴더를 찾아가야 했다. 하루에도 몇 번씩 반복되는 사소한 마찰.

→ **"만든 .md를 바로 열어주는 스킬"**을 만들기로 했다.

**③ SKILL.md 해부**

```markdown
---
name: markdown-reader
description: MarkdownReader - opens a markdown (.md) file with the OS's default
  associated program (no conversion or styling). Use this whenever the user
  explicitly asks to open, view, or show a specific .md file (e.g. "이 파일 열어줘",
  "README 보여줘", "open this markdown file"). Do not use it just because a .md
  path is mentioned in passing (e.g. as part of a request to edit or fix
  something in that file) - only when the user wants to actually look at the file.
---

Resolve the target file to an absolute path, then run:

python -c "import os,sys; os.startfile(sys.argv[1])" "<absolute-path-to-file.md>"
```

만들면서 알게 된 것 3가지:

**⑴ description이 사실상 "발동 조건"이다.** 스킬 파일에서 제일 중요한 건 본문이 아니라 **description**이었다. Claude가 이걸 읽고 "지금 이 스킬을 써야 하나?"를 판단하기 때문. 그래서 두 가지를 다 적었다:

- **써야 할 때** — "이 파일 열어줘", "README 보여줘", "open this markdown file" (한국어·영어 예시 모두)
- **쓰면 안 될 때** — *"그 파일 안의 뭘 고쳐줘" 같은 요청에서 .md 경로가 스쳐 지나가는 경우엔 쓰지 말 것*

이 **반례(negative example)**가 핵심이다. 없으면 md 경로만 나와도 창을 벌컥벌컥 열어버린다. 13일차에 배운 system prompt 작성 요령("금지보다 허용 범위를 구체적으로")과 정확히 같은 이야기 — **AI에게 주는 지시는 경계선을 그려주는 일**이다.

**⑵ 실제 동작은 파이썬 한 줄.**

```python
python -c "import os,sys; os.startfile(sys.argv[1])" "<절대경로.md>"
```

`os.startfile()` = 윈도우에서 **더블클릭한 것과 같은 효과** — OS에 등록된 기본 연결 프로그램으로 파일을 연다. 변환도, 스타일링도 없이 그냥 연다. `sys.argv[1]`은 명령줄로 넘긴 첫 번째 값(파일 경로). 14일차에 배운 `import os` · `sys`가 이렇게 쓰인다.

거창한 코드가 아니라 **한 줄짜리 명령**이라는 게 포인트. 스킬의 본질은 "복잡한 코드"가 아니라 **"반복되는 판단과 명령을 문서로 굳혀두는 것"**이었다.

**⑶ 스킬 + 훅 조합 — 두 갈래로 처리**

파일 맨 아래에 이런 메모를 남겼다:

> 새로 만든 `.md` 파일과, 경로만 덩그러니 드래그해서 넣은 경우는 `settings.json`의 **훅(hook)**이 이미 자동 처리한다. 이 스킬은 대화 중에 "이 파일 열어줘"라고 **명시적으로 요청**할 때만 담당한다.

수업의 STEP 21(스킬 vs 훅)이 딱 이 구도다:

- **📱 스킬 = 배달 앱** — 내가 부를 때만 실행
- **🚪 훅 = 자동문 센서** — 조건이 되면 저절로 실행

즉 **"새 md가 만들어지면 자동으로 열기"는 훅**에게, **"저 파일 좀 열어줘"라는 요청은 스킬**에게 나눠 맡긴 것. 두 방식이 겹치지 않게 역할을 나누는 설계까지 해본 셈이다.

**배운 점:** 4일차에 PowerShell 스크립트로 실습 폴더 생성을 자동화했던 것과 같은 동기다 — **반복되는 마찰을 발견하면 자동화한다.** 다만 그때는 "내가 실행하는 스크립트"였고, 이번엔 **"AI가 판단해서 실행하는 스킬"**이라는 게 다르다. 자동화의 대상이 손에서 판단으로 한 칸 올라갔다.

## 5. 과제 및 다음에 할 것

### ✅ 오늘의 과제
- [x] 깃허브 업로드
- [x] 블로그 작성 (지금 이 글!)

### 📌 다음에 할 것
- markdown-reader 스킬 실전 검증 — 반례 상황("이 md 고쳐줘")에서 안 열리는지 확인
- 스킬 하나 더 만들기 후보: 블로그 md 템플릿 자동 생성? 커밋 메시지 규칙?
- Claude Code 로드맵 남은 부분(20 훅 · 21 스킬vs훅 · 22 Codex CLI · 23 Cowork) 마저 보기
- Gradio `share=True`로 만든 앱을 HuggingFace Spaces에 올려보기 (무료 영구 호스팅)
- Streamlit + Gradio 비교 정리 — 어떤 상황에 뭘 고를까?

---

**오늘의 한 줄 정리:** 파이썬 함수에 화면을 씌우면 웹앱이 되고(Gradio), 반복되는 작업을 문서로 굳히면 스킬이 된다. 오늘은 **내 도구를 내가 만드는 법**을 배웠다. 🛠️
