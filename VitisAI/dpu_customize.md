# DPU + Custom Logic build

`DPU` 와 custom logic build 를 위한 환경 구축

## KV260 DPU 개발 절차

### 1. 개발 환경 구축:

* Download [DPUCZDX8G_SGD](https://drive.google.com/uc?export=download&id=1trgXKtgrAkagofGQxBAUsXjChU7UjLvd)



### 2. Build `dpu.xo`

* Woring in `~/DPUCZDX8G/prj/Vitis`

  ```bash
  cd ~/DPUCZDX8G/prj/Vitis
  ```

* DPU configuration
  * `dpu_conf.vh`
  * `./config_file/system_conf`
* Build `dpu.xo`

  ```bash
  make all KERNEL=DPU DEVICE=SOM
  ls build_output
  ```

### 3. Design costom logic w/ dpu ready -> Export platform

> `design_1_wrapper.xsa`를 생성

### 4. Build `.xclbin` with `v++`

```bash
v++ --link --target hw \
--platform </DATA/home/edu019/work/prj/prj_dpu/design_1_wrapper.xsa> \
--config ./config_file/system_conf \
--save-temps --temp_dir ./build_temp \
--output ./build_output/dpu.xclbin \
./binary_container_1/dpu.xo
```

### 5. copy `.bit` & `.hwh` to `./build_ouput` folder

```bash
cp ./build_temp/link/vivado/vpl/prj/prj.gen/sources_1/bd/design_1/hw_handoff/design_1.hwh build_output/dpu.hwh

cp ./build_temp/link/vivado/vpl/prj/prj.runs/impl_1/design_1_wrapper.bit build_output/dpu.bit
```

### 6. `.xmodel` recompile

* Pynq default dpu와 새로 생성한 dpu가 아래 참고사항과 같이 다소 차이가 있다.
* 서로 맞지 않을 경우 문제가 발생
* dpu와 xmodel은 각각 architecture의 고유번호인 `fingerprint` 가지고 있다.
* 새로 만든 dpu의 `fingerprint`에 맞게 `xmodel`을 다시 만들어야 한다.
  * `compil-only` option이 추가된 [quantize.py](https://drive.google.com/uc?export=download&id=1f-oP1NdLb2D2SadZefSNrkFMSmVbk-j4)
  * [arch.json](https://drive.google.com/uc?export=download&id=1rWVqSXz2WdvGePZbPl02HCY9-F6466Nh)을 다운 받아 docker project folder copy
  * build w/ options

    ```bash
    python quantize.py --compile-only --arch <arch/arch.json>
    ```

### 7. Test on board

---

## 참고 사항

B4096 아키텍처를 사용할 때 Vivado/Vitis Implementation 단계에서 Place & Route 실패(Timing violation 또는 Resource over-utilization)가 발생했던 이유는 **Kria KV260(xck26) MPSoC 칩 내부의 하드웨어 리소스 한계** 때문입니다.

공식 PYNQ DPUEZ / DPU-TRD 이미지에서 기본 제공되는 B4096 비트스트림과, 사용자가 직접 Vivado/Vitis `v++`로 Customizing하여 빌드하는 B4096에는 **결정적인 차이점**이 존재합니다.

---

### 1. 결정적 원인: Kria KV260(xck26)의 BRAM 및 DSP 리소스 한계

`xck26` FPGA는 UltraScale+ 라인업 중 비교적 작은 소형 MPSoC 디바이스입니다.

| DPU Architecture | Peak Ops (FLOPs/cycle) | BRAM (18Kb) 소요량 | DSP Block 소요량 | KV260(xck26) 수용 여부 |
| --- | --- | --- | --- | --- |
| **B2304** | 2304 | 적음 | ~576개 | **여유로움** |
| **B3136** | 3136 | 보통 | ~784개 | **적정 수준 (추천)** |
| **B4096** | 4096 | **매우 높음** | **1,024개 (100% 육박)** | **적색 경보 (Timing Failure / Overflow)** |

#### ① DSP Block 및 Timing Closure (타이밍 맞춤 실패)

B4096 구조는 내부 연산 엔진이 커서 **1,000개가 넘는 DSP Block**을 사용합니다. `xck26` 디바이스 전체 DSP의 약 **80~90% 이상**을 DPU 하나가 독점하게 되는데, 로직 배치가 칩 전체에 빽빽하게 흩어지면서 **라우팅(Routing) 길어져 300MHz/600MHz 클록 타이밍(Setup Time Violation)을 맞추지 못해 Implementation 실패**가 발생합니다.

#### ② BRAM 부족 및 Low-Resource 옵션 미적용

B4096을 기본 옵션으로 생성하면 Feature Map 및 Weight 버퍼용 BRAM 소요량이 `xck26`의 총 BRAM 개수를 초과합니다.

---

### 2. 기존 공식 바이너리는 어떻게 B4096 빌드에 성공했는가?

Xilinx 공식 빌드나 pre-built `.bit` 파일의 B4096 아키텍처는 **매우 빡빡하게 튜닝된 전용 옵션**이 들어가 있습니다.

1. **`RAM_USAGE = LOW` 및 URAM 활용**: BRAM 사용량을 줄이기 위해 URAM(UltraRAM)을 적극 활용하도록 설정
2. **DSP Cascade 및 Direct Connection 설정**: DSP 간의 배선을 줄이기 위해 Cascade 방식을 고정
3. **Vivado Implementation Strategy 최적화**:
* `Performance_ExplorePostRoutePhysOpt`
* `Congestion_Overmarching` 등 최고 수준의 합성/배치 파이프라인 수동 지정



사용자가 Vitis `v++`에서 일반적인 config로 Customizing하여 빌드할 경우, 이러한 UltraScale+ 전용 BRAM/DSP 튜닝 옵션이 빠져 있으므로 Vivado가 라우팅 혼잡(Congestion)을 해결하지 못하고 Implementation을 중단(Fail)시키는 것입니다.

---

### 3. Customizing 빌드 시 권장 대안

KV260 보드에서 직접 커스텀 DPU를 빌드하여 안정적으로 구동하려면 다음 **2가지 방법** 중 하나를 선택하는 것이 표준적인 해결책입니다.

#### 방법 A. B3136 아키텍처로 커스텀 빌드 (가장 추천)

KV260 환경에서 성능과 리소스 안정성이 가장 뛰어난 아키텍처는 `B3136`입니다.

* `arch.json`:

```json
{
    "fingerprint": "0x101000056010406"
}

```


* B3136으로 `v++` 커스텀 빌드 후, 해당 fingerprint로 `.xmodel`만 재컴파일하여 올려주시면 Implementation 오류 없이 깔끔하게 동작합니다.

### 방법 B. B4096 커스텀 빌드가 꼭 필요할 경우 (`dpu_conf.vh` 옵션 수정)

B4096을 커스텀 빌드하려면 리소스 사용량을 강제로 줄이는 옵션을 넣어주어야 합니다.

* `RAM_USAGE_LOW` 설정
* `URAM_ENABLE` 활성화
* `DSP_48` Cascade 설정 적용

---

### 결론

기존에 아무 문제 없이 사용하셨던 B4096 비트스트림은 **Xilinx 엔지니어들이 리소스 배치를 극한으로 튜닝하여 빌드해둔 사전 정의 바이너리**였기 때문입니다.

현재 커스텀 하드웨어 빌드(`dpu.xclbin`)에 성공하신 **`0x101000056010406` (B3136/B2304 계열)** 지문 기반으로 PC에서 `.xmodel`만 다시 컴파일하여 실행하시는 것이 가장 안정적이고 속도 면에서도 KV260에 최적화된 방법입니다!