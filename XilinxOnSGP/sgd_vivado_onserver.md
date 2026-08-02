# Working on Sogang Sever

vitis 2022.04 version을 install 하여 놓았다. 이를 기반으로 환경을 구축해 보자

## setup vivado & vitis environment

```plaintxt
****   전체 구성   ****
[ 학교 서버 (Linux) ]                        [ 내 로컬 PC (Unbuntu/Mac/Win) ]       [ Kria KV260 보드 ]
 Vivado / Vitis        --- (SSH 터널링) --->   hw_server (Vivado Lab)   --- (USB-JTAG) --->   KV260 Board
 (비트스트림 빌드/다운)                           (포트 3121 대기)
       |
       +-------------- (X11 forwarding) ---> Design on GUI env.
```

### 1. 각자의 서버 login 정보를 기반으로 X11 forwarding 과 SSH 터널링( Reverse Tunneling ) 연결

```bash
ssh -Y -R 3121:localhost:3121 eduxxx@1921.168.1.<server IP>
```

집에서 접속시

* Mac / Ubunut

```bash
ssh -Y -R 3121:localhost:3121 eduxxx@<sogang pankyo serverIP> -p <port#>
```

* WSL2 에서 접속: windows network 보안 문제로 사용이 어렵다.

```bash
# WSL2에서 Windows 본체 IP를 자동으로 추출하여 SSH 역방향 터널링 실행
ssh -Y -R 3121:$(host_ip=$(ip route show | grep default | awk '{print $3}'); echo $host_ip):3121 <user id>@<학교서버IP> -p <서버포트>
```

### 2. 점속 후 vivado 환경 setup

```bash
source /DATA/home/edu000/Xilinx/Vivado/2022.2/settings64.sh
```

`~/.bashrc`에 추가 한다. --> 추후 login시 환경 자동 setup

### 3. Vivado / Vitis board 에서 local board 에 연결시 Local 환경 setup

Xilinx 에서 제공하는 `hw_server` tool 사용: KV260 SOM은 boot_jtag setup 하여야 함.

```bash
hw_server
```

### **주의** 

board를 JTAG mode로 사용할 때는 SD를 제거하고 사용하자
