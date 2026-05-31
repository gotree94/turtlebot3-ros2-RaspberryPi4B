# Day 8 — TurtleBot3 SBC 패키지 설치 (RPi)

> **목표:** Raspberry Pi에 TurtleBot3 구동에 필요한 ROS2 패키지를 설치하고, OpenCR과 LDS를 설정한다.

---

## 8.1 개요

TurtleBot3 Burger는 다음과 같은 핵심 하드웨어로 구성됩니다:

| 부품 | 역할 | 통신 방식 |
|------|------|----------|
| **OpenCR 1.0** | 모터 제어 보드 (STM32F7) | USB (UART CDC) |
| **DYNAMIXEL XL430-W250** x2 | 바퀴 구동 모터 | Dynamixel Protocol 2.0 |
| **LDS-01 / LDS-02** | 360° 레이저 스캐너 | UART (USB) |
| **Raspberry Pi 4B** | 메인 컨트롤러 (SBC) | GPIO / USB |

---

## 8.2 필수 시스템 패키지 설치

```bash
# TurtleBot3 의존성 패키지
sudo apt install -y \
  python3-argcomplete \
  python3-colcon-common-extensions \
  libboost-system-dev \
  build-essential \
  libudev-dev
```

---

## 8.3 TurtleBot3 ROS2 패키지 설치

### 방법 A: Debian 바이너리 설치 (권장, 간편)

```bash
# ROS2 Humble TurtleBot3 패키지
sudo apt install -y \
  ros-humble-hls-lfcd-lds-driver \
  ros-humble-turtlebot3-msgs \
  ros-humble-dynamixel-sdk \
  ros-humble-xacro

# 위 패키지로도 부족하면 turtlebot3 메타패키지 설치
sudo apt install -y ros-humble-turtlebot3
```

### 방법 B: 소스 빌드 (최신 버전 필요 시)

RPi 4B 8GB에서 빌드 시간: **약 40-60분** (유선 전원 필수!)

```bash
# 워크스페이스 생성
mkdir -p ~/turtlebot3_ws/src
cd ~/turtlebot3_ws/src

# 저장소 클론 (humble 브랜치)
git clone -b humble https://github.com/ROBOTIS-GIT/turtlebot3.git
git clone -b humble https://github.com/ROBOTIS-GIT/ld08_driver.git
git clone -b humble https://github.com/ROBOTIS-GIT/hls_lfcd_lds_driver.git

# 불필요한 Navigation 패키지 제거 (빌드 시간 단축)
cd ~/turtlebot3_ws/src/turtlebot3
rm -rf turtlebot3_cartographer turtlebot3_navigation2

# 의존성 확인
cd ~/turtlebot3_ws
source /opt/ros/humble/setup.bash
rosdep install --from-paths src --ignore-src -r -y
```

> ⚠️ **rosdep이 없으면:** `sudo apt install -y python3-rosdep && sudo rosdep init && rosdep update`

```bash
# 빌드 (RPi에서는 --parallel-workers 1 권장)
cd ~/turtlebot3_ws
colcon build --symlink-install --parallel-workers 1

# 환경 설정
echo 'source ~/turtlebot3_ws/install/setup.bash' >> ~/.bashrc
source ~/.bashrc
```

---

## 8.4 OpenCR udev 규칙 설정

OpenCR 보드가 USB로 연결될 때 올바른 권한으로 인식되도록 설정:

```bash
# turtlebot3_bringup 패키지에서 udev 규칙 복사
# 방법 1: apt 설치 시
sudo cp $(ros2 pkg prefix turtlebot3_bringup)/share/turtlebot3_bringup/script/99-turtlebot3-cdc.rules /etc/udev/rules.d/

# 방법 2: 소스 빌드 시
# 패키지 경로 직접 확인
find / -name "99-turtlebot3-cdc.rules" 2>/dev/null

# 또는 수동 작성
sudo nano /etc/udev/rules.d/99-turtlebot3-cdc.rules
```

다음 내용 입력:

```rules
# OpenCR - TurtleBot3 CDC
SUBSYSTEM=="tty", ATTRS{idVendor}=="0483", ATTRS{idProduct}=="5740", MODE="0666", GROUP="dialout"
SUBSYSTEM=="tty", ATTRS{idVendor}=="0483", ATTRS{idProduct}=="df11", MODE="0666", GROUP="dialout"
```

규칙 적용:

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger

# 사용자를 dialout 그룹에 추가
sudo usermod -aG dialout $USER

# 재부팅 또는 로그아웃 후 재로그인 필요
sudo reboot
```

---

## 8.5 OpenCR 펌웨어 확인 & 업데이트

### OpenCR 연결 확인

```bash
# 재부팅 후 OpenCR USB 포트 확인
ls -la /dev/ttyACM*

# 권한 확인
ls -la /dev/ttyACM0
# crw-rw-rw- 권한이면 정상
```

### 펌웨어 업데이트 (필요한 경우에만)

현재 TurtleBot3 Burger는 OpenCR에 **ROS2 Humble용 펌웨어**가 필요합니다.

```bash
# OpenCR 업데이트 도구 설치
sudo apt install -y ros-humble-turtlebot3*

# 펌웨어 업로드 (OpenCR이 USB로 연결된 상태)
export OPENCR_PORT=/dev/ttyACM0
export OPENCR_MODEL=burger
rm -rf ~/opencr_update
mkdir -p ~/opencr_update && cd ~/opencr_update
wget https://github.com/ROBOTIS-GIT/OpenCR-Binaries/raw/master/turtlebot3/ROS2/latest/opencr_firmware.tar.gz
tar xvf opencr_firmware.tar.gz
cd ~/opencr_update/opencr_firmware
./update.sh $OPENCR_PORT $OPENCR_MODEL
```

> **OpenCR 펌웨어 업데이트 시 주의:**
> - OpenCR의 펌웨어 업데이트 버튼(push button)을 누른 상태에서 USB 연결
> - 업데이트 중 전원이 끊기지 않도록 주의

---

## 8.6 환경 변수 설정

```bash
# TurtleBot3 관련 환경 변수를 bashrc에 추가
cat >> ~/.bashrc << 'EOF'

# TurtleBot3 Settings
export TURTLEBOT3_MODEL=burger
export ROS_DOMAIN_ID=30
export LDS_MODEL=LDS-01        # LDS-02 사용 시 LDS-02로 변경
export OPENCR_PORT=/dev/ttyACM0
EOF

source ~/.bashrc

# 환경 변수 확인
echo $TURTLEBOT3_MODEL
echo $ROS_DOMAIN_ID
echo $LDS_MODEL
```

> **LDS 모델 확인 방법:** 
> - TurtleBot3 구매 시기가 2022년 이전 → LDS-01
> - 2022년 이후 → LDS-02 (레이저 센서 하우징이 더 작고 둥글게 생김)
> - 잘 모르겠으면 LDS 하단의 라벨 확인

---

## 8.7 시스템 최적화 (RPi)

### 자동 업데이트 비활성화

```bash
# ROS2 통신 중 업데이트로 인한 재부팅 방지
sudo nano /etc/apt/apt.conf.d/20auto-upgrades
```

다음으로 변경:

```conf
APT::Periodic::Update-Package-Lists "0";
APT::Periodic::Unattended-Upgrade "0";
```

### 로그 제한

```bash
# ROS2 로그 크기 제한 (RPi SD 카드 용량 보호)
echo 'export RCUTILS_CONSOLE_OUTPUT_FORMAT="[{severity}] [{time}] [{name}]: {message}"' >> ~/.bashrc
echo 'export RCUTILS_LOGGING_USE_STDOUT=1' >> ~/.bashrc
source ~/.bashrc
```

---

## 8.8 설치 확인

```bash
# 패키지 설치 확인
ros2 pkg list | grep turtlebot3

# 예상 출력:
# turtlebot3
# turtlebot3_bringup
# turtlebot3_description
# turtlebot3_example
# turtlebot3_msgs
# turtlebot3_node
# turtlebot3_teleop

# OpenCR 연결 확인 (OpenCR이 연결된 상태에서)
ls -la /dev/ttyACM0

# udev 규칙 적용 확인
udevadm info -a -n /dev/ttyACM0 | grep -i "idVendor\|idProduct"
```

---

## 📝 연습 문제

1. `ros2 pkg list | grep turtlebot3`로 설치된 패키지 목록을 확인하고, 각 패키지의 역할을 조사하세요
2. `lsusb` 명령어로 OpenCR과 LDS가 USB 장치로 인식되는지 확인하세요
3. OpenCR 포트의 권한(`ls -la /dev/ttyACM*`)을 확인하고, 권한이 없으면 dialout 그룹에 사용자가 추가되었는지 확인하세요
4. `udevadm info -a -n /dev/ttyACM0`로 OpenCR의 상세 정보를 출력하세요
5. RPi의 CPU 온도(`cat /sys/class/thermal/thermal_zone0/temp`)를 확인하고, 빌드 전후 온도 차이를 기록하세요

---

## ⚠️ 트러블슈팅

| 문제 | 해결 방법 |
|------|----------|
| OpenCR 권한 오류 (`Permission denied: /dev/ttyACM0`) | `sudo usermod -aG dialout $USER` 후 재로그인 |
| `colcon build` 중 메모리 부족 | `--parallel-workers 1` 옵션 추가 확인 |
| `rosdep install` 실패 | `sudo apt install -y python3-rosdep && sudo rosdep init && rosdep update` |
| udev 규칙 적용 안 됨 | `sudo udevadm control --reload-rules && sudo udevadm trigger` |
| OpenCR 펌웨어 업데이트 실패 | OpenCR의 Bootloader 버튼 누르고 USB 연결 후 재시도 |
