# Day 16 — 실제 SLAM 매핑 실습

> **목표:** 실제 TurtleBot3로 slam_toolbox를 실행하여 환경 지도를 생성하고 저장한다.

---

## 16.1 준비 사항

```bash
# RPi: TurtleBot3 Bringup 실행
ssh ubuntu@turtlebot-pi.local
ros2 launch turtlebot3_bringup robot.launch.py

# PC: 필요한 패키지 확인
sudo apt install -y ros-humble-slam-toolbox
export TURTLEBOT3_MODEL=burger

# 환경 변수 확인
echo $ROS_DOMAIN_ID   # 30
```

---

## 16.2 SLAM 실행

### Step 1: 토픽 확인

```bash
# PC에서 Bringup 토픽 확인
ros2 topic list
# /scan, /odom, /tf 등이 보여야 함
# 없으면 RPi의 bringup이 정상인지 확인
```

### Step 2: SLAM Toolbox 실행

```bash
# PC에서 SLAM 실행
ros2 launch slam_toolbox online_async_launch.py \
  slam_params_file:=/opt/ros/humble/share/turtlebot3_navigation2/param/burger.yaml \
  use_sim_time:=false
```

**정상 실행 시 로그:**
```
[INFO] [slam_toolbox]: Configuration:
[INFO] [slam_toolbox]:   solver_plugin: ceres_linear_solver
[INFO] [slam_toolbox]:   map_file_name: map_name
[INFO] [slam_toolbox]:   do_loop_closing: true
[INFO] [slam_toolbox]: Started
```

### Step 3: RViz2 실행

```bash
# PC에서 RViz2 실행
rviz2
```

**RViz2 설정:**
1. Fixed Frame → `map`
2. Add → `Map` → Topic: `/map`
3. Add → `LaserScan` → Topic: `/scan`
4. Add → `RobotModel`
5. Add → `TF`

### Step 4: 텔레옵으로 매핑 시작

```bash
# PC에서 텔레옵 실행
ros2 run turtlebot3_teleop teleop_keyboard
```

---

## 16.3 효과적인 매핑 전략

### 주행 패턴

```
좋은 예:                     나쁜 예:
┌─────┬─────┐               ┌─────────┐
│     │     │               │         │
│  ┌──┘     │               │  급회전  │
│  │        │               │  급가속  │
│  └──┐     │               │  불규칙  │
│     │     │               │         │
└─────┴─────┘               └─────────┘

느리고 꾸준하게               급격한 움직임
왕복 패턴                    비효율적
```

### 실내 공간 매핑 순서

```
1. 방의 둘레를 따라 이동 (외곽선 탐색)
2. 내부를 지그재그로 이동
3. 중앙 영역 커버
4. 주요 지점 재방문 (Loop Closure)
5. 시작점으로 복귀
```

### 매핑 팁

| 팁 | 설명 |
|----|------|
| **천천히** | 0.1~0.15 m/s로 이동, 급회전 금지 |
| **일정 간격** | LIDAR 스캔이 겹치도록 천천히 |
| **Loop Closure** | 같은 공간 2번 이상 지나가기 |
| **가장자리** | 벽/가구의 경계를 따라 이동 |
| **재방문** | 특히 모서리/입구 재방문 |

---

## 16.4 지도 저장

### 방법 1: map_saver_cli 사용

```bash
# PC에서 (SLAM 실행 중)
ros2 run nav2_map_server map_saver_cli -f ~/my_map

# my_map.yaml + my_map.pgm 파일 생성됨
ls -la ~/my_map.*
```

### 방법 2: slam_toolbox의 serialize

```bash
# SLAM 실행 중인 PC에서
ros2 service call /slam_toolbox/serialize_map slam_toolbox/srv/SerializePoseGraph \
  "{filename: '$HOME/my_pose_graph'}"

# 로드할 때:
ros2 service call /slam_toolbox/deserialize_map slam_toolbox/srv/DeserializePoseGraph \
  "{filename: '$HOME/my_pose_graph'}"
```

---

## 16.5 저장된 지도 파일 이해

### YAML 파일

```yaml
# ~/my_map.yaml
image: my_map.pgm       # 이미지 파일명
resolution: 0.05        # 1픽셀당 5cm (낮을수록 상세)
origin: [-10.0, -10.0, 0.0]  # 지도 원점 (x, y, yaw)
negate: 0               # 반전 (0=정상)
occupied_thresh: 0.65   # 장애물 임계값
free_thresh: 0.25       # 빈 공간 임계값
```

### PGM 파일

```
P5                          # 그레이스케일 포맷
# CREATOR: map_saver.cpp     # 주석
800 800                     # 너비 x 높이 (픽셀)
255                         # 최대값 (255=흰색)
... binary image data ...   # 0=검정(장애물), 205=회색(미탐색), 254=흰색(빈공간)
```

### 픽셀 값 의미

| 값 | 색상 | 의미 |
|----|------|------|
| 0 | 검정 | 장애물 (occupied) |
| 205 | 회색 | 미탐색 영역 (unknown) |
| 254 | 흰색 | 빈 공간 (free) |

---

## 16.6 지도 시각화

```bash
# 저장된 지도를 RViz2에 표시
rviz2

# Add → Map → Topic: /map (SLAM 실행 중에는 실시간)
# 저장된 지도는 이미지 뷰어로 확인
display ~/my_map.pgm
```

---

## 16.7 매핑 품질 평가

### 좋은 지도의 특징

```
✅ 벽이 뚜렷하고 일직선 (흐릿함 없음)
✅ 가구/장애물의 윤곽이 선명함
✅ Loop Closure로 인한 이중선 없음
✅ 미탐색 영역(회색)이 최소화
✅ 지도의 규모가 실제와 일치
```

### 나쁜 지도의 특징

```
❌ 벽이 2중으로 보임 (odometry drift)
❌ 전체적으로 흐릿함 (너무 빠른 주행)
❌ 특정 영역이 심하게 왜곡됨
❌ 미탐색 영역이 너무 넓음
❌ 실제와 축척이 맞지 않음
```

---

## 16.8 매핑 실패 시 대처

### 동일 경로 재매핑

```bash
# SLAM 중단
# Ctrl+C로 slam_toolbox 종료

# 기존 지도 삭제
rm -f ~/my_map.*

# SLAM 재실행
ros2 launch slam_toolbox online_async_launch.py \
  slam_params_file:=/opt/ros/humble/share/turtlebot3_navigation2/param/burger.yaml \
  use_sim_time:=false
```

### Bag 파일로 오프라인 매핑

```bash
# 1. 데이터 녹음
ros2 bag record /scan /odom /tf /imu -o mapping_session

# 2. 오프라인 SLAM 실행
ros2 launch slam_toolbox offline_launch.py \
  slam_params_file:=burger.yaml \
  bag_file:=mapping_session
```

---

## 📝 연습 문제

1. **매핑 실습:** 5평 크기의 방을 SLAM으로 매핑하고, 지도를 저장하세요. 소요 시간과 정확도를 기록하세요
2. **Loop Closure:** 매핑 시작점으로 돌아오지 않은 경우와 돌아온 경우의 지도 품질 차이를 비교하세요
3. **속도 비교:** 0.05 m/s, 0.1 m/s, 0.2 m/s로 각각 매핑하고 지도 품질의 차이를 분석하세요
4. **지도 분석:** 생성된 PGM 파일을 이미지 뷰어로 열고, 각 영역의 픽셀 값을 확인하세요
5. **Clearance:** 저장된 YAML 파일의 파라미터를 수정하여 occupied_thresh와 free_thresh의 효과를 테스트하세요

---

## ⚠️ 트러블슈팅

| 문제 | 해결 방법 |
|------|----------|
| SLAM 시작 시 "No map received" | /scan, /tf, /odom 토픽 발행 확인 |
| 지도가 자꾸 흔들림 | `minimum_travel_distance`를 0.05로 낮춤 |
| Loop Closure가 너무 느림 | `loop_search_maximum_distance`를 5.0으로 증가 |
| 저장한 지도가 검은 화면 | YAML의 `origin` 값이 잘못되었을 수 있음 |
| map_saver_cli 명령어 없음 | `sudo apt install ros-humble-nav2-map-server` |
