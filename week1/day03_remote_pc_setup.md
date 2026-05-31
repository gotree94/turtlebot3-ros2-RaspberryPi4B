# Day 3 — Remote PC 세팅

> **목표:** Remote PC (노트북/데스크탑)에 Ubuntu 22.04 + ROS2 Humble Desktop + Gazebo를 설치하고, RPi와 통신을 확인한다.

---

## 3.1 개요

Remote PC는 TurtleBot3를 원격으로 제어하고 데이터를 시각화하는 메인 워크스테이션입니다.
시뮬레이션, RViz2, RQt 등 GUI 도구가 필요하므로 **Desktop 버전**의 ROS2를 설치합니다.

---

## 3.2 Ubuntu 22.04 Desktop 설치

> **이미 Ubuntu 22.04가 설치되어 있으면 건너뜁니다.**

1. [Ubuntu 22.04 LTS Desktop 다운로드](https://releases.ubuntu.com/jammy/)
2. 부팅 USB로 설치 (Rufus, balenaEtcher 등 사용)
3. 설치 완료 후 시스템 업데이트

```bash
sudo apt update && sudo apt upgrade -y && sudo apt autoremove -y
```

---

## 3.3 ROS2 Humble Desktop 설치

### Step 1: Repository 추가

```bash
# Locale 설정
sudo apt update && sudo apt install -y locales
sudo locale-gen en_US en_US.UTF-8
sudo update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8
export LANG=en_US.UTF-8

# ROS2 GPG 키
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key \
  -o /usr/share/keyrings/ros-archive-keyring.gpg

# Repository 추가
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu jammy main" | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null

sudo apt update
```

### Step 2: Desktop 패키지 설치

```bash
# Desktop Full (GUI 도구, 라이브러리, 시뮬레이션 포함)
sudo apt install -y ros-humble-desktop

# 추가 개발 도구
sudo apt install -y \
  python3-colcon-common-extensions \
  python3-argcomplete \
  python3-rosdep \
  ros-dev-tools \
  python3-vcstool
```

### Step 3: 환경 설정

```bash
echo 'source /opt/ros/humble/setup.bash' >> ~/.bashrc
source ~/.bashrc
```

---

## 3.4 Gazebo Classic 설치

```bash
# Gazebo와 TurtleBot3 시뮬레이션 의존성
sudo apt install -y \
  ros-humble-gazebo-* \
  ros-humble-cartographer \
  ros-humble-cartographer-ros \
  ros-humble-slam-toolbox

# Gazebo 환경 설정
echo 'source /usr/share/gazebo/setup.sh' >> ~/.bashrc
source ~/.bashrc
```

---

## 3.5 TurtleBot3 시뮬레이션 패키지 설치 (PC)

```bash
# Debian 패키지로 설치 (간편)
sudo apt install -y ros-humble-turtlebot3*

# 또는 소스 빌드 (최신 버전 필요 시)
mkdir -p ~/tb3_sim_ws/src
cd ~/tb3_sim_ws/src
git clone -b humble https://github.com/ROBOTIS-GIT/turtlebot3_simulations.git
cd ~/tb3_sim_ws
colcon build --symlink-install

# 환경 설정
echo 'source ~/tb3_sim_ws/install/setup.bash' >> ~/.bashrc
source ~/.bashrc
```

### Gazebo 시뮬레이션 테스트

```bash
export TURTLEBOT3_MODEL=burger
ros2 launch turtlebot3_gazebo empty_world.launch.py
```

> Gazebo 창이 뜨고 빈 공간에 TurtleBot3가 나타나면 성공!
> **Ctrl+C**로 종료합니다.

---

## 3.6 ROS_DOMAIN_ID 설정

ROS2는 DDS로 통신하며, 동일 네트워크에서 여러 ROS2 시스템을 격리하기 위해 `ROS_DOMAIN_ID`를 사용합니다.

```bash
# RPi와 동일한 ID로 설정 (기본값 30 권장)
echo 'export ROS_DOMAIN_ID=30 #TURTLEBOT3' >> ~/.bashrc
source ~/.bashrc
```

> **ROS_DOMAIN_ID 규칙:** RPi와 Remote PC의 ID가 같아야 통신 가능 (0~232, 기본값 30 권장)

---

## 3.7 RPi ↔ PC 통신 확인

### Step 1: 양쪽에서 ROS_DOMAIN_ID 확인

```bash
# RPi와 PC 각각에서 실행
echo $ROS_DOMAIN_ID
# 30이 출력되어야 함
```

### Step 2: RPi에서 talker 실행

```bash
# RPi SSH 터미널
ros2 run demo_nodes_py talker
```

### Step 3: PC에서 listener 실행

```bash
# PC 터미널
ros2 run demo_nodes_py listener
```

### Step 4: PC에서 토픽 확인

```bash
# PC에서 RPi의 토픽이 보이는지 확인
ros2 topic list
# /chatter가 보여야 함

ros2 topic echo /chatter
# "Hello World: N" 메시지가 PC에 표시되어야 함
```

> 🎉 **메시지가 보인다면 RPi와 PC 간 ROS2 통신 성공!**

---

## 3.8 네트워크 최적화

### 방화벽 확인

```bash
# Ubuntu 방화벽 (기본적으로 DDS는 UDP 멀티캐스트 사용)
sudo ufw status

# 필요 시 DDS 포트 허용 (ROS_DOMAIN_ID=30 기준)
# UDP 7400-7500, 11311 등
```

### WiFi 성능 테스트

```bash
# 네트워크 지연 시간 확인
ping -c 10 turtlebot-pi.local
# 1ms 이내면 이상적, 5ms 이상이면 WiFi 신호 개선 필요
```

---

## 📝 연습 문제

1. Gazebo의 `empty_world.launch.py` 대신 `turtlebot3_world.launch.py`를 실행하고 차이점을 확인하세요
2. PC에서 `ros2 run turtlebot3_teleop teleop_keyboard`를 실행하고 Gazebo 속 TurtleBot3를 움직여보세요
3. `rqt_graph`를 실행하여 Gazebo 속 노드들의 통신 구조를 확인하세요
4. RPi의 talker와 PC의 listener가 통신하는 동안 PC에서 `rqt_console`을 실행하여 로그를 확인하세요
5. `ROS_DOMAIN_ID`를 RPi와 PC에서 다르게 설정하면 통신이 안 되는 것을 확인해보세요

---

## ⚠️ 트러블슈팅

| 문제 | 해결 방법 |
|------|----------|
| Gazebo가 검은 화면만 표시 | `export LIBGL_ALWAYS_SOFTWARE=1` 실행 후 재시도 |
| PC에서 RPi 토픽 안 보임 | 양쪽 `ROS_DOMAIN_ID` 동일 확인, `ping` 테스트 |
| DDS 통신 간헐적 끊김 | WiFi 5GHz 사용, `sudo iwconfig wlan0 power off` |
| Gazebo 실행 중 크래시 | `killall gzserver; killall gzclient`로 프로세스 정리 후 재시작 |
