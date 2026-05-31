# Week 1 연습문제

> **범위:** Day 1 ~ Day 7 (환경 구축 & ROS2 기초)

---

## Day 1 — RPi OS 설치

### 기본 문제

1. RPi의 CPU 온도를 확인하는 명령어 3가지를 찾아 실행하고 결과를 기록하세요
   ```bash
   # 힌트: /sys/class/thermal/thermal_zone0/temp
   ```

2. `htop` 명령어로 현재 프로세스 목록을 확인하고, CPU와 메모리 사용률을 기록하세요

3. Remote PC에서 RPi로 SSH 키 기반 로그인을 설정하세요
   ```bash
   # 힌트: ssh-keygen, ssh-copy-id, ssh-add
   ```

4. RPi의 WiFi 신호 강도를 확인하는 명령어를 찾아 실행하세요

### 심화 문제

5. RPi 4B 8GB의 idle 상태와 stress 상태의 온도 차이를 측정하고 그래프로 그리세요
   ```bash
   # sudo apt install stress
   # stress --cpu 4 --timeout 120
   ```

6. netplan을 사용하여 고정 IP를 설정하고, 재부팅 후에도 IP가 유지되는지 확인하세요

---

## Day 2 — ROS2 Humble 설치

### 기본 문제

1. `ros2 run demo_nodes_cpp talker`와 `ros2 run demo_nodes_py listener`를 각각 실행하고, publisher/subscriber가 어떻게 통신하는지 설명하세요

2. `ros2 node list`와 `ros2 node info` 명령어로 talker 노드의 상세 정보를 확인하고, 다음 항목을 기록하세요:
   - Subscribers 목록
   - Publishers 목록
   - Service Servers 목록

3. `ros2 topic echo /chatter` 명령어로 발행되는 메시지 내용을 5개 이상 캡처하세요

### 심화 문제

4. `ros2 topic pub /test std_msgs/String "data: 'Hello ROS2'" -r 5`를 실행하고, `ros2 topic hz /test`로 발행 주기를 확인하세요

5. `ros2 topic bw /chatter`로 토픽의 대역폭을 확인하고, 발행 주기를 변경하면서 대역폭 변화를 측정하세요

6. `ros2 doctor --report`를 실행하고 결과를 분석하세요

---

## Day 3 — Remote PC 세팅

### 기본 문제

1. Gazebo의 `empty_world.launch.py` 대신 `turtlebot3_world.launch.py`를 실행하고 차이점을 서술하세요

2. PC에서 `ros2 run turtlebot3_teleop teleop_keyboard`를 실행하고 Gazebo 속 TurtleBot3를 움직여보세요

3. `rqt_graph`를 실행하여 Gazebo 속 노드들의 통신 구조를 스크린샷으로 저장하세요

### 심화 문제

4. RPi의 talker와 PC의 listener가 통신하는 동안 PC에서 `rqt_console`을 실행하고, 로그 레벨을 필터링하는 방법을 익히세요

5. `ROS_DOMAIN_ID`를 RPi와 PC에서 각각 30, 50으로 다르게 설정하고 통신이 안 되는 것을 확인한 후, 다시 30으로 맞춰 복구하세요

---

## Day 4 — ROS2 핵심 개념

### 기본 문제

1. turtlesim에서 `/turtle1/cmd_vel` 토픽의 타입을 확인하고, `ros2 topic pub`으로 거북이를 정사각형으로 움직이게 하는 명령어를 작성하세요

2. `/spawn` 서비스를 호출하여 새로운 거북이 3마리를 좌표 (2,2), (5,5), (8,2)에 추가하세요

3. 모든 거북이를 각도 3.14 rad으로 회전시키는 액션 goal을 보내고 피드백을 확인하세요

### 심화 문제

4. 배경색을 RGB 값 (255, 100, 50)으로 변경하고, 파라미터를 YAML 파일로 저장한 후 복원하세요

5. `rqt_graph`, `rqt_console`, `rqt_plot`을 동시에 실행하고 각 도구가 보여주는 정보의 차이점을 비교하세요

6. `rqt_plot`으로 `/turtle1/pose`의 x, y, theta를 동시에 그래프로 출력하고, 거북이가 움직일 때 변화를 관찰하세요

---

## Day 5 — 첫 ROS2 패키지

### 기본 문제

1. 0.5초마다 1씩 증가하는 Int32 메시지를 `/counter` 토픽으로 발행하는 노드를 만드세요

2. 위에서 만든 `/counter` 토픽을 구독하여 값을 제곱한 결과를 `/squared` 토픽으로 다시 발행하는 노드를 만드세요

3. 위 두 노드를 한 번에 실행하는 Launch 파일을 작성하세요

### 심화 문제

4. `/temperature`, `/humidity` 두 개의 토픽을 동시에 발행하는 노드를 만들고, 각각 1초, 2초 주기로 발행하세요

5. Python과 동일한 기능의 Publisher/Subscriber 노드를 C++로도 작성하고 빌드/실행하세요

6. Launch 파일에 `remappings`을 사용하여 토픽 이름을 변경하는 예제를 작성하세요

---

## Day 6 — 복습 & 트러블슈팅

### 기본 문제

1. RPi와 PC 사이에서 `ros2 doctor`를 양쪽에서 실행하고 결과를 비교하세요

2. RPi와 PC의 RMW를 `rmw_fastrtps_cpp`에서 `rmw_cyclonedds_cpp`로 변경하고 통신이 정상 동작하는지 확인하세요

3. ROS_DOMAIN_ID를 30과 50 두 개로 분리하여 각각 talker/listener를 실행하고, 도메인이 격리되는 것을 확인하세요

### 심화 문제

4. RPi에서 `ros2 run demo_nodes_py talker` 실행 중, PC에서 `ros2 topic echo /chatter`가 안 될 때 체계적으로 진단하는 절차를 순서도로 작성하세요

5. RPi의 메모리 상태(`free -h`), CPU 온도, 네트워크 latency를 10분간 모니터링하고 ROS2 통신 품질과의 상관관계를 분석하세요

6. WiFi 전원 관리 모드(`iwconfig wlan0 power off`)가 ROS2 통신 지연에 미치는 영향을 측정하세요

---

## Day 7 — 미니 프로젝트 (온도 모니터링)

### 기본 구현 확인

- [ ] `cpu_temp_publisher.py`가 정상 실행되고 `/cpu_temp` 토픽 발행
- [ ] `temp_monitor.py`가 정상 실행되고 온도 표시
- [ ] CSV 파일에 온도 데이터가 기록됨
- [ ] `rqt_plot /cpu_temp`로 그래프 표시

### 추가 도전

1. 온도가 60°C를 초과하면 `/temp_warning` (String 타입) 토픽을 별도로 발행하는 기능을 추가하세요

2. 5분, 30분, 1시간 평균 온도를 계산하여 별도 토픽으로 발행하는 노드를 추가하세요

3. 온도 데이터를 10초 간격으로 히스토그램으로 변환하여 출력하는 분석 노드를 만드세요

4. 온도의 변화율(dT/dt)을 계산하고, 급격한 변화(>2°C/s)를 감지하면 경고를 발행하세요

5. Launch 파일 하나로 publisher + monitor + rqt_plot을 동시에 실행하는 `temp_monitor.launch.py`를 작성하세요
