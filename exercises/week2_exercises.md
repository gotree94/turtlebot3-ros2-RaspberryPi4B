# Week 2 연습문제

> **범위:** Day 8 ~ Day 14 (TurtleBot3 구동 & 센서/제어)

---

## Day 8 — TurtleBot3 SBC 패키지 설치

### 기본 문제

1. `ros2 pkg list | grep turtlebot3`로 설치된 패키지 목록을 확인하고, 각 패키지의 역할을 조사하여 표로 정리하세요

2. `lsusb` 명령어로 OpenCR과 LDS가 USB 장치로 인식되는지 확인하고, 각 장치의 Vendor ID와 Product ID를 기록하세요

3. OpenCR 포트의 권한(`ls -la /dev/ttyACM*`)을 확인하고, 권한 문제가 있다면 해결 방법을 서술하세요

4. `udevadm info -a -n /dev/ttyACM0`로 OpenCR의 상세 정보를 출력하고, idVendor와 idProduct 값을 기록하세요

### 심화 문제

5. RPi의 CPU 온도(`cat /sys/class/thermal/thermal_zone0/temp`)를 빌드 전후로 측정하고, 차이를 기록하세요. TurtleBot3 패키지 빌드 시간도 측정하세요

6. `colcon build`에서 `--parallel-workers 1`과 `--parallel-workers 4`의 빌드 시간 차이를 측정하고, 온도 변화도 함께 기록하세요

---

## Day 9 — Remote PC TurtleBot3 패키지

### 기본 문제

1. Gazebo의 `turtlebot3_world.launch.py` 실행 후 `ros2 topic echo /scan`으로 LIDAR 데이터를 확인하고, 데이터 구조의 각 필드를 설명하세요

2. `rqt_plot`으로 `/odom`에서 선속도(linear.x)와 각속도(angular.z)를 실시간 그래프로 출력하고, teleop 조작에 따른 변화를 관찰하세요

3. 텔레옵 키보드로 TurtleBot3를 정사각형 경로로 주행시키고, `/odom`의 x, y 값 변화를 기록하세요

### 심화 문제

4. `turtlebot3_world.launch.py`와 `turtlebot3_house.launch.py`의 Gazebo 환경 차이점을 지도 구조, 장애물 배치 측면에서 비교 분석하세요

5. Gazebo에서 페이크 노드 없이 직접 `/cmd_vel`을 발행하여 로봇을 제어하고, Gazego 물리 엔진의 반응을 관찰하세요

---

## Day 10 — Bringup & Teleoperation

### 기본 문제

1. `/scan` 데이터에서 전방(0°)의 거리 값을 확인하고, 벽까지의 거리를 측정한 후 실제 거리와 비교하세요

2. 로봇을 1m 직진시키고, 실제 이동 거리와 odometry 값을 비교하여 오차율(%)을 계산하세요

3. 로봇을 360° 회전시키고, 실제 회전 각도와 odometry의 차이를 측정하세요

4. 텔레옵으로 5분간 연속 주행 후 배터리 전압 변화를 1분 간격으로 기록하세요

### 심화 문제

5. `ros2 topic pub -r 10 /cmd_vel ...`으로 0.1 m/s로 3초 직진 → 정지 2초 → 0.5 rad/s로 3초 회전 → 정지하는 자동 시퀀스를 bash 명령어로 작성하고, `/odom` 데이터로 실제 경로를 추정하세요

6. 로봇을 정면 벽에서 1m 떨어진 곳에 위치시키고, `/scan`의 전방 거리 값을 읽어 0.5m까지 천천히 접근한 후 정지하는 스크립트를 작성하세요

---

## Day 11 — 센서 데이터 & RViz2 시각화

### 기본 문제

1. RViz2의 `LaserScan` 데이터를 보고 방 안의 물체 5개의 거리를 추정하고, 실제 줄자로 측정한 값과의 오차를 기록하세요

2. 로봇을 정사각형(1m x 1m)으로 주행시킨 후, 시작점과 종점의 odometry 오차를 기록하세요

3. `tf2_echo base_footprint base_scan`으로 LIDAR와 로봇 중심 간의 변환 관계를 확인하고, x, y, z 오프셋 값을 기록하세요

### 심화 문제

4. 30초간의 센서 데이터를 bag 파일로 기록하고(`ros2 bag record -a -o test_bag`), `ros2 bag info`로 내용을 확인한 후 재생하세요

5. RViz2에 RobotModel + LaserScan + Odometry + Path + TF를 모두 표시한 설정을 `~/tb3_complete.rviz`로 저장하세요

6. bag 파일에서 `/scan` 데이터만 추출하여 Python으로 분석하고, 주변 장애물의 위치를 xy 좌표로 변환하는 스크립트를 작성하세요

---

## Day 12 — PID 제어 & 로봇 구동

### 기본 문제

1. TurtleBot3를 0.5m x 0.5m 정사각형 경로로 정확히 주행시키는 Python 스크립트를 작성하고, 종점 오차를 측정하세요

2. 좌회전과 우회전을 번갈아가며 ∞(8자) 패턴을 주행시키는 코드를 작성하세요

3. `kp` 값을 100, 500, 1000으로 변경하며 주행 안정성(진동, 오버슈트)의 차이를 관찰하고 기록하세요

### 심화 문제

4. (0,0)에서 (1.0, 1.5)까지 자동 주행하는 Go-to-Goal을 실행하고, 실제 경로를 RViz2의 Marker LINE_STRIP으로 시각화하세요

5. 로봇이 정지 상태일 때와 주행 중일 때 DYNAMIXEL 모터의 부하(load) 값을 `/joint_states`에서 확인하고 비교하세요

6. OpenCR의 PID 게인을 변경한 후, 동일 경로 주행의 정확도가 어떻게 달라지는지 정량적으로 측정하세요

---

## Day 13 — RViz2 심화 & URDF

### 기본 문제

1. TurtleBot3 Burger의 URDF에서 모든 link와 joint의 이름, parent-child 관계를 트리 다이어그램으로 그리세요

2. LIDAR 장애물 위치에 SPHERE 마커를 동적으로 표시하는 노드를 작성하세요

3. RViz2에 현재 배터리 전압을 TEXT_VIEW_FACING 마커로 표시하는 노드를 만드세요

### 심화 문제

4. Go-to-Goal 주행 시 이동 경로를 LINE_STRIP 마커로 실시간 표시하는 기능을 추가하세요

5. 마커의 색상을 로봇 속도에 따라 변경하세요 (정지=녹색, 이동=파란색, 급정지=빨간색)

6. TurtleBot3 Burger의 URDF를 수정하여 가상의 카메라 링크를 추가하고, RViz2에서 표시되는지 확인하세요

---

## Day 14 — 미니 프로젝트 (장애물 회피)

### 기본 구현 확인

- [ ] LIDAR 데이터 정상 수신 확인
- [ ] 전방 180° 3구역(좌/중앙/우) 분할 분석
- [ ] 0.3m 이내 장애물 감지 시 회피 동작
- [ ] 장애물 없으면 직진 (0.15 m/s)
- [ ] 상태 정보 로그 출력

### 추가 도전

1. `SAFE_DIST`를 0.5m, 0.3m, 0.15m로 변경하며 주행 패턴의 차이를 기록하고, 주어진 환경에 최적인 값을 선정하세요

2. 오른쪽 벽을 따라 주행하는 Wall Follower 모드를 추가하고, 좌회전만으로 복도를 빠져나올 수 있는지 테스트하세요

3. 주행 중 장애물 회피 횟수, 총 이동 거리, 평균 속도를 기록하는 분석 노드를 만드세요

4. Gazebo 시뮬레이션에서 먼저 테스트한 후, 실제 로봇에서 동일하게 동작하는지 비교하고 차이점을 분석하세요

5. 방 안에 5개 이상의 장애물을 배치하고, 로봇이 모든 장애물을 회피하며 2분 이상 주행할 수 있는지 테스트하고 한계를 기록하세요

6. 동적 장애물(사람이 천천히 움직이는)에 대한 반응 속도와 회피 성공률을 측정하세요
