좋아. 그럼 **5명 × 5일** 기준으로, 실제로 첫날부터 뭘 해야 하는지까지 내려서 계획을 짜보자. 핵심은 “5일 동안 기능을 많이 넣는 것”이 아니라 **Day 1에 SDK baseline을 확보하고, Day 2~3에 팀이 병렬 작업할 수 있는 인터페이스를 고정하는 것**이야.

자료상 APACHE6 SDK에는 `applications` 아래 샘플 애플리케이션과 공통 라이브러리가 있고, Wayland NPU 샘플은 `V4L2 → render() → cnn_task → NPU → pp_task → Wayland/Overlay Mixer` 흐름이다. SDK 문서에도 application만 별도 `build.sh`로 빌드해서 `output/user_data`로 복사하고, NFS/SD/eMMC에서 실행하는 흐름이 제공된다.  

다만 첨부 문서에는 **Trichimera 샘플의 정확한 소스 파일명/함수명 전체가 공개되어 있지 않다.** 그래서 아래에서 `cnn_task`, `pp_task`, `render()`처럼 문서에서 확인되는 함수명은 그대로 쓰고, 나머지 파일명은 실제 SDK 소스에서 첫날 확인해서 매핑해야 한다.

---

# 전체 목표

금요일 최종 실행 프로그램 하나를 만드는 걸 목표로 한다.

```text
Camera
  ↓
V4L2
  ↓
Trichimera @ NPU
  ↓
Detection + Freespace + Lane
  ↓
Risk Engine
  ↓
SAFE / WARNING / DANGER
  ↓
Wayland Overlay
```

그리고 소프트웨어 구조는 최종적으로 이렇게 나누자.

```text
src/
 ├─ ai/
 │   └─ ai_adapter.*
 │
 ├─ risk/
 │   ├─ geometry.*
 │   └─ risk_engine.*
 │
 ├─ system/
 │   ├─ shared_state.*
 │   ├─ perf_monitor.*
 │   └─ signal_handler.*
 │
 ├─ ui/
 │   └─ adas_overlay.*
 │
 └─ main / 기존 SDK application
```

실제 SDK 디렉터리를 마음대로 바꾸라는 뜻은 아니고, **논리적으로 이 단위로 담당 영역을 분리하자**는 의미다.

---

# 팀원 5명 역할

처음부터 아래처럼 고정하는 게 좋다.

| 팀원 | 주 담당               | 최종 산출물                       |
| -- | ------------------ | ---------------------------- |
| A  | AI/NPU             | `AIResult` 구조체까지 결과 전달       |
| B  | Geometry           | freespace/lane/point 판정      |
| C  | Risk               | SAFE/WARNING/DANGER 로직       |
| D  | UI                 | bbox/mask/lane/risk overlay  |
| E  | System/Integration | thread/mutex/signal/build/통합 |

여기서 E는 “남는 일 하는 사람”이 아니다. 오히려 **전체 프로그램 구조를 관리하는 사람**으로 잡아야 한다.

---

# 시작하기 전에 Git부터 정리

5일밖에 없으니 Git도 단순하게 한다.

```text
main
 ├─ feat/ai
 ├─ feat/geometry
 ├─ feat/risk
 ├─ feat/ui
 └─ feat/system
```

원칙은 세 개만.

1. `main`은 항상 실행 가능한 상태
2. 기능 하나 끝날 때마다 PR/merge
3. 여러 사람이 SDK 원본 파일 하나를 동시에 수정하지 않기

특히 기존 sample의 `main`이나 `render()`를 A, D, E가 동시에 수정하기 시작하면 충돌 때문에 시간이 날아간다.

그래서 **원본 application 수정 권한은 통합 담당 E가 중심**, 나머지는 가능하면 새 `.cpp/.h` 또는 `.c/.h` 모듈을 만든다.

---

# DAY 1 — “무조건 기존 AI 데모 실행”

Day 1 목표는 개발이 아니다.

**카메라 → Trichimera → 화면 출력이 정상 동작한다는 걸 확보하는 날**이다.

SDK에는 Wayland NPU Application 자체가 카메라 입력을 받아 NPU에서 Object Detection + Freespace + Lane Detection을 수행하고 결과를 Wayland로 출력하는 샘플로 설명돼 있다. 

## 09:00–10:00 전체 팀

### ① 개발환경 확인

호스트:

```bash
pwd
git status
git branch
```

SDK 상위 구조 확인:

```bash
ls
```

문서 기준으로 대략:

```text
linux-sdk-apache6/
 ├─ applications
 ├─ arm-trusted-firmware
 ├─ buildroot
 ├─ linux-kernel
 ├─ modules
 ├─ tools
 ├─ u-boot
 └─ ...
```

이 구조는 Quick Guide에도 명시돼 있다. 

---

### ② applications 구조 확인

```bash
cd applications
find . -maxdepth 2 -type f | less
```

특히 찾아야 할 것:

```bash
find . -iname '*npu*'
find . -iname '*wayland*'
find . -iname '*trichimera*'
find . -iname '*.aiwbin'
find . -iname '*.ini'
```

소스에서도:

```bash
grep -R "cnn_task" -n .
grep -R "pp_task" -n .
grep -R "render(" -n .
```

**이 세 grep 결과는 팀 채팅에 바로 공유한다.**

왜냐하면 문서에서 확인되는 핵심 흐름이 바로 이 부분이기 때문이다. 

---

# 10:00–12:00

## A — AI 담당

찾는다.

```text
Trichimera model load 위치
NPU init 위치
cnn_task
inference 호출부
post-processing callback
Detection 결과 구조
Freespace 결과 구조
Lane 결과 구조
```

코드를 수정하지 말고 우선 메모한다.

예:

```text
AI init:
xxx.cpp:142

cnn_task:
xxx.cpp:381

post process:
xxx.cpp:511
```

그리고 **출력 타입을 종이에 그린다.**

목표:

```cpp
struct AIResult {
    ObjectInfo objects[...];
    int object_count;

    FreespaceInfo freespace;
    LaneInfo lane;
};
```

아직 구현 안 해도 된다.

---

## B — Geometry 담당

A와 같이 post-processing 코드를 본다.

확인 대상:

```text
Segmentation 결과가
- 픽셀 mask인가?
- polygon인가?
- overlay용 buffer인가?

Lane 결과가
- point 배열인가?
- polynomial coefficient인가?
- lane index인가?
```

이걸 모르면 geometry 모듈을 설계할 수 없다.

---

## C — Risk 담당

이때는 SDK 건드리지 않는다.

PC에서 독립적으로 risk API부터 설계.

```cpp
enum RiskLevel {
    RISK_SAFE,
    RISK_WARNING,
    RISK_DANGER
};
```

```cpp
RiskLevel evaluateRisk(...);
```

그리고 종이에 판단 흐름 작성.

```text
Object
 ↓
bottom center
 ↓
Freespace?
 NO → SAFE
 YES
 ↓
Ego lane?
 NO → SAFE
 YES
 ↓
Danger zone?
 YES → DANGER
 NO
 ↓
Warning zone?
 YES → WARNING
 NO → SAFE
```

---

## D — UI 담당

기존 overlay 코드를 찾는다.

```bash
grep -R "bbox" -n .
grep -R "overlay" -n .
grep -R "draw" -n .
```

확인:

```text
bbox를 어디서 그리는가
text drawing 지원 여부
freespace 색상은 어디서 입히는가
lane drawing 함수는 무엇인가
```

Day 1에는 변경하지 않는다.

---

## E — System 담당

전체 실행 흐름을 추적한다.

```text
main
 ↓
camera init
 ↓
V4L2
 ↓
thread create
 ↓
render
 ↓
NPU
 ↓
post-process
 ↓
Wayland
 ↓
cleanup
```

그리고 thread 관련 코드 검색:

```bash
grep -R "pthread_create" -n .
grep -R "pthread_mutex" -n .
grep -R "pthread_cond" -n .
```

---

# 13:00–15:00

## 기존 sample 빌드

전체 SDK를 매번 빌드하지 않는다.

문서에서도 application에는 별도 `build.sh`가 제공되며 빌드 결과가 `output/user_data`로 복사된다고 되어 있다. 

예:

```bash
cd applications
./build.sh
```

빌드 성공하면 결과 확인.

```bash
ls ../output/user_data/applications
```

---

# 15:00–17:00

## 보드에서 baseline 실행

가능하면 NFS를 추천한다.

문서상 NFS 실행 흐름은:

```bash
mount -t nfs <PC_IP>:/home/... /mnt -o nolock
cd /mnt/user_data/applications
./run_app_xxx.sh
```

형태로 제공된다. 

NFS가 이미 세팅되어 있지 않으면 **첫날 NFS 설정 때문에 3시간 쓰지 말고 기존 방식 그대로 사용**해도 된다.

목표는 단 하나.

```text
Camera 화면 보임
Object bbox 보임
Freespace 보임
Lane 보임
```

스크린샷과 동영상 10초 저장.

**이게 Day 1 보험이다.**

---

# 17:00–18:00 — 전체 팀 회의

화이트보드에 실제 코드 기준으로 이것을 완성한다.

```text
              실제 파일 / 함수

Camera      → __________
V4L2        → __________
render      → __________
cnn_task    → __________
NPU         → __________
pp_task     → __________
Detection   → __________
Freespace   → __________
Lane        → __________
Wayland     → __________
```

그리고 Day 2부터 쓸 **공통 데이터 구조를 확정한다.**

---

# Day 1 종료 조건

아래 5개가 다 되어야 한다.

```text
[ ] 기존 Trichimera demo 실행
[ ] 빌드 방법 확정
[ ] 실행 방법 확정
[ ] AI output 데이터 위치 확인
[ ] 수정할 주요 파일/함수 위치 확인
```

하나라도 안 됐으면 밤에 새 기능 개발하면 안 된다.

---

