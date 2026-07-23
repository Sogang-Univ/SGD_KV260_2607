# Kria Pynq DPU example: YOLOv3

## 전체 흐름

1. 하드웨어/모델 준비: DPU overlay + YOLOv3 모델 로드
2. 유틸 준비: 전처리, 디코딩, NMS, 시각화 함수 정의
3. 러너/텐서 정보 확인: 입력·출력 shape와 버퍼 구성
4. 단일 추론 함수 구성: run()에서 end-to-end 처리
5. 검증: 단일 이미지 시각화
6. 성능 측정: 전체 이미지 배치 처리 후 FPS 계산

---

## VART 란?
### 1. VART(Versal/Zynq AI Engine Run-Time) 개요

**VART**는 자일링스(AMD Xilinx)의 Zynq UltraScale+ MPSoC 및 KV260과 같은 보드에서 DPU(Deep Learning Processing Unit)를 활용해 인공지능 모델을 효율적으로 실행할 수 있도록 지원하는 **런타임 라이브러리 및 API 세트**입니다.

* **역할:** 호스트 애플리케이션(Python 또는 C++)과 하드웨어 가속기(DPU) 사이의 인터페이스 역할을 담당합니다.
* 주요 기능:
* DPU가 수행할 입력 텐서와 출력 텐서의 구조(Shape, Data Size) 파악
* 메모리 할당 및 버퍼 관리(`input_data`, `output_data`)
* DPU 태스크(Task) 생성 및 실행 제어

---

### 2. 이미지 코드 라인바이라인 분석

제공해주신 이미지의 코드 블록(In [14] ~ In [20])은 YOLOv3 모델을 DPU에서 실행하기 위해 **입출력 텐서의 구조를 확인하고 데이터를 담을 메모리 버퍼를 준비하는 과정**입니다.

---

#### `In [14]`

```python
dpu = overlay.runner

```

* **설명:** FPGA 오버레이(Bitstream)에 로드된 DPU 러너(Runner) 객체를 `dpu` 변수에 할당합니다. 이 객체를 통해 DPU에 모델 연산을 요청할 수 있습니다.

---

#### `In [15]` & `In [16]`

```python
inputTensors = dpu.get_input_tensors()
outputTensors = dpu.get_output_tensors()

```

* **설명:**
* `dpu.get_input_tensors()`: 모델에 입력되는 데이터의 정보(텐서) 목록을 가져옵니다.
* `dpu.get_output_tensors()`: 모델 연산 결과로 나오는 출력 데이터의 정보(텐서) 목록을 가져옵니다. YOLOv3는 보통 여러 개의 출력 층을 가집니다.



---

#### `In [17]`

```python
shapeIn = tuple(inputTensors[0].dims)

```

* **설명:** 첫 번째 입력 텐서(`inputTensors[0]`)의 차원 크기(Dimensions, 예: 배치 크기, 높이, 너비, 채널 수)를 가져와서 튜플(`tuple`) 형태로 `shapeIn`에 저장합니다.

---

#### `In [18]`

```python
shapeOut0 = (tuple(outputTensors[0].dims)) # (1, 13, 13, 75)
shapeOut1 = (tuple(outputTensors[1].dims)) # (1, 26, 26, 75)
shapeOut2 = (tuple(outputTensors[2].dims)) # (1, 52, 52, 75)

```

* **설명:** YOLOv3의 특징인 3개의 서로 다른 스케일(Grid size)에 따른 출력 텐서의 차원 크기를 각각 추출합니다.
* 주석에 나온 대로 각각 `(1, 13, 13, 75)`, `(1, 26, 26, 75)`, `(1, 52, 52, 75)`와 같은 형태의 텐서 구조를 가집니다.



---

#### `In [19]`

```python
outputSize0 = int(outputTensors[0].get_data_size() / shapeIn[0]) # 12675
outputSize1 = int(outputTensors[1].get_data_size() / shapeIn[0]) # 50700
outputSize2 = int(outputTensors[2].get_data_size() / shapeIn[0]) # 202800

```

* **설명:** 각 출력 텐서의 전체 데이터 크기에서 배치 크기(`shapeIn[0]`)를 나누어, **배치당(Per image) 필요한 출력 데이터 요소의 개수**를 계산합니다. 후처리(Post-processing) 단계에서 버퍼를 다룰 때 유용하게 쓰입니다.

---

#### `In [20]`

```python
input_data  = [np.empty(shapeIn, dtype=np.float32, order="C")]
output_data = [np.empty(shapeOut0, dtype=np.float32, order="C"),
               np.empty(shapeOut1, dtype=np.float32, order="C"),
               np.empty(shapeOut2, dtype=np.float32, order="C")]
image = input_data[0]

```

* **설명:**
* `input_data`: DPU에 입력할 이미지 데이터를 담을 NumPy 빈 배열(Buffer)을 생성합니다. (`shapeIn` 크기, `float32` 타입, C언어 메모리 배치 기준 `order="C"`)
* `output_data`: DPU가 연산한 결과를 받아오기 위해 3개의 출력 스케일에 맞춘 빈 배열들을 리스트 형태로 생성합니다.
* `image = input_data[0]`: 추후 전처리된 실제 이미지 배열을 쉽게 넣을 수 있도록 첫 번째 입력 데이터 공간을 `image` 변수로 참조합니다.
