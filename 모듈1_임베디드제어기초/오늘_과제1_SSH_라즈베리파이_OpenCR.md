# 오늘 실습: SSH로 세팅하고 과제 1 실행

대상 환경은 **라즈베리파이의 Ubuntu Server 22.04**입니다. PC는 SSH 접속에만 사용합니다. **OpenCR USB 케이블은 라즈베리파이에 연결하고, 업로드·시리얼 수신·로그 저장을 모두 라즈베리파이에서 수행합니다.** GUI, VNC, PC Arduino IDE, PC 시리얼 모니터는 사용하지 않습니다.

오늘은 통합 과제 중 **문제 1만** 진행합니다. Kp 비교와 micro-ROS 연동은 오늘 범위가 아닙니다. 세팅 40~60분, 과제 1 실행·정리 20~30분을 운영 예시로 잡고 네트워크·최초 설치 상태에 따라 조절합니다.

## 0. 장비와 오늘의 방식

PC의 SSH 터미널 → 라즈베리파이 Ubuntu Server → USB → OpenCR → 다이나믹셀의 연결입니다. 모터 전원은 해당 모델의 정격에 맞게 준비합니다.

OpenCR 공식 보드 매니저는 ARM SBC의 Arduino IDE를 지원하지 않는다고 안내합니다. 따라서 오늘은 **수업용으로 미리 빌드한 펌웨어 파일을 라즈베리파이에서 업로드**합니다. 업로더는 라즈베리파이에서 소스 빌드하여 CPU 아키텍처를 맞춥니다. 펌웨어 자체를 라즈베리파이에서 컴파일하는 것은 오늘 실습에 포함하지 않습니다.

제공 바이너리는 **XM430-W350(모델 1020), ID 12, 1 Mbps, Protocol 2.0** 설정입니다. 다르면 진행자가 호환 펌웨어를 준비해야 합니다. 검사를 우회하거나 다른 모터에 실행하지 않습니다.

진행자 사전 점검: SSH 계정·IP, 네트워크, 모터 모델·ID·통신속도, 전원·고정 상태, USB 데이터 케이블, 전원 차단 위치, 움직임 범위를 확인합니다. 실기 업로드와 구동은 진행자가 먼저 검증해야 합니다.

## 1. PC에서 SSH 접속

다음 한 명령만 PC에서 실행합니다. `사용자명`과 `라즈베리파이IP`를 배포받은 값으로 바꾸세요. 최초 접속의 호스트 키는 진행자가 알려 준 값과 대조합니다.

```bash
ssh 사용자명@라즈베리파이IP
```

**이후 모든 명령은 SSH로 접속한 라즈베리파이 터미널에서 실행합니다.**

```bash
hostname
cat /etc/os-release
uname -m
```

Ubuntu 22.04인지 확인합니다. `aarch64`이면 64비트 ARM입니다. 이 문서는 업로더를 현재 머신에서 빌드하므로 x86 실행 파일을 가져와 실행하지 않습니다.

## 2. 도구 설치와 시리얼 권한

```bash
sudo apt update
sudo apt install -y git build-essential python3-serial usbutils file
sudo usermod -aG dialout "$USER"
exit
```

PC에서 SSH로 다시 접속한 뒤 그룹 적용을 확인합니다.

```bash
id -nG
```

출력에 `dialout`이 있어야 합니다. 시리얼 프로그램을 무조건 sudo로 실행하여 권한 문제를 숨기지 않습니다.

## 3. 과제 레포와 펌웨어 확인

홈 폴더에 같은 이름의 폴더가 이미 있으면 기존 내용을 지우지 말고 진행자에게 확인하세요.

```bash
cd ~
git clone https://github.com/SpartaPA/physicalai-lv2-assignments.git
cd ~/physicalai-lv2-assignments/모듈1_임베디드제어기초/firmware
sha256sum -c SHA256SUMS
```

`opencr_position_p.ino.bin: OK`인지 확인합니다. 이것은 파일 무결성 확인이며 실기 동작 검증은 아닙니다.

## 4. 라즈베리파이용 OpenCR 업로더 빌드

ROBOTIS의 업로더 소스를 고정된 커밋으로 가져옵니다. 다음 블록은 처음 한 번만 실행합니다. 기존 `~/opencr-uploader-src`가 있으면 덮어쓰지 말고 진행자에게 확인하세요.

```bash
mkdir ~/opencr-uploader-src
cd ~/opencr-uploader-src
git init
git remote add origin https://github.com/ROBOTIS-GIT/OpenCR.git
git sparse-checkout init --cone
git sparse-checkout set arduino/opencr_develop/opencr_ld
git fetch --depth 1 --filter=blob:none origin 68ec75d8a400949580ecf263e0105ea9743b878e
git checkout --detach FETCH_HEAD
cd arduino/opencr_develop/opencr_ld
make
file opencr_ld
```

`aarch64` OS라면 생성 파일도 ARM aarch64 실행 파일이어야 합니다. `Exec format error`는 실행 파일과 CPU가 맞지 않는 상황을 의심합니다.

## 5. OpenCR 포트 확인

OpenCR을 **라즈베리파이의 USB**에 연결합니다. USB 포트 확인 단계에서는 움직임을 시작하지 않습니다.

```bash
lsusb
ls -l /dev/ttyACM*
ls -l /dev/serial/by-id/
```

OpenCR만 연결한 상태에서 확인한 포트를 사용합니다. 아래는 `/dev/ttyACM0`인 경우입니다. 장비가 여러 개면 USB를 연결하기 전후 목록과 장치 정보를 비교합니다.

```bash
PORT=/dev/ttyACM0
udevadm info --query=property --name="$PORT"
test -r "$PORT" && test -w "$PORT" && echo '포트 읽기/쓰기 가능'
```

재연결·업로드 뒤 번호가 바뀔 수 있습니다. 그때 목록을 다시 확인하고 PORT를 갱신합니다. 모뎀 관리 서비스가 포트를 점유하는 수업 장비에서는 진행자가 필요 여부를 확인한 후 일시 정지할 수 있습니다.

```bash
if systemctl is-active --quiet ModemManager; then
  sudo systemctl stop ModemManager
fi
```

## 6. 라즈베리파이에서 펌웨어 업로드

열려 있는 시리얼 모니터를 종료합니다. 모터를 고정하고 이동 범위를 비운 상태로 진행합니다. 이 업로드는 OpenCR의 기존 응용 펌웨어를 교체합니다.

```bash
cd ~/physicalai-lv2-assignments/모듈1_임베디드제어기초/firmware
mkdir -p ~/lv2_module1_results
set -o pipefail
~/opencr-uploader-src/arduino/opencr_develop/opencr_ld/opencr_ld \
  "$PORT" 115200 "$PWD/opencr_position_p.ino.bin" 1 \
  2>&1 | tee ~/lv2_module1_results/upload.log
```

출력에서 **`CRC OK`와 `[OK] Download`**를 확인합니다. 업로더가 오류여도 종료 코드만으로 성공을 판단하면 안 됩니다. 뒤 단계에서 READY 및 모터 설정 확인까지 이어가세요.

실패하면 포트·권한·포트 점유·USB 케이블부터 확인합니다. 진행자 안내로 복구 모드가 필요할 때는 PUSH SW2를 누른 채 RESET을 눌렀다 놓고, SW2를 놓습니다. 포트를 다시 확인해 같은 업로드 명령을 실행합니다. DFU 부트로더 덮어쓰기를 학생이 임의로 진행하지 않습니다.

## 7. SSH 시리얼 모니터와 로그 저장

업로드 뒤 포트 목록과 PORT를 다시 확인합니다. 모터 모델에 맞는 전원을 준비하고 실행 범위를 비워 둡니다.

```bash
ls -l /dev/ttyACM*
PORT=/dev/ttyACM0
script -q -f -c "python3 -m serial.tools.miniterm $PORT 115200 --eol LF" \
  "$HOME/lv2_module1_results/실행A_터미널.log"
```

이 명령은 **라즈베리파이에서** 시리얼을 받고 **라즈베리파이 파일**에 기록합니다. 로그에 터미널 제어 문자나 안내 문구가 섞일 수 있으므로 원문을 보존하고 측정 행을 구분합니다.

연결 후 READY를 확인합니다. 초기 문구를 놓쳤다면 정지·안전 상태에서 진행자와 RESET을 눌러 초기 출력을 확인합니다. 재열거로 연결이 끊기면 포트를 다시 확인하고 모니터를 재실행합니다. FAULT가 있으면 원인을 확인하고, 정상 상태로 복구되기 전에는 움직임 명령을 보내지 않습니다.

## 8. 과제 1 실행 A

다음은 **셸 명령이 아니라 시리얼 모니터 안에 입력할 명령**입니다. 진행자가 장비에서 확인한 설정을 사용합니다.

```text
s 0.3 15 30
```

의미는 Kp 0.3, 속도 상한 15°/s, 시작 위치 기준 상대 목표 +30°입니다. 모든 장비의 안전을 보장하는 값은 아닙니다. 예제는 시작 위치를 0°로 잡고 약 2초 뒤 목표를 바꿉니다. 전체 5~10초간 관찰하되 이상 움직임이면 즉시 정지합니다. 목표 변경 이후의 로그를 5행 이상 남깁니다.

시리얼 모니터 안에서 아래 명령과 Enter로 정지합니다.

```text
x
```

정지 확인 후 **Ctrl+]**로 시리얼 모니터를 종료합니다. 모니터 종료나 SSH 종료를 모터 정지 명령 대신 사용하지 않습니다. x에 응답하지 않으면 진행자가 안내한 전원 차단 절차를 사용합니다.

## 9. 라즈베리파이에서 기록 확인

```bash
tail -n 30 ~/lv2_module1_results/실행A_터미널.log
grep -a 'target_deg:' ~/lv2_module1_results/실행A_터미널.log | tail -n 10
```

| 로그 항목 | 뜻 | 단위 |
|---|---|---|
| target_deg | 목표각 | ° |
| position_deg | 현재각 | ° |
| error_deg | 목표각−현재각 | ° |
| u_deg_s | OpenCR이 보낸 속도 명령 | °/s |
| speed_deg_s | 실제 측정 속도 | °/s |
| t_s | 실행 시작 이후 시간 | s |

목표에 완전히 도달하지 않아도 실제 기록을 그대로 사용합니다. 오늘은 Kp를 더 튜닝하지 않습니다.

## 10. 오늘 제출할 내용

1. 라즈베리파이 `hostname`, Ubuntu 버전, 아키텍처, 사용 포트 확인 기록.
2. 라즈베리파이의 `upload.log`에서 CRC·다운로드 성공 기록.
3. `실행A_터미널.log`와 모터 모델·ID·Kp·목표각·속도 상한·예제 이름.
4. 목표값·측정값·제어 출력의 이름과 단위, 실제 움직임 설명 2문장.

계속 SSH만 사용하려면 `cat`·`tail`로 내용을 확인하고 필요한 기록을 SSH 화면에서 복사해 답안에 넣을 수 있습니다. PC 시리얼 모니터나 GUI 원격 데스크톱으로 대체하지 않습니다. 문제 2~4와 실행 B는 이후에 진행합니다.

과제 1은 설정·실행 10점, 목표·측정·출력 구분 10점, 관찰 설명 5점입니다. 업로드·환경 기록은 오늘 실습 수행을 확인하는 보충 증거입니다.

## 문제 해결

| 상황 | 확인할 것 |
|---|---|
| ttyACM 포트가 없음 | USB 데이터 케이블, Pi 연결 여부, `lsusb`, 재연결 후 포트 목록 |
| Permission denied | dialout 추가 후 SSH 재접속 여부, 포트 소유 그룹 |
| 업로드 중 timeout | 시리얼 모니터 종료, 정확한 포트, 케이블, 복구 모드 |
| READY 대신 FAULT | 모터 모델·ID·통신속도·전원·통신 케이블을 진행자가 점검 |
| 입력이 화면에 안 보임 | miniterm은 로컬 에코가 기본 꺼짐. Enter 전송 후 응답 확인. Ctrl+T 다음 Ctrl+E로 에코 전환 가능 |
| 로그가 전혀 안 나옴 | 시작 명령·새줄·READY 확인. 예제는 실행 중 주기적으로 기록 |
| SSH 끊김 | 자동 정지를 가정하지 않기. 주변 진행자가 장비 정지·전원 절차 수행 |

## 검증 범위와 근거

제공 펌웨어의 컴파일과 업로더의 Linux x86_64 소스 빌드는 확인했습니다. 라즈베리파이 ARM에서의 업로드와 모터 실동작은 아직 수행하지 않았습니다. 진행자는 동일 장비에서 사전 검증한 뒤 학생에게 배포하세요.

- [OpenCR 공식 설치 및 ARM SBC 제한](https://emanual.robotis.com/docs/en/platform/turtlebot3/opencr_setup/)
- [OpenCR 공식 업로더 소스](https://github.com/ROBOTIS-GIT/OpenCR/tree/68ec75d8a400949580ecf263e0105ea9743b878e/arduino/opencr_develop/opencr_ld)
- [펌웨어 빌드 정보와 체크섬](firmware/README.md)
