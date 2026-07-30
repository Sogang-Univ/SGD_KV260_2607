Xilinx(AMD) Kria KV260의 DPU(Deep Learning Processing Unit) 오버레이에서 AI 모델을 추론하려면, 기존 모델(PyTorch, TensorFlow 등)을 DPU 전용 명령어 집합인 **`.xmodel`** 포맷으로 변환해야 합니다.


# [Vitis AI Overview](https://docs.amd.com/r/2.5-English/ug1414-vitis-ai/Vitis-AI-Overview)

---

## 1. `.xmodel` 변환 환경의 구조와 원리

변환 작업은 KV260 보드 위에서 직접 수행하는 것이 아니라, PC(호스트 컴퓨터, Ubuntu 22.04)에서 **Vitis AI Docker 컨테이너**를 구동하여 진행합니다.

```
[학습된 모델 (FP32)]
       │ (PyTorch / TF / ONNX)
       ▼
┌────────────────────────────────────────────────────────┐
│  Vitis AI Quantizer (양자화기)                          │
│  - FP32 가중치를 INT8 정수형으로 변환 (Calibration 진행)   │
└────────────────────────────────────────────────────────┘
       │
       ▼
┌────────────────────────────────────────────────────────┐
│  Vitis AI Compiler (컴파일러)                           │
│  - Target DPU 아키텍처(예: DPUCZDX8G - B4096) 정보 참조 │
│  - DPU 가속 레이어와 CPU 연산 레이어 구분 및 컴파일    │
└────────────────────────────────────────────────────────┘
       │
       ▼
[.xmodel 생성] ──▶ KV260 보드로 전달 ──▶ Vitis AI Runtime(VART)으로 추론

```

* **Vitis AI Quantizer:** 부동소수점(FP32) 연산을 DPU가 빠르게 처리할 수 있는 정수형(INT8)으로 변환합니다.
* **Vitis AI Compiler:** KV260의 DPU IP 설정(`arch.json` 또는 ISA 사양)을 참조하여 DPU가 이해할 수 있는 바이너리 명령어로 직렬화합니다.
* **Docker 환경을 사용하는 이유:** C++ 바인딩, 특정 PyTorch/TensorFlow 버전, Xilinx 프레임워크 라이브러리의 버전 의존성이 얽혀 있어 Xilinx 공식 Docker 이미지로 환경을 통일하는 것이 가장 안정적입니다.

---

## 2. Ubuntu 22.04 설치 및 환경 구축 과정

### Step 1. 필수 기본 패키지 및 Docker 설치

1. **기존 환경 정리 및 필수 패키지 설치:** 저장소 등록에 필요한 기본 도구를 설치합니다.
터미널에 아래 명령어를 복사하여 실행합니다.

    ```bash
    sudo apt update
    sudo apt install -y ca-certificates curl gnupg
    ```


2. **Docker 공식 GPG 키 등록:** 패키지 위변조 방지를 위한 보안 키를 추가합니다.
GPG 키를 보관할 디렉토리를 만들고 Docker 공식 키를 다운로드합니다.

    ```bash
    sudo install -m 0755 -d /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    sudo chmod a+r /etc/apt/keyrings/docker.gpg
    ```


3. **Docker 공식 APT 저장소 추가:** 시스템 패키지 목록에 Docker 저장소를 등록합니다.
현재 아키텍처와 Ubuntu 버전에 맞춰 저장소 경로를 추가합니다.

    ```bash
    echo \
      "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
      $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
      sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    ```

4. **패키지 목록 업데이트 및 Docker 설치:** 원래 실행하려고 했던 설치 명령어를 재시도합니다.
저장소를 새로 등록했으므로 `apt update`를 먼저 진행한 뒤 설치합니다.

    ```bash
    sudo apt update
    sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
    ```

5. **설치 확인 및 권한 설정 (선택):** sudo 없이 docker 명령어를 사용하기 위한 설정입니다.
설치가 잘 완료되었는지 확인하고, 현재 계정을 `docker` 그룹에 추가해 줍니다.

    ```bash
    # 설치 확인
    sudo docker run hello-world

    # sudo 없이 사용하기 위한 계정 권한 추가 (설정 후 재접속 필요)
    sudo usermod -aG docker $USER
    ```

    위 단계를 완료하면 `E: Package 'docker-ce' has no installation candidate` 에러 없이 깨끗하게 설치될 것입니다.

6. 현재 사용자 권한으로 sudo 없이 Docker를 사용하도록 설정

    ```bash
    sudo usermod -aG docker $USER
    # 아래 명령을 수행해야만 됨.
    newgrp docker
    ```

---

### Step 2. (선택사항) NVIDIA GPU 컨테이너 툴킷 설정

> **참고:** CPU 환경에서도 양자화 및 컴파일이 가능하지만, GPU를 활용하면 Quantization 속도가 대폭 향상됩니다. GPU를 사용하는 경우에만 실행하세요.

```bash
# 2-1. NVIDIA Container Toolkit 저장소 추가
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

# 2-2. 툴킷 설치 및 Docker 재시작
sudo apt update
sudo apt install -y nvidia-container-toolkit
sudo systemctl restart docker

```

---

### Step 3. Vitis AI 저장소 클론

Vitis AI의 최신 스크립트와 예제 코드, 툴체인을 가져옵니다.

```bash
# 3-1. 홈 디렉토리로 이동 후 Vitis AI 클론
cd ~
git clone https://github.com/Xilinx/Vitis-AI.git
cd Vitis-AI
```

---

### Step 4. KV260 타겟 설정 파일 (`arch.json`) 작성

KV260의 기본 DPU 아키텍처(보통 `DPUCZDX8G`의 B4096 설정)에 맞춰 컴파일러가 인식할 `arch.json` 파일을 작성합니다.

```bash
# 4-1. 설정 파일 저장을 위한 디렉토리 생성
mkdir -p ~/Vitis-AI/kv260_config
cd ~/Vitis-AI/kv260_config

# 4-2. vi로 arch.json 파일 생성 및 편집
vi arch.json

```

`vi` 편집기가 열리면 `i` 키를 눌러 **입력 모드**로 전환한 후 다음 내용(KV260 표준 B4096 DPU 설정)을 작성합니다.

```json
{
    "fingerprint":"0x101000016010407"
}

```

> **작성 완료 후:**
> 1. `Esc` 키를 누릅니다.
> 2. `:wq`를 입력하고 `Enter`를 눌러 저장 후 종료합니다.
> 
> 

---

### Step 5. Vitis AI Docker 이미지 가져오기 및 컨테이너 실행

프레임워크(PyTorch 또는 TensorFlow)에 따라 원하는 이미지를 선택하여 실행합니다. (여기서는 일반적으로 많이 쓰이는 **PyTorch** 기준 설명입니다.)

```bash
# 5-1. Vitis AI 루트 디렉토리로 이동
cd ~/Vitis-AI
```

```bash
# 5-2 boar의 vitis 환경 확인
ubuntu@kria:~$ dpkg -l | grep vart
ii  libvart                                 2.5.0                                   arm64        Vitis Ai RuntTime
```

```bash
# # 5-2. Docker 컨테이너 실행 스크립트 수정 (필요 시)
# vi docker_run.sh

# ```

# > `vi` 실행 시 확인/수정 사항:
# > 스크립트 내 기본 Docker 이미지가 최신 태그로 지정되어 있는지 확인할 수 있습니다. 수정이 필요 없으면 `:q` 입력 후 엔터로 바로 빠져나옵니다.

# ```bash
# # 5-3. CPU/GPU 환경에 따라 Docker 실행
# # PyTorch GPU 버전 실행 시:
# ./docker_run.sh xilinx/vitis-ai-pytorch-gpu:latest

# # PyTorch CPU 전용 버전 실행 시:
# ./docker_run.sh xilinx/vitis-ai-pytorch-cpu:latest

~/Vitis-AI$ ./docker_run.sh xilinx/vitis-ai-pytorch-cpu:2.5.0.1260
                                           Container File Notices

NOTICE - BY INVOKING THIS SCRIPT AND USING THE SOFTWARE INSTALLED BY THE SCRIPT, YOU AGREE ON BEHALF OF YOURSELF AND YOUR EMPLOYER (IF APPLICABLE) TO BE BOUND TO THE LICENSE AGREEMENTS APPLICABLE TO THE PACKAGES IDENTIFIED BELOW THAT YOU INSTALL BY RUNNING THE SCRIPT. YOU UNDERSTAND THAT THE INSTALLATION OF THE PACKAGES LISTED BELOW MAY ALSO RESULT IN THE INSTALLATION ON YOUR SYSTEM OF ADDITIONAL PACKAGES NOT LISTED BELOW IN ORDER TO OPERATE (EACH, A ‘DEPENDENCY’). ADVANCED MICRO DEVICES, INC., ON BEHALF OF ITSELF AND ITS SUBSIDIARIES AND AFFILIATES, DOES NOT GRANT TO YOU ANY RIGHTS OR LICENSES TO ANY SUCH DEPENDENCY. THE SCRIPT ITSELF IS LICENSED TO YOU SUBJECT TO THE FOLLOWING TERMS: 

Copyright © 2023 Advanced Micro Devices, Inc. All Rights Reserved. 
Press any key to continue...


```



실행하면 자동으로 Docker 이미지를 풀(Pull) 받아오고, 컨테이너 내부 터미널(`Vitis-AI /workspace>`)로 진입하게 됩니다.

---

### Step 6. Vitis AI Conda 환경 활성화 및 정상 동작 확인

컨테이너 내부에서 프레임워크 가상환경을 활성화합니다.

```bash
# 6-1. PyTorch 가상환경 활성화 (컨테이너 내부에서 실행)
conda activate vitis-ai-pytorch

# 6-2. Vitis AI 컴파일러(xicc) 정상 설치 여부 확인
vai_c_xir --help

```

`vai_c_xir` 도움말 화면이 출력되면 **`.xmodel` 변환용 Vitis AI 환경 구축이 완료**된 것입니다.

---

## 3. Model quantization evieonment setup

### ultralytics install in vitis-ai-pytorch conda env

ultralytics install 시 기존 환경을 그대로 유지한채 인스톨 진행

```bash
pip install ultralytics requests urllib3 charset_normalizer idna --no-deps
```