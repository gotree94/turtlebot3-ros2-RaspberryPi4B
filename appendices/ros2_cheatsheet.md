# 부록 B: ROS2 명령어 치트시트

> **자주 사용하는 ROS2 명령어를 한눈에 정리**

---

## 1. 기본 명령어

```bash
# ROS2 환경 설정
source /opt/ros/humble/setup.bash

# 설치된 ROS2 버전 확인
ros2 --version

# 도움말
ros2 -h
ros2 <command> -h
```

---

## 2. 노드 (Node)

```bash
# 노드 목록
ros2 node list

# 노드 상세 정보
ros2 node info <node_name>

# 노드 실행
ros2 run <package_name> <executable_name>
ros2 run <package_name> <executable_name> --ros-args -r __ns:=/namespace
```

---

## 3. 토픽 (Topic)

```bash
# 토픽 목록
ros2 topic list
ros2 topic list -t               # 타입 포함

# 토픽 정보
ros2 topic info <topic_name>
ros2 topic type <topic_name>

# 토픽 모니터링
ros2 topic echo <topic_name>
ros2 topic echo <topic_name> --once       # 1회만
ros2 topic echo <topic_name> --field data # 특정 필드만

# 토픽 통계
ros2 topic hz <topic_name>                # 발행 주기
ros2 topic bw <topic_name>                # 대역폭
ros2 topic delay <topic_name>             # 지연 시간

# 메시지 발행
ros2 topic pub <topic_name> <msg_type> "<args>"
ros2 topic pub <topic_name> <msg_type> "<args>" -r 10  # 10Hz로 반복
ros2 topic pub --once <topic_name> <msg_type> "<args>" # 1회

# 메시지 타입 확인
ros2 interface show <msg_type>
ros2 interface list | grep <keyword>
```

### 자주 사용하는 메시지 예시

```bash
# Twist (속도 명령)
ros2 topic pub /cmd_vel geometry_msgs/Twist \
  "{linear: {x: 0.1, y: 0.0, z: 0.0}, angular: {x: 0.0, y: 0.0, z: 0.1}}" \
  -r 10

# String
ros2 topic pub /hello std_msgs/String "data: 'Hello ROS2'"

# Float64
ros2 topic pub /temperature std_msgs/Float64 "data: 25.5"

# PoseStamped
ros2 topic pub /goal_pose geometry_msgs/PoseStamped \
  "{header: {frame_id: 'map'}, pose: {position: {x: 1.0, y: 0.0, z: 0.0}, orientation: {w: 1.0}}}"
```

---

## 4. 서비스 (Service)

```bash
# 서비스 목록
ros2 service list
ros2 service list -t              # 타입 포함

# 서비스 타입 확인
ros2 service type <service_name>

# 서비스 호출
ros2 service call <service_name> <service_type> "<args>"

# 서비스 타입 구조 확인
ros2 interface show <service_type>
```

### 서비스 예시

```bash
# Spawn turtle
ros2 service call /spawn turtlesim/srv/Spawn \
  "{x: 2.0, y: 2.0, theta: 0.0, name: 'turtle2'}"

# 배터리 상태
ros2 service call /battery_state/set_parameters \
  rcl_interfaces/srv/SetParameters
```

---

## 5. 액션 (Action)

```bash
# 액션 목록
ros2 action list
ros2 action list -t               # 타입 포함

# 액션 정보
ros2 action info <action_name>

# 액션 타입 확인
ros2 action type <action_name>

# 액션 전송
ros2 action send_goal <action_name> <action_type> "<args>"
ros2 action send_goal <action_name> <action_type> "<args>" --feedback

# 액션 타입 구조
ros2 interface show <action_type>
```

### 액션 예시

```bash
# 거북이 회전
ros2 action send_goal /turtle1/rotate_absolute \
  turtlesim/action/RotateAbsolute "{theta: 3.14}" --feedback

# Nav2 네비게이션
ros2 action send_goal /navigate_to_pose \
  nav2_msgs/action/NavigateToPose \
  "{pose: {header: {frame_id: 'map'}, pose: {position: {x: 1.0, y: 1.0}}}}"
```

---

## 6. 파라미터 (Parameter)

```bash
# 파라미터 목록
ros2 param list

# 파라미터 값 조회
ros2 param get <node_name> <param_name>

# 파라미터 값 설정
ros2 param set <node_name> <param_name> <value>

# 파라미터 덤프 (파일 저장)
ros2 param dump <node_name>
ros2 param dump <node_name> --output-file params.yaml

# 파라미터 로드
ros2 param load <node_name> params.yaml

# 파라미터 설명 확인
ros2 param describe <node_name> <param_name>
```

---

## 7. Launch (런치 파일)

```bash
# 런치 파일 실행
ros2 launch <package_name> <launch_file.py>

# 인자 확인
ros2 launch <package_name> <launch_file.py> --show-args

# 인자 전달
ros2 launch <package_name> <launch_file.py> arg1:=value1 arg2:=value2
```

---

## 8. 빌드 (Build)

```bash
# 워크스페이스 빌드
cd ~/ros2_ws
colcon build
colcon build --symlink-install     # 심볼릭 링크 (개발용)
colcon build --packages-select <pkg>  # 특정 패키지만
colcon build --parallel-workers 1   # 단일 코어 (RPi)
colcon build --event-handlers console_direct+  # 로그 상세 출력

# 캐시 정리 후 재빌드
rm -rf build/ install/ log/
colcon build --symlink-install

# 환경 설정
source install/setup.bash
```

### 패키지 생성

```bash
# Python 패키지
ros2 pkg create <package_name> \
  --build-type ament_python \
  --dependencies rclpy std_msgs

# C++ 패키지
ros2 pkg create <package_name> \
  --build-type ament_cmake \
  --dependencies rclcpp std_msgs
```

---

## 9. Bag (데이터 기록/재생)

```bash
# 모든 토픽 기록
ros2 bag record -a -o <bag_name>

# 특정 토픽만 기록
ros2 bag record /scan /odom /tf -o <bag_name>

# Bag 파일 정보
ros2 bag info <bag_name>

# Bag 파일 재생
ros2 bag play <bag_name>
ros2 bag play <bag_name> -r 0.5   # 0.5배속
ros2 bag play <bag_name> --loop   # 반복 재생
ros2 bag play <bag_name> --topics /scan /odom  # 특정 토픽만
```

---

## 10. 진단 & 디버깅

```bash
# ROS2 환경 진단
ros2 doctor
ros2 doctor --report

# 그래프 시각화
rqt_graph
rqt_graph -t        # 토픽 이름 표시

# 로그 콘솔
rqt_console

# 데이터 플롯
rqt_plot
rqt_plot /topic/field

# CLI 기반 모니터링
ros2 topic echo /topic | jq '.'   # JSON 포맷 예쁘게 출력

# 노드 라이프사이클 확인
ros2 lifecycle list

# 인터페이스 (메시지/서비스/액션) 검색
ros2 interface list
ros2 interface package <package_name>
```

---

## 11. TurtleBot3 특화 명령어

```bash
# 환경 변수 설정
export TURTLEBOT3_MODEL=burger
export ROS_DOMAIN_ID=30
export LDS_MODEL=LDS-01  # 또는 LDS-02

# Bringup
ros2 launch turtlebot3_bringup robot.launch.py

# Teleop (키보드)
ros2 run turtlebot3_teleop teleop_keyboard

# RViz2
ros2 launch turtlebot3_bringup rviz2.launch.py

# SLAM
ros2 launch slam_toolbox online_async_launch.py \
  slam_params_file:=/opt/ros/humble/share/turtlebot3_navigation2/param/burger.yaml \
  use_sim_time:=false

# 지도 저장
ros2 run nav2_map_server map_saver_cli -f ~/my_map

# Nav2
ros2 launch turtlebot3_navigation2 navigation2.launch.py \
  map:=/home/ubuntu/my_map.yaml

# Gazebo 시뮬레이션
ros2 launch turtlebot3_gazebo empty_world.launch.py
ros2 launch turtlebot3_gazebo turtlebot3_world.launch.py

# TF 확인
ros2 run tf2_tools view_frames.py
ros2 run tf2_ros tf2_echo base_link base_scan

# Fake Node (오프라인 테스트)
ros2 launch turtlebot3_fake_node turtlebot3_fake_node.launch.py
```

---

## 12. 환경 변수

```bash
# bashrc에 추가할 환경 변수들

# ROS2
source /opt/ros/humble/setup.bash

# TurtleBot3
export TURTLEBOT3_MODEL=burger
export ROS_DOMAIN_ID=30
export LDS_MODEL=LDS-01

# OpenCR
export OPENCR_PORT=/dev/ttyACM0
export OPENCR_MODEL=burger

# Gazebo
source /usr/share/gazebo/setup.sh

# DDS 선택
export RMW_IMPLEMENTATION=rmw_fastrtps_cpp
# export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp

# 로그 포맷
export RCUTILS_CONSOLE_OUTPUT_FORMAT="[{severity}] [{time}] [{name}]: {message}"
export RCUTILS_LOGGING_USE_STDOUT=1
```

---

## 13. 유용한 tmux 단축키

```bash
# tmux 세션 시작
tmux new -s ros2_session

# 창 분할
Ctrl+B %        # 세로 분할
Ctrl+B "        # 가로 분할

# 창 이동
Ctrl+B 방향키   # 방향으로 이동
Ctrl+B o        # 다음 창
Ctrl+B q        # 번호 표시 후 이동

# 스크롤 모드
Ctrl+B [        # 스크롤 시작 (방향키 또는 Page Up/Down)
q               # 스크롤 종료

# 세션 분리/재접속
Ctrl+B d        # 세션 분리 (detach)
tmux attach     # 세션 재접속

# 창 종료
Ctrl+B &        # 현재 창 종료 (확인 필요)
exit            # 셸 종료 → 자동 창 종료
```
