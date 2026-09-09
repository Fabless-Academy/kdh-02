좋은 질문이에요! 참고 자료에서 확인할 수 있는 APACHE6용 `.aiwbin` 네트워크 파일들을 정리해 드릴게요.자료를 다 확인했어요. 정리해 드릴게요!

---

## APACHE6에서 사용 가능한 AI 모델

크게 **태스크(용도)** 기준으로 나눌 수 있어요.

---

### 객체 탐지 (Detection)

| 모델 파일명 | 입력 크기 | 클래스 수 | 비고 |
|---|---|---|---|
| `yolov5s_coco_640x384_apache6sr250_aiw4939.aiwbin` | 640×384 | 6 | per, bik, car, mot, bus, tru |
| `yolov8n_coco_640x384_apache6sr250_aiw4939.aiwbin` | 640×384 | 6 | per, bik, car, mot, bus, tru |
| `yolov8s_coco_640x384_apache6sr250_aiw4939.aiwbin` | 640×384 | 6 | per, bik, car, mot, bus, tru |
| `yolov8n_ncdl_640x384_apache6sr250_aiw4939.aiwbin` | 640×384 | 6 | per, car, bus, tru, cycle, mot |
| `yolov11s_coco9_640x384_3outputs_minmax.aiwbin` | 640×384 | 9 | person, bicycle, car, bike, bus, train, truck, sign, stop |
| `peleedet_3class_640x384_apache6sr250_aiw4939.aiwbin` | 640×384 | 3 | bik, bus, car |
| `peleedet_10class_640x384_apache6sr250_aiw4939.aiwbin` | 640×384 | 10 | bik, bus, car, mot, per, rid, t-l, t-s, tra, tru |

---

### 세그멘테이션 (Segmentation)

| 모델 파일명 | 입력 크기 | 클래스 수 | 비고 |
|---|---|---|---|
| `peleeseg_640x384_apache6sr250_aiw4939.aiwbin` | 640×384 | 2 | road, background |
| `pplite_2class_640x384_apache6sr250_aiw4939.aiwbin` | 640×384 | 2 | background, freespace |

---

### 차선 인식 (Lane Detection)

| 모델 파일명 | 입력 크기 | 비고 |
|---|---|---|
| `ufld_a6sr250_aiw4939.aiwbin` | - | 최대 4개 차선 검출 |

---

### 멀티태스크 (Multi-task)

| 모델 파일명 | 입력 크기 | 비고 |
|---|---|---|
| `trichimera_640x384_a6sr_aiw4939.aiwbin` | 640×384 | Detection + Segmentation + Lane을 한 번에 처리 |

---

### 참고 사항

- 모든 모델은 **`.aiwbin`** 포맷으로 APACHE6의 AIWare 런타임 위에서 동작해요.
- 각 모델에는 대응되는 **`.ini` 설정 파일**이 있어서, 클래스 이름·색상·임계값 등을 거기서 관리해요.
- **TFLite 모델**도 별도로 지원되는데, `ssd_mobilenet_v2_coco.tflite` 예제가 확인돼요 (GPU 경로로 실행).
- `trichimera`는 내부적으로 yolov8n + pplite + ufld 세 모델을 묶은 구조예요.
