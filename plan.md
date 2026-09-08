가능해. 다만 **애자일을 "매일 새로운 기능을 추가하는 방식"이 아니라, 매일 실행 가능한 결과물을 남기면서 범위를 조절하는 방식**으로 쓰는 게 좋아.

너희 조건을 기준으로 잡을게.

* 인원: **임베디드 입문자 5명**
* 기간: **7일**
* 플랫폼: **APACHE6**
* AI: SDK 제공 **Trichimera 활용**
* 최종 목표: `Detection + Freespace + Lane → Risk Engine → SAFE/WARNING/DANGER → Wayland 시각화`
* Linux 시스템 프로그래밍 요소는 **MVP 완성 후 추가**
* APACHE6 보드 자체에는 카메라 입력뿐 아니라 UART, I2C, CAN, SPI, PWM 등의 인터페이스가 있으므로, 마지막 단계에서 하드웨어 출력으로 확장할 여지도 있다. 

## 0. 일주일 전체 전략

```text
Day 1       Day 2        Day 3        Day 4
환경/SDK → AI 출력 → Risk Engine → 전체 연결
                            │
                            ▼
                         ★ MVP ★

Day 5              Day 6             Day 7
Linux 구조 개선 → 안정화/테스트 → 발표/데모
      │
      └─ 실패하면 기존 MVP로 즉시 복귀
```

**Day 4에 이미 데모가 돌아가야 한다**는 게 핵심이야.

Day 5~7에 처음 전체 시스템을 붙이려고 하면 초보자 팀에게 상당히 위험해.

---

# Sprint 0 — 프로젝트 시작 전

### Definition of Done부터 정한다

최소 성공 조건(MVP):

```text
영상 입력
 ↓
Trichimera
 ├─ Detection
 ├─ Freespace
 └─ Lane
 ↓
Risk Engine
 ↓
SAFE / WARNING / DANGER
 ↓
Wayland 화면 출력
```

**MVP에서 제외**

* epoll
* timerfd
* signalfd
* 복잡한 IPC
* 실제 차량 제어
* 새로운 AI 학습
* 복잡한 거리 추정
* TTC

시간이 남으면 추가하는 **Stretch Goal**:

```text
1순위 : 프로세스 분리 + Shared Memory
2순위 : LED/Buzzer
3순위 : 로그 저장
```

---

# Day 1 — Bring-up Sprint

### 목표

> **APACHE6에서 기존 Trichimera 예제를 실행한다.**

첫날은 개발보다 **SDK 파악**이 중요해.

### 담당

**A — AI/SDK**

Trichimera 예제 실행.

```text
Camera
 ↓
Trichimera
 ↓
Detection
Freespace
Lane
```

각 출력이 어느 코드에서 나오는지 확인.

**B — SDK 분석 보조**

A와 함께 다음을 기록.

```text
Detection 결과 구조체
bbox 좌표 형식
class
confidence

Freespace 출력 형식
Lane 출력 형식
```

**C/D — Risk Engine**

APACHE6 없이 PC에서 먼저 작성.

```cpp
struct Object {
    int x1;
    int y1;
    int x2;
    int y2;
    int class_id;
};

enum Risk {
    SAFE,
    WARNING,
    DANGER
};

Risk evaluate_risk(...);
```

가짜 bbox를 넣어서 테스트.

**E — 통합/빌드**

Git 저장소와 디렉터리 구성.

```text
adas/
├── perception/
├── risk/
├── display/
├── common/
├── test/
└── README.md
```

### Day 1 DoD

최소한 이것이 보여야 한다.

```text
APACHE6
   ↓
Trichimera example
   ↓
화면에 inference 결과
```

**여기서 실패하면 Day 2도 SDK 분석에 투자한다.**

---

# Day 2 — Perception Interface Sprint

### 목표

> **AI 결과를 우리 코드에서 사용할 수 있게 만든다.**

AI 모델 내부는 건드리지 않는다.

### 통합 데이터 구조 정의

예:

```cpp
struct Detection {
    int class_id;
    float confidence;

    int x1;
    int y1;
    int x2;
    int y2;
};

struct PerceptionResult {
    Detection objects[32];
    int object_count;

    // freespace/lane 정보
};
```

중요한 건 **AI 코드와 Risk Engine 사이에 인터페이스를 만드는 것**이야.

```text
Trichimera

   ↓

PerceptionResult

   ↓

Risk Engine
```

### 동시에 Risk 팀은

가장 단순한 위험 판단부터 만든다.

예를 들어 화면을:

```text
┌─────────────────────────────┐
│                             │
│          SAFE               │
│                             │
│       ┌──────────┐          │
│       │ WARNING  │          │
│       │          │          │
│       │ DANGER   │          │
└───────┴──────────┴──────────┘
```

처럼 영역으로 나눈다.

객체 bbox의 **bottom-center**:

```text
      ┌───────┐
      │  CAR  │
      │       │
      └───●───┘
          ↑
    bottom-center
```

를 기준점으로 사용한다.

### Day 2 DoD

```text
실제 Trichimera 결과
        ↓
PerceptionResult
        ↓
printf() 등으로 확인
```

여기까지 되면 상당히 중요한 고비를 넘은 거야.

---

# Day 3 — Risk Engine Sprint

### 목표

> **AI 결과를 ADAS 판단으로 바꾼다.**

여기가 프로젝트의 핵심 구현이야.

처음부터 복잡한 알고리즘을 만들지 않는다.

### Version 1

```cpp
if (object in danger_zone)
    DANGER;
else if (object in warning_zone)
    WARNING;
else
    SAFE;
```

### Version 2

Freespace 결합.

```text
Object
   ↓
Freespace 내부인가?
   ├─ NO  → SAFE
   │
   └─ YES
       ↓
    위험 영역?
       ↓
SAFE / WARNING / DANGER
```

### Version 3

Lane까지 성공했다면:

```text
Object
   ↓
Freespace?
   ↓
Ego Lane?
   ↓
Distance Zone?
   ↓
Risk Level
```

### 반드시 Unit Test도 만든다

예:

```text
Case 1
차량 + 내 차선 + 멀리
→ SAFE

Case 2
차량 + 내 차선 + 중간
→ WARNING

Case 3
보행자 + 내 차선 + 가까움
→ DANGER

Case 4
차량 + 옆 차선
→ SAFE
```

### Day 3 DoD

```text
PerceptionResult
       ↓
evaluate_risk()
       ↓
SAFE/WARNING/DANGER
```

가 **AI 없이도 독립적으로 테스트 가능**해야 한다.

---

# Day 4 — Integration Sprint ★

가장 중요한 날.

### 목표

> **무조건 MVP 완성**

전체를 처음 연결한다.

```text
Camera
 ↓
Trichimera
 ↓
Detection ─┐
Freespace ─┼─→ Risk Engine
Lane ──────┘
                ↓
        SAFE/WARNING/DANGER
                ↓
             Wayland
```

화면에는 최소한:

```text
┌──────────────────────────────┐
│ STATUS : WARNING             │
│                              │
│       ┌─────────┐            │
│       │  CAR    │            │
│       └─────────┘            │
│                              │
│       /          \           │
│      / FREESPACE  \          │
│     /              \         │
└──────────────────────────────┘
```

정도만 표시돼도 된다.

### 오후 6시 기준 Scope Freeze

**이때부터 새로운 핵심 기능 추가 금지.**

현재 동작하는 상태를 Git에:

```text
v0.1-mvp
```

같이 태그해 둔다.

### Day 4 DoD

**카메라 앞에 물체를 놓으면 화면의 위험 상태가 실제로 변한다.**

이게 되면 프로젝트는 사실상 성공이다.

---

# Day 5 — Linux System Programming Sprint

이제 너희가 보여주고 싶어 했던 **Linux 시스템 프로그래밍 요소**를 넣는다.

### 목표

기존:

```text
[ 하나의 프로세스 ]

AI
Risk
Display
```

에서 가능하면:

```text
Perception Process
        │
        │ POSIX Shared Memory
        ↓
ADAS Process
 ├─ Risk Engine
 └─ Display
```

로 변경.

예를 들면:

```text
/dev/shm/adas_perception
```

공유 데이터:

```cpp
struct SharedPerception {
    uint64_t frame_id;
    Detection objects[32];
    int object_count;
    ...
};
```

여기서 배울 수 있는 것:

```text
fork()/exec() 또는 별도 프로세스 실행
shm_open()
ftruncate()
mmap()
munmap()
```

### 중요한 규칙

**3~4시간 해보고 안 되면 포기한다.**

Day 4 버전으로 돌아간다.

```text
git checkout v0.1-mvp
```

시스템 프로그래밍을 보여주겠다고 완성된 ADAS를 망가뜨리면 안 돼.

---

# Day 6 — Stabilization Sprint

### 목표

> **기능 추가 금지. 버그를 잡는다.**

이날부터는 욕심을 버려야 해.

테스트 시나리오를 만든다.

| 상황          | 기대 결과          |
| ----------- | -------------- |
| 객체 없음       | SAFE           |
| 멀리 차량       | SAFE           |
| 접근 차량       | WARNING        |
| 가까운 차량      | DANGER         |
| 옆 차선 차량     | SAFE           |
| 주행 영역 밖 보행자 | SAFE           |
| 주행 영역 내 보행자 | WARNING/DANGER |

그리고 가능하다면 간단한 로그도 추가.

```text
[Frame 312]
Object: CAR
BBox: (312, 210, 450, 420)
Freespace: YES
EgoLane: YES
Zone: WARNING
Risk: WARNING
```

이런 로그는 발표할 때도 꽤 유용하다.

### 시간이 남는다면

그때 LED/Buzzer.

```text
SAFE
→ LED OFF

WARNING
→ LED blinking

DANGER
→ LED + Buzzer
```

하지만 **이건 없어도 된다.**

---

# Day 7 — Demo Sprint

마지막 날에는 코딩하지 않는다고 생각하는 게 좋아.

### 오전

실제 데모를 **10번 반복**한다.

```text
부팅
↓
프로그램 실행
↓
카메라 입력
↓
객체 인식
↓
WARNING
↓
DANGER
↓
종료
```

10번 중 9~10번 성공하는지 확인.

### 오후

발표 자료와 영상 준비.

발표 흐름도 기술 나열보다 다음 순서가 좋아.

```text
① 문제

단순 객체 Detection만으로
실제 충돌 위험을 판단할 수 없다.

        ↓

② 해결 아이디어

Detection
+
Freespace
+
Lane

        ↓

③ Risk Engine

객체 위치
+
주행 가능 영역
+
차선
+
위험 영역

        ↓

④ 결과

SAFE
WARNING
DANGER

        ↓

⑤ Linux 설계

AI 처리와
ADAS 판단 모듈 분리
```

---

# 5명 역할 분담

일주일 동안 담당자를 계속 바꾸지 않는 걸 추천해.

| 담당    | 주요 업무                           |
| ----- | ------------------------------- |
| **A** | Trichimera/NPU                  |
| **B** | Trichimera 출력 + 영상 파이프라인        |
| **C** | Risk Engine                     |
| **D** | Wayland/UI                      |
| **E** | Linux IPC + Build + Integration |

다만 **E는 Day 1부터 shared memory를 만들면 안 돼.**

Day 1~4에는 통합 담당자로 움직이고 Day 5부터 Linux 구조 개선을 담당하는 게 좋아.

---

# 매일 Scrum도 아주 짧게

아침 **10분**만 한다.

각자 세 가지만 말한다.

> 어제 무엇을 했는가?
> 오늘 무엇을 할 것인가?
> 막힌 것이 무엇인가?

저녁에는 **15분 Sprint Review**.

```text
실제로 실행해서 보여줄 수 있는가?
          ↓
       YES / NO
          ↓
내일 범위를 유지할 것인가?
축소할 것인가?
```

문서로 "80% 구현" 같은 표현은 사용하지 말고 **실행되냐/안 되냐**로 판단하는 게 좋다.

---

# 그리고 Kill Rule을 정해두자

초보자 5명이 일주일 프로젝트를 하면 이게 굉장히 중요해.

**Day 2까지 Lane 연동 실패**

→ Lane 제외하고 `Detection + Freespace`로 진행.

**Day 3까지 Freespace 연동 실패**

→ `Detection + 화면상의 Risk Zone`으로 MVP 완성.

**Day 4까지 Wayland Overlay 수정 실패**

→ 기존 SDK 화면을 최대한 활용하고 위험 상태만 추가.

**Day 5 Shared Memory 실패**

→ 단일 프로세스 버전으로 복귀.

**Day 6부터**

→ 신규 기능 개발 금지.

즉, 프로젝트가 실패하는 방향은

```text
Detection
+ Freespace
+ Lane
+ Shared Memory
+ Socket
+ epoll
+ GPIO
+ Buzzer
+ Logging
+ ...
          ↓
       전부 미완성
```

이고 우리가 원하는 방향은

```text
        Day 2
     최소 기능 확보
          ↓
        Day 4
       MVP 완성
          ↓
        Day 5
    시스템 요소 추가
          ↓
        Day 6
       안정화
          ↓
        Day 7
     확실한 데모
```

야.

**특히 너희 팀에서는 Day 4 MVP 완성을 절대선으로 잡는 걸 추천해.** Linux 시스템 프로그래밍 요소가 생각보다 적게 들어가더라도, **실제로 작동하는 APACHE6 ADAS를 먼저 완성한 뒤 그 구조를 개선했다**는 프로젝트 스토리가 훨씬 설득력 있어.
