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
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key -o /usr/share/keyrings/ros-archive-keyring.gpg

# Repository 추가
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu jammy main" | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null

# 패키지 목록 업데이트
sudo apt update
```

---

## 2.4 ROS2 Humble Base 설치

```bash
# ROS2 Humble ros-base + 필수 도구 설치
sudo apt install -y ros-humble-ros-base python3-argcomplete python3-colcon-common-extensions ros-dev-tools

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


---


# Day 2 — Raspberry Pi에 ROS2 Humble 설치

## 목표

RPi에 ROS2 Humble Hawksbill (ros-base)을 설치하고 정상 동작을 확인한다.

---

# 2.1 개요

ROS2 Humble Hawksbill은 Ubuntu 22.04 (Jammy)를 공식 지원하는 LTS 배포판입니다.

RPi의 리소스를 고려하여 `ros-base` (핵심 통신 라이브러리 + 빌드 도구)만 설치합니다.

## ROS2 설치 방식 비교

| 방식 | 설명 |
|--------|--------|
| Debian 패키지 (권장) | apt으로 간단 설치, 관리 용이 |
| 소스 빌드 | 최신 기능 사용 가능, 빌드 시간 2시간 이상 (RPi에서는 비권장) |

---

# 2.2 ARM64 확인

ROS2 Humble은 ARM64(aarch64) 환경을 권장합니다.

현재 OS 아키텍처 확인:

```bash
uname -m
```

정상 결과:

```text
aarch64
```

만약 아래와 같이 나오면:

```text
armv7l
```

32비트 OS이므로 Ubuntu 22.04 64bit 재설치를 권장합니다.

---

# 2.3 시스템 업데이트

ROS 설치 전에 시스템을 최신 상태로 업데이트합니다.

```bash
sudo apt update
sudo apt full-upgrade -y
sudo reboot
```

---

# 2.4 Locale 설정

ROS2는 UTF-8 Locale을 필요로 합니다.

현재 Locale 확인:

```bash
locale
```

정상 예:

```text
LANG=en_US.UTF-8
```

또는

```text
LANG=C.UTF-8
```

UTF-8이 설정되지 않은 경우:

```bash
sudo apt update
sudo apt install -y locales

sudo locale-gen en_US en_US.UTF-8
sudo update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8

export LANG=en_US.UTF-8
```

확인:

```bash
locale
```

---

# 2.5 ROS2 Repository 추가

필수 패키지 설치:

```bash
sudo apt install -y \
software-properties-common \
curl \
gnupg \
lsb-release
```

ROS2 GPG Key 추가:

```bash
sudo curl -sSL \
https://raw.githubusercontent.com/ros/rosdistro/master/ros.key \
-o /usr/share/keyrings/ros-archive-keyring.gpg
```

Repository 등록:

```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu jammy main" | \
sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null
```

패키지 목록 갱신:

```bash
sudo apt update
```

---

# 2.6 ROS2 Humble Base 설치

ROS2 Humble Base와 필수 개발 도구 설치:

```bash
sudo apt install -y \
ros-humble-ros-base \
python3-colcon-common-extensions \
python3-argcomplete \
ros-dev-tools
```

설치 후 시스템 정리:

```bash
sudo apt update
sudo apt upgrade -y
sudo apt autoremove -y
```

설치 용량:

- 약 300~400MB

참고:

- `ros-humble-desktop`은 1GB 이상
- Raspberry Pi에서는 `ros-base` 권장

---

# 2.7 ROS2 환경 변수 등록

매 로그인 시 자동 적용되도록 설정합니다.

```bash
echo "source /opt/ros/humble/setup.bash" >> ~/.bashrc
```

적용:

```bash
source ~/.bashrc
```

---

# 2.8 DDS 설정 (CycloneDDS 권장)

Raspberry Pi 여러 대를 사용할 경우 CycloneDDS가 안정적입니다.

설치:

```bash
sudo apt install -y \
ros-humble-rmw-cyclonedds-cpp
```

환경 변수 등록:

```bash
echo "export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp" >> ~/.bashrc
```

적용:

```bash
source ~/.bashrc
```

확인:

```bash
echo $RMW_IMPLEMENTATION
```

정상 결과:

```text
rmw_cyclonedds_cpp
```

---

# 2.9 설치 확인

ROS 버전 확인:

```bash
ros2 --version
```

환경 확인:

```bash
printenv | grep ROS
```

진단:

```bash
ros2 doctor
```

정상 결과:

```text
All checks passed
```

---

# 2.10 ROS2 통신 테스트

## 터미널 1

```bash
source /opt/ros/humble/setup.bash

ros2 run demo_nodes_cpp talker
```

## 터미널 2

```bash
source /opt/ros/humble/setup.bash

ros2 run demo_nodes_cpp listener
```

정상 결과:

```text
I heard: Hello World
```

메시지가 계속 출력되면 ROS2 설치 성공.

---

# 2.11 Raspberry Pi 성능 최적화

메모리 확인:

```bash
free -h
```

GPU 메모리 조정:

```bash
sudo nano /boot/firmware/config.txt
```

추가 또는 수정:

```text
gpu_mem=128
```

저장 후 재부팅:

```bash
sudo reboot
```

---

# 완료 기준

다음 항목이 모두 성공하면 Day 2 완료.

- [ ] Ubuntu 22.04 64bit 설치 확인
- [ ] ARM64(aarch64) 확인
- [ ] ROS2 Humble 설치 완료
- [ ] bashrc 환경 등록 완료
- [ ] CycloneDDS 설정 완료
- [ ] ros2 doctor 정상 통과
- [ ] Talker / Listener 통신 성공

---

# 다음 단계 (Day 3)

- ROS2 Topic
- ROS2 Service
- ROS2 Action
- ROS2 Parameter
- ROS2 Launch File
- ROS2 Workspace 생성
- Colcon 빌드 실습

---

# Day 2 — Raspberry Pi에 ROS2 Humble 설치 (최종본)

**목표:** Raspberry Pi에 ROS2 Humble Hawksbill을 설치하고 talker/listener 통신을 확인한다.

---

## 사전 확인 사항

- Ubuntu 22.04 Server (64bit) 이미징 완료
- SSH 접속 가능 상태
- 인터넷 연결 확인

---

## Step 1 — 아키텍처 확인

```bash
uname -m
```

**정상 결과:**
```
aarch64
```

> ⚠️ `armv7l` 이 나오면 32bit OS → Ubuntu 22.04 64bit로 재이미징 필요

---

## Step 2 — 시스템 업데이트

> ⚠️ **핵심 주의사항**
> - `full-upgrade` 절대 사용 금지 → 커널 교체로 부팅 불가 발생
> - `upgrade` 도 대용량(500MB+) 쓰기로 저속 SD카드에서 Read-only 오류 발생
> - **`apt update` 만 실행하고 upgrade 없이 바로 ROS2 설치로 진행**

```bash
sudo apt update
```

---

## Step 3 — Locale 확인

```bash
locale
```

**정상 결과 (둘 중 하나면 OK):**
```
LANG=C.UTF-8
```
또는
```
LANG=en_US.UTF-8
```

> `C.UTF-8` 은 ROS2와 완전 호환되므로 추가 설정 불필요

---

## Step 4 — 필수 패키지 설치

```bash
sudo apt install -y \
  software-properties-common \
  curl \
  gnupg \
  lsb-release
```

---

## Step 5 — ROS2 Repository 추가

**GPG Key 추가:**
```bash
sudo curl -sSL \
  https://raw.githubusercontent.com/ros/rosdistro/master/ros.key \
  -o /usr/share/keyrings/ros-archive-keyring.gpg
```

**Repository 등록:**
```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu jammy main" | \
  sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null
```

**패키지 목록 갱신:**
```bash
sudo apt update
```

---

## Step 6 — ROS2 Humble 설치

```bash
sudo apt install -y \
  ros-humble-ros-base \
  python3-colcon-common-extensions \
  python3-argcomplete \
  ros-dev-tools
```

> 설치 중 **"Which services should be restarted?"** 화면이 나오면
> 아무것도 선택하지 않고 **Tab → OK → 엔터** 로 넘어가세요.

---

## Step 7 — 데모 노드 설치

> `ros-base` 에는 demo_nodes가 포함되지 않으므로 별도 설치 필요

```bash
sudo apt install -y \
  ros-humble-demo-nodes-cpp \
  ros-humble-demo-nodes-py
```

---

## Step 8 — 환경 변수 등록

```bash
echo "source /opt/ros/humble/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

---

## Step 9 — CycloneDDS 설치 및 설정

```bash
sudo apt install -y ros-humble-rmw-cyclonedds-cpp
```

```bash
echo "export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp" >> ~/.bashrc
source ~/.bashrc
```

**확인:**
```bash
echo $RMW_IMPLEMENTATION
```

**정상 결과:**
```
rmw_cyclonedds_cpp
```

---

## Step 10 — 설치 확인

**환경 변수 확인:**
```bash
printenv | grep ROS
```

**패키지 수 확인:**
```bash
ros2 pkg list | wc -l
```
> 180개 이상이면 정상

**전체 진단:**
```bash
ros2 doctor
```

**정상 결과:**
```
All 5 checks passed
```

> `UserWarning: XXX has been updated to a new version` 경고는 무시해도 됩니다.

---

## Step 11 — talker / listener 통신 테스트

SSH 환경에서는 tmux로 두 터미널을 동시에 운용합니다.

**tmux 설치:**
```bash
sudo apt install -y tmux
```

**tmux 시작:**
```bash
tmux
```

**터미널 1 — talker 실행:**
```bash
ros2 run demo_nodes_cpp talker
```

**터미널 2 열기:**
```
Ctrl+B 누른 후 "  (가로 분할)
Ctrl+B 방향키    (창 이동)
```

**터미널 2 — listener 실행:**
```bash
ros2 run demo_nodes_cpp listener
```

**정상 결과:**
```
[INFO] [talker]: Publishing: 'Hello World: 1'
[INFO] [listener]: I heard: Hello World: 1
[INFO] [talker]: Publishing: 'Hello World: 2'
[INFO] [listener]: I heard: Hello World: 2
```

종료: 각 터미널에서 `Ctrl+C`

---

## Step 12 — 재부팅 및 최종 확인

모든 설정 완료 후 **단 1회** 재부팅합니다.

```bash
sudo reboot
```

재접속 후 확인:
```bash
echo $RMW_IMPLEMENTATION
ros2 doctor
```

---

## ✅ 완료 기준

| 항목 | 확인 방법 | 정상 결과 |
|------|-----------|-----------|
| aarch64 확인 | `uname -m` | `aarch64` |
| ROS2 설치 | `ros2 pkg list \| wc -l` | 180+ |
| 환경 변수 | `printenv \| grep ROS` | ROS 관련 변수 출력 |
| CycloneDDS | `echo $RMW_IMPLEMENTATION` | `rmw_cyclonedds_cpp` |
| 전체 진단 | `ros2 doctor` | `All 5 checks passed` |
| 통신 테스트 | talker/listener | `I heard: Hello World` 출력 |

---

## ⚠️ 시행착오 요약 (참고)

| 문제 | 원인 | 해결 |
|------|------|------|
| 재부팅 후 SSH 불가, 화면 없음 | `full-upgrade` 로 커널 교체 | `upgrade` 금지, `update` 만 사용 |
| Read-only file system | 저속 SD카드(U1)에서 대용량 쓰기 실패 | `upgrade` 생략, 고속 SD카드 사용 |
| `ros2 --version` 에러 | Humble은 `--version` 미지원 | `ros2 doctor` 또는 `ros2 pkg list` 로 확인 |
| `demo_nodes_cpp` not found | `ros-base` 에 미포함 | `ros-humble-demo-nodes-cpp` 별도 설치 |
| `$RMW_IMPLEMENTATION` 빈값 | CycloneDDS 미설치 | Step 9 진행 |

