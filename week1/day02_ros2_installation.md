# Day 2 — Raspberry Pi에 ROS2 Humble 설치

> **목표:** RPi에 ROS2 Humble Hawksbill (ros-base)을 설치하고 정상 동작을 확인한다.

---

## 2.1 개요

ROS2 Humble Hawksbill은 Ubuntu 22.04 (Jammy)를 공식 지원하는 LTS 배포판입니다.
RPi의 리소스를 고려하여 **ros-base** (핵심 통신 라이브러리 + 빌드 도구)만 설치합니다.

> **ROS2 설치 방식 비교**  
> - **Debian 패키지 (권장):** `apt`으로 간단 설치, 관리 용이  
> - **소스 빌드:** 최신 기능, 그러나 빌드 시간 2시간+ (RPi에서는 비권장)

---

## 2.2 Locale 설정

ROS2는 UTF-8 locale을 필요로 합니다:

```bash
# 현재 locale 확인
locale

# UTF-8 locale이 없으면 설정
sudo apt update && sudo apt install -y locales
sudo locale-gen en_US en_US.UTF-8
sudo update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8
export LANG=en_US.UTF-8

# 확인
locale
```

---

## 2.3 ROS2 Repository 추가

```bash
# ROS2 GPG 키 추가
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key \
  -o /usr/share/keyrings/ros-archive-keyring.gpg

# Repository 추가
echo "deb [arch=$(dpkg --print-architecture) \
  signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] \
  http://packages.ros.org/ros2/ubuntu jammy main" | \
  sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null

# 패키지 목록 업데이트
sudo apt update
```

---

## 2.4 ROS2 Humble Base 설치

```bash
# ROS2 Humble ros-base + 필수 도구 설치
sudo apt install -y \
  ros-humble-ros-base \
  python3-argcomplete \
  python3-colcon-common-extensions \
  ros-dev-tools

# 의존성 업데이트
sudo apt update && sudo apt upgrade -y
sudo apt autoremove -y
```

> **설치 용량:** 약 300-400MB (ros-base 기준)
> **참고:** `ros-humble-desktop`은 1GB+로 RPi에는 무거우므로 설치하지 않습니다.

---

## 2.5 환경 변수 설정

```bash
# ROS2 환경을 bashrc에 등록 (매 로그인 시 자동 적용)
echo 'source /opt/ros/humble/setup.bash' >> ~/.bashrc
source ~/.bashrc
```

---

## 2.6 설치 확인

### 방법 1: 패키지 확인

```bash
# ROS2 환경 변수 확인
printenv | grep ROS

# 설치된 패키지 목록
ros2 pkg list | wc -l    # 패키지 수 확인
ros2 pkg list | head -20 # 일부 패키지 확인
```

### 방법 2: 토픽 확인 (통신 테스트)

```bash
# 터미널 1 (RPi SSH)
ros2 topic list
# 아직 아무 노드도 실행하지 않았으므로 빈 리스트 출력

# 토픽 echo 테스트를 위해 데모 노드 실행
ros2 run demo_nodes_cpp talker &
sleep 2
ros2 topic list
# /chatter, /parameter_events, /rosout 등이 보여야 함
```

### 방법 3: talker/listener 데모

```bash
# 터미널 1에서 listener 실행
ros2 run demo_nodes_py listener

# 터미널 2에서 talker 실행 (SSH 새 창)
ros2 run demo_nodes_py talker
# "Publishing: 'Hello World: N'" 메시지가 반복되어야 함
```

> **Ctrl+C**로 종료합니다.

---

## 2.7 Colcon 빌드 도구 확인

```bash
# Colcon 설치 확인
colcon --help

# 테스트 워크스페이스 생성
mkdir -p ~/test_ws/src
cd ~/test_ws
colcon build
# build/, install/, log/ 디렉토리가 생성되면 성공

# 불필요한 테스트 워크스페이스 삭제
cd ~
rm -rf ~/test_ws
```

---

## 2.8 RPi 메모리 확인 (8GB 최적화)

```bash
# 스왑 메모리 확인 (8GB 모델은 불필요하지만 확인)
free -h

# ZRAM 설정 확인
cat /etc/default/zramswap 2>/dev/null || echo "ZRAM not configured"
```

RPi 4B 8GB는 RAM이 충분하므로 별도의 스왑 설정이 필요 없습니다.

---

## 2.9 추가 팁: tmux 또는 screen 사용

SSH 환경에서 여러 ROS2 노드를 실행하려면 터미널 멀티플렉서가 유용합니다:

```bash
# tmux 설치
sudo apt install -y tmux

# tmux 기본 사용법
tmux                    # 세션 시작
Ctrl+B %                # 세로 분할
Ctrl+B "                # 가로 분할
Ctrl+B 방향키           # 창 이동
Ctrl+B d                # 세션 분리 (detach)
tmux attach             # 세션 재접속
```

---

## 📝 연습 문제

1. `ros2 run demo_nodes_cpp talker`와 `ros2 run demo_nodes_py listener`를 각각 실행하고, publisher/subscriber가 어떻게 통신하는지 설명하세요
2. `ros2 node list`와 `ros2 node info` 명령어로 talker 노드의 상세 정보를 확인하세요
3. `ros2 topic echo /chatter` 명령어로 발행되는 메시지 내용을 확인하세요
4. `ros2 topic pub /test std_msgs/String "data: 'Hello ROS2'"`를 실행하고, `ros2 topic echo /test`로 수신을 확인하세요
5. `ros2 topic bw /chatter`로 토픽의 대역폭을 확인하고, 결과를 기록하세요

---

## ⚠️ 트러블슈팅

| 문제 | 해결 방법 |
|------|----------|
| `ros2: command not found` | `source /opt/ros/humble/setup.bash` 실행 후 재시도 |
| apt key expired | `sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key -o /usr/share/keyrings/ros-archive-keyring.gpg` 재실행 |
| colcon build 실패 | `sudo apt install python3-colcon-common-extensions` 확인 |
| `Failed to communicate with DDS` | `ifconfig`로 네트워크 인터페이스 확인, loopback만 있으면 정상 |
