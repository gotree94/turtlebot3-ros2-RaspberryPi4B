# Day 9 — Remote PC TurtleBot3 패키지 설치

> **목표:** Remote PC에 TurtleBot3 관련 패키지를 설치하고, Gazebo 시뮬레이션으로 Week 2 내용을 미리 테스트한다.

---

## 9.1 개요

Remote PC는 TurtleBot3의 두뇌가 아닌, **데이터 시각화 및 고급 연산**을 담당합니다.
RPi에서 처리하기 어려운 SLAM, Navigation2, RViz2, rqt 등을 PC에서 실행합니다.

---

## 9.2 Debian 패키지 설치 (권장)

가장 간단하고 안정적인 방법:

```bash
# 모든 TurtleBot3 관련 패키지 설치
sudo apt install -y ros-humble-turtlebot3*

# 설치 확인
ros2 pkg list | grep turtlebot3
```

**설치되는 패키지:**
- `turtlebot3` — 메타패키지
- `turtlebot3_bringup` — 로봇 구동 런치 파일
- `turtlebot3_cartographer` — Cartographer SLAM
- `turtlebot3_description` — URDF 모델
- `turtlebot3_example` — 예제 코드
- `turtlebot3_fake_node` — 시뮬레이션용 페이크 노드
- `turtlebot3_msgs` — 커스텀 메시지
- `turtlebot3_navigation2` — Navigation2 설정
- `turtlebot3_node` — 메인 로봇 노드
- `turtlebot3_teleop` — 키보드/조이스틱 원격 제어

---

## 9.3 환경 변수 설정 (PC)

```bash
cat >> ~/.bashrc << 'EOF'

# TurtleBot3 Settings
export TURTLEBOT3_MODEL=burger
export ROS_DOMAIN_ID=30
export LDS_MODEL=LDS-01
EOF

source ~/.bashrc
```

---

## 9.4 시뮬레이션 패키지 설치

Gazebo에서 TurtleBot3를 시뮬레이션하기 위한 패키지:

### 방법 A: Debian 패키지

```bash
sudo apt install -y ros-humble-turtlebot3-simulations
```

### 방법 B: 소스 빌드 (최신 업데이트 필요 시)

```bash
mkdir -p ~/tb3_sim_ws/src
cd ~/tb3_sim_ws/src
git clone -b humble https://github.com/ROBOTIS-GIT/turtlebot3_simulations.git
cd ~/tb3_sim_ws
colcon build --symlink-install
echo 'source ~/tb3_sim_ws/install/setup.bash' >> ~/.bashrc
source ~/.bashrc
```

---

## 9.5 Gazebo 시뮬레이션 실행 테스트

### 1) 빈 월드에서 TurtleBot3 실행

```bash
export TURTLEBOT3_MODEL=burger
ros2 launch turtlebot3_gazebo empty_world.launch.py
```

### 2) 월드가 있는 환경에서 실행

```bash
# TurtleBot3 전용 월드 (벽, 장애물 포함)
ros2 launch turtlebot3_gazebo turtlebot3_world.launch.py

# 집 환경
ros2 launch turtlebot3_gazebo turtlebot3_house.launch.py
```

### 3) RViz2로 시각화

```bash
# Gazebo가 실행 중인 상태에서 새 터미널
ros2 launch turtlebot3_bringup rviz2.launch.py

# 또는 직접 RViz2 실행
rviz2 -d /opt/ros/humble/share/turtlebot3_description/rviz/model.rviz
```

### 4) 키보드로 시뮬레이션 로봇 제어

```bash
# 새 터미널
ros2 run turtlebot3_teleop teleop_keyboard
```

```
사용법:
        w
   a    s    d
        x

w/x: 전진/후진
a/d: 좌회전/우회전
s: 정지
Ctrl+C: 종료
```

---

## 9.6 노드 구조 이해 (시뮬레이션)

Gazebo 시뮬레이션에서 실행되는 노드들:

```bash
# Gazebo 실행 후 노드 목록 확인
ros2 node list
```

**예시 출력:**
```
/gazebo
/robot_state_publisher
/turtlebot3_diff_drive
```

**rqt_graph로 통신 구조 시각화:**

```bash
rqt_graph
```

```
    ┌─────────────────┐    /scan     ┌────────────┐
    │   Gazebo        │────────────► │  RViz2     │
    │  (LDS Sensor)   │    /odom     │            │
    │                 │────────────► │  (시각화)   │
    │  (Motor Control)│    /tf       │            │
    │                 │────────────► │            │
    └────────┬────────┘              └────────────┘
             │
             │  /cmd_vel
             │
    ┌────────▼────────┐
    │  teleop_keyboard │
    └─────────────────┘
```

---

## 9.7 Fake Node를 이용한 오프라인 테스트

실제 로봇 없이도 PC에서 TurtleBot3의 토픽을 테스트:

```bash
# 페이크 노드 실행 (실제 하드웨어 없이 시뮬레이션)
ros2 launch turtlebot3_fake_node turtlebot3_fake_node.launch.py

# 다른 터미널에서 토픽 확인
ros2 topic list
# /cmd_vel, /odom, /scan (fake), /tf 등이 보여야 함

# 텔레옵 실행
ros2 run turtlebot3_teleop teleop_keyboard

# RViz2 실행
ros2 run rviz2 rviz2 -d /opt/ros/humble/share/turtlebot3_description/rviz/model.rviz
```

---

## 9.8 TurtleBot3 Burger 하드웨어 사양

실제 로봇을 사용하기 전에 제원을 숙지합니다:

| 사양 | 값 |
|------|-----|
| 최대 이동 속도 | 0.22 m/s |
| 최대 회전 속도 | 2.84 rad/s (162.7°/s) |
| 최대 탑재 하중 | 15 kg (바퀴 제외) |
| 크기 (L x W x H) | 138 x 178 x 192 mm |
| 무게 | 약 1 kg |
| 배터리 | 11.1V Li-Po 1800mAh |
| 사용 시간 | 약 2-3시간 (사용 환경에 따라 다름) |
| LDS 사양 | 360°, 0.5-12m, 360°/s |

---

## 📝 연습 문제

1. Gazebo의 `turtlebot3_world.launch.py` 실행 후 `ros2 topic echo /scan`으로 LIDAR 데이터를 확인하고, 데이터 구조를 분석하세요
2. `rqt_plot`으로 `/odom`에서 선속도(linear.x)와 각속도(angular.z)를 실시간 그래프로 출력하세요
3. 텔레옵 키보드로 TurtleBot3를 정사각형 경로로 주행시켜보고, `/odom` 데이터를 확인하세요
4. `ros2 launch turtlebot3_gazebo turtlebot3_world.launch.py`와 `turtlebot3_house.launch.py`의 환경 차이를 비교하세요
5. 페이크 노드와 실제 Gazebo 시뮬레이션의 `/scan` 데이터 차이점을 분석하세요

---

## ⚠️ 트러블슈팅

| 문제 | 해결 방법 |
|------|----------|
| `turtlebot3_simulations` 설치 안 됨 | `sudo apt update` 후 재시도 또는 소스 빌드 |
| Gazebo 모델 다운로드 느림 | `export GAZEBO_MODEL_DATABASE_URI=""`로 자동 다운로드 비활성화 |
| Gazebo 화면이 검게 나옴 | `export LIBGL_ALWAYS_SOFTWARE=1` 실행 후 재시도 |
| RViz2에서 로봇 모델 안 보임 | `RobotModel` 디스플레이 추가 후 Description Topic 확인 |
| teleop_keyboard가 반응 없음 | 실행 중인 터미널이 포커스되어 있는지 확인 |
