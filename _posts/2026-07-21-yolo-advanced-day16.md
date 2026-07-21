---
layout: post
title: "[Day 16] YOLO 심화 — CCTV 실전 기능 19종, 그리고 SSL 인증서와 싸운 날"
date: 2026-07-21
categories: [TIL, Python, AI]
tags: [YOLO, 추적, 히트맵, SAHI, OpenVINO, Streamlit, SSL, certifi]
---

웹개발 수업 16일차. 어제 YOLO 기초를 뗐고, 오늘은 **심화 19개 스텝 전부** — 추적, 히트맵, 보안 알람, 블러, 거리·속도 측정, 출입 카운팅, SAHI, SAM, OpenVINO, Streamlit까지. CCTV 영상에 붙일 수 있는 실전 기능을 총망라한 날이다. 그리고 수업 내내 발목을 잡은 **SSL 인증서 오류**와 그 해결 과정을 심화로 정리했다.

## 1. 수업 내용 요약 — YOLO 심화 19종

### 1-1. 추적 (Tracking) — 탐지에 이름표(ID) 붙이기

탐지는 매 프레임 독립적으로 새로 찾지만, **추적은 물체마다 고유 ID를 붙여 계속 따라간다.** 사람 1번이 화면 밖으로 나갔다 들어와도 같은 1번.

```python
results = model.track(frame, persist=True, conf=0.6)
# persist=True → 이전 프레임 정보 유지 (ID 지속의 핵심)
```

`model()` 대신 `model.track()` — 메서드 하나 차이로 탐지가 추적이 된다.

### 1-2. solutions 패밀리 — 조립식 실전 기능 6종

오늘의 최대 발견: ultralytics의 **`solutions.*`** — 탐지+추적+후처리를 통째로 묶은 완성형 도구들. **전부 같은 사용 패턴**이라 하나를 익히면 나머지가 공짜다:

```python
도구 = solutions.기능이름(model="yolo11n.pt", show=True, 옵션...)
while cap.isOpened():
    success, frame = cap.read()
    도구(frame)          # 인스턴스를 함수처럼 직접 호출 — 이게 전부!
```

| 기능 | 클래스 | 핵심 옵션 · 배운 점 |
|------|--------|---------------------|
| 히트맵 | `Heatmap` | `colormap=cv2.COLORMAP_MAGMA` — 자주 나타난 곳을 열지도로 (매장 동선 분석) |
| 보안 알람 | `SecurityAlarm` | 특정 클래스 감지 시 **Gmail 자동 발송**. `records=2`(N회 감지 시 1회 메일 — 오탐 방지), 앱 비밀번호 16자리 필수 |
| 블러 | `ObjectBlurrer` | `blur_ratio=0.3` — 탐지 영역만 자동 모자이크 (얼굴·번호판 익명화) |
| 거리 계산 | `DistanceCalculation` | 좌클릭 2회로 객체 선택 → 중심점 픽셀 거리. `process()` 결과의 `pixels_distance`를 꺼내 **SAFE/Warning/DANGER 3단계 분류**도 직접 구현 |
| 영역 카운팅 | `RegionCounter` | 다각형 4점 안의 객체만 집계. **좌표는 마우스 클릭 도구로 추출** (`setMouseCallback`) |
| 출입 카운팅 | `ObjectCounter` | 가상의 선(2점)을 넘으면 IN/OUT 자동 집계 — 편의점 문 앞 보이지 않는 줄. 한 ID는 한 번만 카운트 |

**공통 함정 — 좌표계 일치:** 영역·선 좌표는 `cv2.resize(frame, (640, 480))` 후 기준이다. resize를 빼먹으면 영역이 엉뚱한 곳에 생긴다. 좌표를 찍을 때도, 쓸 때도 같은 해상도!

**보안 알람의 보안:** 앱 비밀번호를 코드에 직접 쓰지 말고 환경변수로 분리 — 12일차 `.env` 교훈이 파이썬에서도 그대로 (`os.environ.get("GMAIL_PW")`).

### 1-3. 마우스 콜백 — 클릭으로 좌표 얻기

영역 좌표를 하드코딩하는 대신 화면을 클릭해 추출하는 도구도 만들었다:

```python
def mouse_callback(event, x, y, flags, param):   # 인자 5개 순서 고정!
    if event == cv2.EVENT_LBUTTONDOWN:
        points.append((x, y))

cv2.namedWindow("GET_X_Y", cv2.WINDOW_NORMAL)    # 창 먼저 만들고
cv2.setMouseCallback("GET_X_Y", mouse_callback)  # 콜백 연결
```

- 콜백 인자 5개는 안 써도 자리를 비우면 안 됨 (OpenCV 호출 규칙)
- 6일차 JS의 `addEventListener("click", ...)`와 완전히 같은 개념 — **이벤트에 함수를 등록**하는 패턴은 언어를 넘나든다
- 콜백 안에서 `points = []` 재할당 금지 — 전역이 가려진다. `append`로만!

### 1-4. SAHI 3부작 — 작은 물체 잡는 돋보기

멀리 있는 작은 차·사람은 일반 YOLO가 놓친다. **SAHI**는 사진을 타일로 잘라 조각마다 추론 후 합치는 기법 — 큰 그림책을 돋보기로 한 페이지씩 보는 방식.

```python
results = get_sliced_prediction(
    "demo_data/small-vehicles1.jpeg", detection_model,
    slice_height=200, slice_width=200,        # 타일 크기 (작을수록 작은 물체 잘 봄)
    overlap_height_ratio=0.1, overlap_width_ratio=0.1  # 경계 물체 놓침 방지 10% 겹침
)
```

같은 사진을 STEP 11(통째 추론)과 12(슬라이싱)로 비교하니 **탐지 수가 확 늘어난다.** 트레이드오프도 여전 — 타일이 작을수록 정확하지만 느려진다.

### 1-5. 속도 추정 — 픽셀을 km/h로

```python
yolo_speed = solutions.SpeedEstimator(
    model="yolo11n.pt", show=True,
    meter_per_pixel=0.5,    # 픽셀 1개 = 실제 0.5m — 직접 보정 필수!
    classes=[2], max_speed=120
)
```

핵심은 **`meter_per_pixel` 보정** — 화면 1픽셀이 실제 몇 m인지. 도로 점선 길이(일반도로 3m)처럼 **길이를 아는 기준물**을 화면에서 재서 `3m ÷ 6px = 0.5` 식으로 구한다. 단일 환산값은 원근 왜곡 때문에 근사치라는 한계까지 — 정밀하게 하려면 Homography(조감도 변환)나 카메라 캘리브레이션이 필요하다고.

### 1-6. SAM & FastSAM — 뭐든지 잘라내는 AI

- **SAM** (Meta) — 별도 학습 없이 이미지의 **모든 물체를 픽셀 단위로** 세그먼트. YOLO보다 정밀하지만 느림.
- **FastSAM** — SAM의 다이어트 버전 (YOLO 골격, ~50배 빠름). `texts="dog"`처럼 **텍스트 한 단어로 원하는 물체만** 콕 집어 잘라낸다.

```python
results = model(source, texts="dog")   # 말로 시킨 것만 세그먼트!
```

### 1-7. OpenVINO — CPU 부스터 & FPS 측정

```python
model.export(format="openvino")   # .pt → _openvino_model/ 폴더로 변환
```

Intel의 CPU 전용 최적화. GPU 없이도 추론이 2~3배 빨라진다. 같은 프레임을 30번씩 돌려 **직접 FPS를 측정·비교**하는 코드도 짰다 — `time.time()`으로 전후 시각을 찍어 평균 내고 `1/평균 = FPS`. "성능 향상 몇 배"를 감이 아니라 숫자로 확인하는 습관.

### 1-8. SearchApp & Streamlit — AI에 UI 입히기

- **SearchApp** — 내 사진 폴더 전용 구글. "dog"라고 치면 비슷한 이미지를 찾아주는 웹 검색 사이트가 뜬다 (내부는 CLIP 임베딩 + FAISS)
- **Streamlit Inference** — 코드 몇 줄로 모델 선택·웹캠 입력·결과 표시가 되는 **YOLO 대시보드** 웹 UI

```bash
streamlit run ./v15_19_yolo_streamlit.py   # python이 아니라 streamlit run!
```

터미널에서만 돌던 AI가 드디어 **웹 화면**을 얻었다 — 프론트엔드 배운 사람 입장에선 여기가 제일 반가운 대목.

## 2. 어려웠던 것 — 심화 학습

### SSL 인증서 오류와의 전투 🔐

오늘 실습 중 코드가 제대로 작동하지 않는 경우가 있었다 — YOLO 가중치나 데모 이미지를 **인터넷에서 내려받는 순간** 에러가 나는 것.

#### 문제 상황

ultralytics는 `YOLO("yolo11n.pt")`처럼 모델을 지정하면 파일이 없을 때 자동으로 다운로드한다. SAHI 데모 이미지 다운로드, Streamlit 실행도 마찬가지로 HTTPS 통신을 한다. 그런데 이 과정에서 **SSL 인증서 검증 실패** 계열 오류(`CERTIFICATE_VERIFY_FAILED`, `ASN1 NOT_ENOUGH_DATA` 등)가 나면서 다운로드가 죽었다.

원인을 정리하면: HTTPS 통신은 "상대 서버가 진짜인지"를 **인증서**로 검증하는데, 파이썬이 그 검증에 쓸 **신뢰 목록(CA 저장소)을 윈도우에서 제대로 읽지 못하는 상태**였던 것. `ASN1 NOT_ENOUGH_DATA`는 인증서 데이터를 읽다가 내용이 손상·불완전해 해석(파싱)에 실패했다는 뜻이다. 즉, 서버가 아니라 **내 컴퓨터의 신뢰 목록이 고장** — 실습망 환경에서 종종 겪는 문제라고 한다.

#### 1차 해결 — 강사님이 복사해준 우회 코드 (임시방편)

```python
import ssl, os
ssl._create_default_https_context = ssl._create_unverified_context
os.environ["CURL_CA_BUNDLE"] = ""
os.environ["REQUESTS_CA_BUNDLE"] = ""
```

각 줄의 의미:

| 코드 | 하는 일 |
|------|---------|
| `ssl._create_default_https_context = ssl._create_unverified_context` | 파이썬이 HTTPS 연결을 만들 때 쓰는 기본 설정을 **"검증 안 하는 버전"으로 통째로 교체.** urllib 계열(ultralytics 내부 다운로드 포함)이 인증서 확인 없이 접속한다 |
| `os.environ["CURL_CA_BUNDLE"] = ""` | requests 등 curl 계열 라이브러리가 참조하는 **신뢰 목록 경로를 빈 값으로** → 검증 생략 |
| `os.environ["REQUESTS_CA_BUNDLE"] = ""` | requests 라이브러리 전용 신뢰 목록 경로도 빈 값으로 → 검증 생략 |

라이브러리마다 인증서 설정을 읽는 통로가 달라서(파이썬 기본 ssl / curl 계열 / requests) **세 통로를 전부 막아야** 확실히 통과된다. 이 네 줄을 스크립트 **맨 위**(다운로드가 일어나기 전)에 붙여넣으면 오류 없이 동작했다.

⚠️ **단, 이건 자물쇠를 고친 게 아니라 떼어버린 것.** 검증을 끄면 "상대가 진짜 서버인지" 확인하지 않으므로, 중간에서 통신을 가로채는 공격에 무방비가 된다. 신뢰할 수 있는 실습 환경에서 임시로만 쓰고, 실무 코드에 넣으면 안 된다.

#### 2차 해결 — 강사님의 fix_ssl.bat (근본 해결)

나중에 강사님이 **.bat 파일**을 업로드해주셔서 실행했는데, 이건 접근이 완전히 다르다 — 검증을 끄는 게 아니라 **고장난 신뢰 목록을 새것으로 교체**한다. bat 파일이 하는 일 3단계:

**[0단계] py39 파이썬 자동 탐색** — Anaconda 설치 경로가 사람마다 달라서(`%USERPROFILE%\anaconda3`, `C:\ProgramData\Anaconda3` 등), 후보 경로 6곳을 돌며 `py39` 가상환경의 python.exe를 찾는다. 못 찾으면 수동 실행법을 안내하고 종료.

**[1/3] `pip install -U certifi`** — **certifi** = 파이썬용으로 배포되는 **믿을 수 있는 인증서 목록**(Mozilla가 관리하는 CA 번들). 고장난 윈도우 저장소 대신 쓸 "새 신뢰 목록"을 설치·최신화하는 것.

**[2/3] sitecustomize.py 생성** — 핵심 한 수. site-packages에 이런 내용의 파일을 만든다:

```python
import ssl, certifi
def _use_certifi(self, purpose=ssl.Purpose.SERVER_AUTH):
    self.load_verify_locations(certifi.where())
ssl.SSLContext.load_default_certs = _use_certifi
```

- `sitecustomize.py`는 **파이썬이 시작될 때마다 자동으로 실행되는 특수 파일** — 그래서 어떤 스크립트를 돌리든 매번 이 패치가 적용된다 (스크립트마다 우회 코드를 붙여넣을 필요가 사라짐!)
- 내용은 "기본 인증서를 불러오는 함수(`load_default_certs`)"를 **"윈도우 저장소 대신 certifi의 목록(`certifi.where()`)을 읽는 함수"로 갈아끼우는** 것. 고장난 통로를 아예 안 거치게 만든다.

**[3/3] 검증 테스트** — `ssl.create_default_context()`가 오류 없이 실행되면 `SSL OK` 출력. 이게 뜨면 성공.

#### 두 해결책 비교 — 오늘의 진짜 교훈

| | 1차: 우회 코드 | 2차: fix_ssl.bat |
|---|---------------|------------------|
| 방식 | 인증서 **검증을 끔** | 신뢰 목록을 certifi로 **교체** |
| 보안 | ❌ 무방비 (자물쇠 제거) | ✅ 검증 유지 (자물쇠 교체) |
| 적용 범위 | 붙여넣은 그 스크립트만 | 그 환경의 **모든 파이썬 실행** |
| 성격 | 응급처치 | 근본 치료 |

같은 오류에 대한 두 접근을 하루에 다 경험한 게 값졌다 — **"일단 돌아가게" 다음엔 반드시 "왜, 그리고 제대로"가 와야 한다.** 그리고 에러 메시지(`ASN1 NOT_ENOUGH_DATA`)를 무서워하지 않고 "인증서 데이터가 제대로 안 읽힘"으로 번역해내는 것부터가 해결의 시작이었다.

## 3. 과제 및 다음에 할 것

### ✅ 오늘의 과제
- [x] 깃허브 업로드
- [x] 블로그 작성 (지금 이 글!)

### 📌 다음에 할 것
- solutions 6종 중 하나 골라 응용 아이디어 만들기 — 출입 카운팅 + putText로 "현재 인원" 표시판?
- 표정 분류 챌린지 계속 — 모델 학습 시 SSL 이슈 재발하면 bat 방식으로 해결
- `sitecustomize.py` 개념 더 알아보기 — 파이썬 시작 시 자동 실행되는 파일이 또 있을까?
- Streamlit 문서 훑어보기 — YOLO 대시보드를 내 입맛대로 꾸밀 수 있는지

---

**오늘의 한 줄 정리:** 탐지 위에 추적을, 추적 위에 카운팅·알람·속도를 — AI 기능은 레고처럼 쌓인다. 그리고 에러 해결에도 급이 있다: 검증을 끄는 응급처치보다, 신뢰 목록을 고치는 근본 치료. 🔐
