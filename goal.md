

**APACHE6 Trichimera 기반 실시간 통합 위험 판단 ADAS**

> 카메라 프레임을 V4L2로 입력받아 APACHE6 NPU의 Trichimera 모델로 객체·주행 가능 영역·차선을 동시에 추론한다. 각 AI 결과를 하나의 위험 판단 모듈에서 결합하여 객체가 주행 가능 영역과 내 차선 위험 영역에 진입했는지를 판단하고, SAFE / WARNING / DANGER 상태를 실시간으로 화면에 표시한다.
