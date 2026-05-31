# Day 4 — ROS2 핵심 개념 학습

> **목표:** ROS2의 5대 핵심 개념 (Node, Topic, Service, Action, Parameter)을 이해하고 실제 명령어로 실습한다.

---

## 4.1 개요

ROS2는 로봇 소프트웨어 개발을 위한 분산 통신 프레임워크입니다.
이번 시간에는 ROS2의 5대 핵심 개념을 **명령어 실습**을 통해 배웁니다.

> 💡 **ROS2의 핵심 철학:** "모든 것은 독립적인 노드(node)이며, 노드들은 토픽(topic), 서비스(service), 액션(action)으로 통신한다"

---

## 4.2 Nodes (노드)

**노드**는 ROS2에서 실행 가능한 최소 단위의 프로세스입니다.
각 노드는 하나의 특정 기능을 담당합니다 (예: LIDAR 드라이버, 모터 제어, 경로 계획).

```bash
# 노드 목록 보기
ros2 node list

# 특정 노드의 상세 정보 확인
ros2 node info /talker

# 노드 출력 메시지:
#   Subscribers: /parameter_events
#   Publishers: /chatter, /parameter_events, /rosout
#   Service Servers: ...
#   Service Clients: ...
#   Action Servers: ...
#   Action Clients: ...
```

### 노드 이름 규칙

- 노드 이름은 **/**로 시작 (예: `/talker`, `/listener`)
- 네임스페이스 사용 가능: `/robot1/lidar`, `/robot1/camera`
- 이름은 알파벳, 숫자, 언더스코어만 사용

---

## 4.3 Topics (토픽)

**토픽**은 노드 간 **비동기 단방향** 통신 채널입니다.
Publisher가 메시지를 발행(publish)하고, Subscriber가 수신(subscribe)합니다.

> **Pub-Sub 패턴:** 발행자와 구독자는 서로를 알 필요가 없음 (느슨한 결합)

```bash
# 활성 토픽 목록
ros2 topic list

# 토픽 상세 정보
ros2 topic info /chatter

# 토픽 메시지 타입 확인
ros2 topic type /chatter

# 토픽 메시지 실시간 모니터링
ros2 topic echo /chatter

# 토픽 발행 주기 확인
ros2 topic hz /chatter

# 토픽 대역폭 확인
ros2 topic bw /chatter

# 직접 메시지 발행
ros2 topic pub /cmd_vel geometry_msgs/Twist \
  "{linear: {x: 0.1, y: 0.0, z: 0.0}, angular: {x: 0.0, y: 0.0, z: 0.1}}" \
  --rate 10
```

### 메시지 타입 구조 확인

```bash
# 메시지 구조 보기
ros2 interface show geometry_msgs/Twist
ros2 interface show std_msgs/String
ros2 interface show sensor_msgs/LaserScan
```

### 토픽 vs 서비스 vs 액션

| 구분 | Topic | Service | Action |
|------|-------|---------|--------|
| 통신 방향 | 단방향 | 양방향 (request/response) | 양방향 + 피드백 |
| 지속성 | 지속적 | 일회성 | 지속적 + 취소 가능 |
| 사용처 | 센서 데이터 | 설정/조회 | 복잡한 태스크 |

---

## 4.4 Services (서비스)

**서비스**는 **동기식 요청-응답** 통신입니다.
클라이언트가 요청(request)을 보내면 서버가 응답(response)을 반환합니다.

```bash
# 서비스 목록
ros2 service list

# 서비스 타입 확인
ros2 service type /clear

# 서비스 호출
ros2 service call /spawn turtlesim/srv/Spawn "{x: 2.0, y: 2.0, theta: 0.0, name: 'turtle2'}"
```

### Turtlesim으로 실습

```bash
# turtlesim 설치 (아직 없으면)
sudo apt install -y ros-humble-turtlesim

# turtlesim 실행
ros2 run turtlesim turtlesim_node

# 새 터미널에서 서비스 호출
ros2 service list
ros2 service type /kill
ros2 service call /kill turtlesim/srv/Kill "{name: 'turtle1'}"
```

---

## 4.5 Actions (액션)

**액션**은 **비동기식 목표 지향** 통신입니다.
서비스와 달리 실행 중에 **피드백(feedback)** 을 지속적으로 받을 수 있고, 취소(cancel)도 가능합니다.

```bash
# 액션 목록
ros2 action list

# 액션 타입 확인
ros2 action list -t

# 액션 정보 확인
ros2 action info /turtle1/rotate_absolute

# 액션 인터페이스 확인
ros2 interface show turtlesim/action/RotateAbsolute

# 액션 전송 (피드백 포함)
ros2 action send_goal /turtle1/rotate_absolute \
  turtlesim/action/RotateAbsolute "{theta: 1.57}" \
  --feedback
```

---

## 4.6 Parameters (파라미터)

**파라미터**는 노드의 설정값을 런타임에 동적으로 변경할 수 있는 메커니즘입니다.

```bash
# 파라미터 목록
ros2 param list

# 특정 파라미터 값 확인
ros2 param get /turtlesim background_b

# 파라미터 값 설정
ros2 param set /turtlesim background_r 150

# 파라미터를 파일로 저장
ros2 param dump /turtlesim

# 파라미터 파일 로드
ros2 param load /turtlesim ./turtlesim.yaml
```

---

## 4.7 Launch Files (런치 파일)

여러 노드를 한 번에 실행하기 위한 시스템:

```bash
# 기본 런치 파일 실행
ros2 launch turtlesim multisim.launch.py

# 런치 파일 구조 확인
ros2 launch --show-args turtlesim multisim.launch.py
```

런치 파일 예시 (`turtlesim_mimic.launch.py` 개념):

```python
from launch import LaunchDescription
from launch_ros.actions import Node

def generate_launch_description():
    return LaunchDescription([
        Node(
            package='turtlesim',
            executable='turtlesim_node',
            name='sim',
            output='screen'
        ),
        Node(
            package='turtlesim',
            executable='mimic',
            name='mimic',
            remappings=[
                ('/input/pose', '/turtle1/pose'),
            ]
        ),
    ])
```

---

## 4.8 RQt 도구 모음

ROS2의 GUI 도구 모음:

```bash
# RQt 전체 실행
rqt

# 통신 그래프 시각화
rqt_graph

# 콘솔 로그 모니터링
rqt_console

# 플로팅 (데이터 그래프)
rqt_plot

# 토픽 모니터
rqt_topic

# 배거 (데이터 녹음/재생)
rqt_bag
```

---

## 4.9 ROS2 Graph Visualizer

노드 간 통신 구조를 시각화:

```bash
# turtlesim 실행 후
ros2 run turtlesim turtlesim_node &
ros2 run turtlesim turtle_teleop_key &

# 그래프 확인
rqt_graph
```

---

## 📝 연습 문제

> 실제 turtlesim을 실행하며 아래 실습을 진행하세요.

1. **토픽 실습:** `/turtle1/cmd_vel` 토픽의 타입을 확인하고, `ros2 topic pub`으로 거북이를 정사각형으로 움직이게 하세요
2. **서비스 실습:** `/spawn` 서비스를 호출하여 새로운 거북이 3마리를 추가하세요
3. **액션 실습:** 모든 거북이를 각도 3.14 rad으로 회전시키는 액션 goal을 보내세요
4. **파라미터 실습:** 배경색을 RGB 값 (255, 100, 50)으로 변경하고, 파라미터를 YAML 파일로 저장하세요
5. **통합 실습:** `rqt_graph`를 실행하고, turtlesim 노드들의 통신 관계를 스크린샷으로 저장하세요
6. **심화:** `rqt_plot`으로 `/turtle1/pose`의 x, y, theta를 동시에 그래프로 출력하세요

---

## ⚠️ 트러블슈팅

| 문제 | 해결 방법 |
|------|----------|
| rqt가 실행 안 됨 | `sudo apt install -y ros-humble-rqt*` 후 재시도 |
| turtlesim 패키지 없음 | `sudo apt install -y ros-humble-turtlesim` |
| action send_goal 실패 | action server가 실행 중인지 확인 (`ros2 action list`) |
| rqt_graph에 아무것도 안 보임 | `source /opt/ros/humble/setup.bash`가 적용되었는지 확인 |
