# Pynq 에서 PL logic 제어

직접 설계한 $\text{PL}$ 로직의 레지스터에 접근하려면, $\text{BaseOverlay('base.bit')}$ 대신 \*\*사용자 정의 오버레이($\text{Overlay}$ 클래스)\*\*를 사용해야 한다.

`BaseOverlay`는 미리 정의된 'base' 하드웨어 구성에 특화된 클래스이며, 커스텀 로직을 로드하려면 해당 로직의 하드웨어 정보를 담고 있는 파일을 명시적으로 지정해야 한다.

-----

## Custom 로직의 레지스터 Access

직접 설계한 $\text{PL}$ 로직의 레지스터에 접근하는 과정은 크게 **오버레이 로드**와 **$\text{IP}$ 객체 접근** 두 단계로 나뉜다.

### 1\. 커스텀 오버레이 정의 및 로드

먼저, $\text{Vivado}$ 또는 $\text{Vitis}$에서 생성한 $\text{FPGA}$ 구성 파일들(비트스트림 및 하드웨어 정보)을 지정하여 $\text{Overlay}$ 객체를 생성한다.

| 파일 유형 | 설명 |
| :--- | :--- |
| **`.bit` 파일** | $\text{FPGA}$에 다운로드할 $\text{PL}$ 로직의 비트스트림 파일 |
| **`.hwh` 파일** | $\text{Hardware HandOff}$ 파일로, $\text{PL}$ 내부의 **모든 $\text{IP}$ 이름, $\text{Base Address}$, 인터럽트 정보** 등이 $\text{XML}$ 형식으로 담겨 있다 |

```python
from pynq import Overlay

# 1. 파일 이름 정의
# Your_Design.bit과 Your_Design.hwh 파일은 KV260 보드로 copy
bitstream_name = 'Your_Design.bit'
hwh_name = 'Your_Design.hwh'

# 2. 커스텀 오버레이 로드
try:
    custom_ol = Overlay(bitstream_name, hwh_name=hwh_name)
    print(f"{bitstream_name} 로드 완료.")
except FileNotFoundError:
    print(f"오류: {bitstream_name} 또는 {hwh_name} 파일을 찾을 수 없습니다.")
    exit()

# 오버레이 로드 후 FPGA의 PL 영역에 Your_Design.bit의 로직이 활성화됩니다.
```

### 2\. IP 객체 및 레지스터 접근

Overlay 객체(`custom_ol`)가 로드되면, 해당 $\text{hwh}$ 파일에 정의된 $\text{AXI}$ $\text{IP}$들(사용자 정의 로직)에 이름으로 접근할 수 있다.

#### A. IP 이름 확인

Overlay 객체의 ip_dict 속성을 통해 PL에 있는 IP들의 이름을 확인할 수 있습니다.

```python
# 로드된 모든 IP 이름 확인
print("로드된 IP 리스트:")
print(custom_ol.ip_dict.keys())

# 예: IP 이름이 'my_custom_adder_0'이라고 가정
ip_name = 'my_custom_adder_0'
```

#### B. 레지스터 읽기 / 쓰기

$\text{IP}$ 이름으로 객체를 가져온 후, $\text{AXI}$ $\text{Lite}$ 인터페이스를 통해 레지스터를 access.

```python
# 1. 사용자 정의 IP 객체 가져오기
my_logic = custom_ol.ip_dict[ip_name]

# 2. Base Address로부터의 Register Offset 정의
# 이 값은 Vivado/Vitis에서 Address Editor에 정의한 레지스터의 OFFSET 값이어야 합니다.
OUTPUT_REGISTER_OFFSET = 0x08 

# 3. 레지스터 값 읽기 (Register Reading)
register_value = my_logic.read(OUTPUT_REGISTER_OFFSET)
print(f"IP '{ip_name}'의 OFFSET {OUTPUT_REGISTER_OFFSET:#x}에서 읽은 값: {register_value}")
```

