# Day 6 — 복습 & 트러블슈팅

> **목표:** Week 1에서 배운 내용을 복습하고, 자주 발생하는 문제와 해결 방법을 익힌다.

---

## 6.1 Week 1 핵심 요약

### Day 1-3: 설치 완료 체크리스트

```
✅ Raspberry Pi 4B → Ubuntu Server 22.04 LTS
✅ SSH 접속 가능 (ssh ubuntu@turtlebot-pi.local)
✅ ROS2 Humble ros-base 설치
✅ Remote PC → Ubuntu 22.04 Desktop
✅ ROS2 Humble desktop 설치
✅ Gazebo Classic 11 설치
✅ RPi ↔ PC ROS_DOMAIN_ID=30 동일 설정
✅ RPi talker / PC listener 통신 성공
```

### Day 4-5: 명령어 요약

| 명령어 | 설명 | 예시 |
|--------|------|------|
| `ros2 run <pkg> <node>` | 노드 실행 | `ros2 run demo_nodes_py talker` |
| `ros2 node list` | 노드 목록 | |
| `ros2 node info <node>` | 노드 상세 | `ros2 node info /talker` |
| `ros2 topic list` | 토픽 목록 | |
| `ros2 topic echo <topic>` | 토픽 모니터 | `ros2 topic echo /chatter` |
| `ros2 topic pub <topic> <type>` | 메시지 발행 | `ros2 topic pub /cmd_vel geometry_msgs/Twist ...` |
| `ros2 topic hz <topic>` | 발행 주기 | `ros2 topic hz /chatter` |
| `ros2 service list` | 서비스 목록 | |
| `ros2 service call <srv>` | 서비스 호출 | `ros2 service call /spawn turtlesim/srv/Spawn ...` |
| `ros2 action list` | 액션 목록 | |
| `ros2 action send_goal <action>` | 액션 전송 | |
| `ros2 param list` | 파라미터 목록 | |
| `ros2 param get <node> <param>` | 파라미터 조회 | |
| `ros2 param set <node> <param> <val>` | 파라미터 설정 | |
| `ros2 launch <pkg> <launch_file>` | 런치 파일 실행 | `ros2 launch turtlesim multisim.launch.py` |
| `colcon build` | 워크스페이스 빌드 | `colcon build --symlink-install` |
| `rqt_graph` | 통신 그래프 | |
| `rqt_console` | 로그 콘솔 | |

---

## 6.2 ROS2 통신 아키텍처 이해

```
┌─────────────────────────────────────────────────┐
│                  Application Layer               │
│  ┌─────────┐  ┌─────────┐  ┌─────────────────┐  │
│  │ Node A  │  │ Node B  │  │    Node C       │  │
│  │ (Pub)   │  │ (Sub)   │  │ (Service Server)│  │
│  └────┬────┘  └────┬────┘  └────────┬────────┘  │
│       │             │                │           │
├───────┴─────────────┴────────────────┴───────────┤
│                  RCL (Client Library)             │
│           rclpy / rclcpp / rclc                   │
├──────────────────────────────────────────────────┤
│                   RMW (Middleware)                │
│   Fast DDS / Cyclone DDS / RTI Connext / etc.    │
├──────────────────────────────────────────────────┤
│                   DDS Protocol                    │
│            UDP/IP (멀티캐스트 + 유니캐스트)       │
└──────────────────────────────────────────────────┘
```

### 주요 개념 연결고리

- **Node** = 실행 단위 (프로세스)
- **Topic** = 데이터 스트림 (Pub/Sub)
- **Service** = 요청/응답 (Request/Response)
- **Action** = 목표 + 피드백 + 결과 (Goal/Feedback/Result)
- **Parameter** = 노드 설정 값

---

## 6.3 DDS (Data Distribution Service) 이해

ROS2는 DDS를 미들웨어로 사용합니다. 이는 ROS1과의 가장 큰 차이점입니다.

### DDS vs ROS1 통신

| 특성 | ROS1 (TCPROS) | ROS2 (DDS) |
|------|--------------|------------|
| 통신 방식 | TCP 연결 (peer-to-peer) | DDS (발행-구독, 자동 발견) |
| QoS | 없음 | 있음 (신뢰성, 지속성 등) |
| 보안 | 없음 | SROS2 (DDS 보안) |
| 실시간 | 지원 안 함 | 지원 가능 |
| 멀티캐스트 | 지원 안 함 | 기본 사용 |

### ROS_DOMAIN_ID 상세

- **역할:** 동일 네트워크에서 여러 ROS2 시스템을 격리
- **범위:** 0 ~ 232 (기본값 30 권장)
- **원리:** DDS 파티셔닝 - 서로 다른 ID의 노드는 통신 불가
- **설정:** `export ROS_DOMAIN_ID=30`

```bash
# 현재 ROS_DOMAIN_ID 확인
echo $ROS_DOMAIN_ID

# 일시적 변경
export ROS_DOMAIN_ID=50

# 영구 설정 (~/.bashrc에 추가)
echo 'export ROS_DOMAIN_ID=30 #TURTLEBOT3' >> ~/.bashrc
```

### RMW (ROS Middleware) 선택

```bash
# 현재 RMW 확인
ros2 doctor | grep middleware

# RMW 변경 (Cyclone DDS가 Fast DDS보다 안정적일 때 있음)
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp

# 영구 설정
echo 'export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp' >> ~/.bashrc
```

---

## 6.4 일반적인 문제 & 해결 방법

### 문제 1: "ros2: command not found"

**원인:** ROS2 환경이 source되지 않음

**해결:**

```bash
# 임시 해결
source /opt/ros/humble/setup.bash

# 영구 해결 (bashrc 확인)
echo 'source /opt/ros/humble/setup.bash' >> ~/.bashrc
```

### 문제 2: RPi와 PC 간 토픽이 보이지 않음

**원인:** DDS 통신 문제 (멀티캐스트, 방화벽, ROS_DOMAIN_ID)

**해결 (순서대로 확인):**

```bash
# 1. 네트워크 연결 확인
ping turtlebot-pi.local
ping [RPi IP]

# 2. ROS_DOMAIN_ID 확인 (양쪽 모두)
echo $ROS_DOMAIN_ID    # 30이어야 함

# 3. DDS 멀티캐스트 확인 (RPi)
ros2 run demo_nodes_py talker &
ros2 topic list        # /chatter 보여야 함

# 4. PC에서 RPi 토픽 확인
ros2 topic list        # /chatter가 보여야 함
ros2 topic echo /chatter
```

**만약 안 되면:**

```bash
# RMW 변경 시도 (양쪽 모두)
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp

# 방화벽 확인
sudo ufw disable  # 임시 비활성화 후 테스트

# WiFi 전원 관리 비활성화
sudo iwconfig wlan0 power off
```

### 문제 3: colcon build 실패

```bash
# 로그 확인
cd ~/ros2_ws
colcon build --symlink-install --event-handlers console_direct+

# 캐시 정리 후 재시도
rm -rf build/ install/ log/
colcon build --symlink-install
```

### 문제 4: Gazebo 실행 안 됨

```bash
# 기존 프로세스 정리
killall gzserver gzclient

# 환경 변수 설정
export LIBGL_ALWAYS_SOFTWARE=1

# swiftshader 설치 (소프트웨어 렌더링)
sudo apt install -y libgl1-mesa-glx
```

### 문제 5: RPi에 SSH 접속이 느림

```bash
# SSH 설정 최적화
sudo nano /etc/ssh/sshd_config

# 추가 또는 수정:
UseDNS no
GSSAPIAuthentication no

# SSH 재시작
sudo systemctl restart sshd
```

---

## 6.5 ROS2 Doctor 활용

```bash
# ROS2 환경 진단
ros2 doctor

# 상세 진단
ros2 doctor --report

# 특정 영역 진단
ros2 doctor --topic /chatter

# 의존성 문제 확인
ros2 doctor --faq
```

---

## 6.6 디버깅 워크플로우

ROS2 개발 시 체계적인 디버깅 순서:

```
1. 증상 확인
   "노드가 실행되나요?" → ros2 node list
   "토픽이 보이나요?" → ros2 topic list/echo
   
2. 통신 계층 확인
   네트워크: ping, ROS_DOMAIN_ID
   DDS: RMW_IMPLEMENTATION, ros2 doctor
   RCL: 로그 레벨 설정 (export RCUTILS_CONSOLE_OUTPUT_FORMAT)
   
3. 코드 진단
   rclpy: try/except로 예외 캐치
   rclcpp: RCLCPP_DEBUG 사용
   rqt_console: 모든 로그 레벨 모니터링
   
4. 리소스 확인
   CPU, 메모리, 디스크, 네트워크 대역폭
```

---

## 6.7 효과적인 학습 팁

### 터미널 관리

```bash
# tmux 설정 예시 (.tmux.conf)
set -g mouse on
set -g history-limit 10000

# tmux 세션 시작
tmux new -s ros2_work

# 창 분할 및 배치
# 좌측: RPi SSH | 우측 상단: 코드 | 우측 하단: ROS2 명령어
```

### 유용한 bash 별칭

```bash
# ~/.bash_aliases에 추가
alias tb3_bringup='ros2 launch turtlebot3_bringup robot.launch.py'
alias tb3_teleop='ros2 run turtlebot3_teleop teleop_keyboard'
alias tb3_slam='ros2 launch slam_toolbox online_async_launch.py params_file:=burger.yaml use_sim_time:=false'
alias cw='cd ~/ros2_ws && colcon build --symlink-install'
alias sw='source ~/ros2_ws/install/setup.bash'
```

---

## 📝 연습 문제

1. **Communication Test:** RPi와 PC 사이에서 `ros2 doctor`를 양쪽에서 실행하고 결과를 비교하세요
2. **RMW Swap:** RPi와 PC의 RMW를 `rmw_fastrtps_cpp`에서 `rmw_cyclonedds_cpp`로 변경하고 통신 차이를 경험해보세요
3. **Multi-Domain:** ROS_DOMAIN_ID를 30과 50 두 개로 분리하여 각 도메인에서 통신이 격리되는 것을 확인하세요
4. **DDS Debug:** RPi에서 `ros2 run demo_nodes_py talker` 실행 중, PC에서 `ros2 topic echo /chatter`가 안 될 때 체계적으로 진단하는 절차를 기록하세요
5. **Diagnosis:** RPi의 메모리 상태 (`free -h`), CPU 온도, 네트워크 latency를 확인하고 ROS2 통신 품질과의 상관관계를 분석하세요
