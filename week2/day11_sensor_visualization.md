# Day 11 — 센서 데이터 이해 & RViz2 시각화

> **목표:** TurtleBot3의 모든 센서 데이터(LIDAR, IMU, Odometry, 배터리)를 이해하고 RViz2로 시각화한다.

---

## 11.1 센서 개요

TurtleBot3 Burger는 다음과 같은 센서 데이터를 제공합니다:

| 토픽 | 타입 | 내용 | 주기 |
|------|------|------|------|
| `/scan` | `sensor_msgs/LaserScan` | 360° LIDAR 거리 데이터 | ~5-10 Hz |
| `/odom` | `nav_msgs/Odometry` | 바퀴 엔코더 기반 위치/속도 추정 | ~30 Hz |
| `/imu` | `sensor_msgs/Imu` | 가속도, 각속도, 방향 (OpenCR 내장) | ~50 Hz |
| `/battery_state` | `sensor_msgs/BatteryState` | 배터리 전압, 잔량 | ~1 Hz |
| `/tf` | `tf2_msgs/TFMessage` | 좌표계 변환 트리 | 연속 |
| `/joint_states` | `sensor_msgs/JointState` | 모터 조인트 위치/속도 | ~30 Hz |

---

## 11.2 LIDAR 데이터 심층 분석

### 메시지 구조

```bash
ros2 interface show sensor_msgs/LaserScan
```

```
# Single scan from a planar laser range-finder
std_msgs/Header header
  uint32 seq
  time stamp
  string frame_id    # "base_scan"
float32 angle_min    # 시작 각도 [rad]
float32 angle_max    # 종료 각도 [rad]
float32 angle_increment  # 각도 해상도 [rad]
float32 time_increment   # 포인트 간 시간 [s]
float32 scan_time       # 전체 스캔 시간 [s]
float32 range_min       # 최소 감지 거리 [m]
float32 range_max       # 최대 감지 거리 [m]
float32[] ranges        # 거리 값 배열 [m]
float32[] intensities   # 반사 강도
```

### 각도 ↔ 배열 인덱스 변환

```
인덱스 0    = angle_min    = -π (-180°) → 오른쪽
인덱스 90   = 0°           → 정면 (LDS-01 기준)
인덱스 180  = +π/2 (90°)  → 왼쪽
인덱스 270  = +π (180°)   → 뒤
```

### 실시간 데이터 확인

```bash
# 전방 0° 데이터만 보기
ros2 topic echo /scan --field ranges | head -5

# 전방 거리만 모니터링 (Python one-liner)
python3 -c "
import rclpy
from rclpy.node import Node
from sensor_msgs.msg import LaserScan

class ScanMonitor(Node):
    def __init__(self):
        super().__init__('scan_monitor')
        self.sub = self.create_subscription(LaserScan, '/scan', self.cb, 10)
    def cb(self, msg):
        front = msg.ranges[len(msg.ranges)//2]  # 정면
        min_dist = min(msg.ranges)
        min_angle = msg.ranges.index(min_dist) * msg.angle_increment
        print(f'Front: {front:.3f}m | Min: {min_dist:.3f}m at {min_angle:.1f}rad')

rclpy.init()
node = ScanMonitor()
rclpy.spin(node)
"
```

---

## 11.3 Odometry (Odometry)

바퀴 엔코더로 추정한 로봇의 위치와 속도:

```bash
# 메시지 구조
ros2 interface show nav_msgs/Odometry
```

```
std_msgs/Header header
string child_frame_id    # "base_footprint"
geometry_msgs/PoseWithCovariance pose
  geometry_msgs/Pose pose
    geometry_msgs/Point position    # x, y, z (위치 [m])
    geometry_msgs/Quaternion orientation  # x, y, z, w (회전)
  float64[36] covariance
geometry_msgs/TwistWithCovariance twist
  geometry_msgs/Twist twist
    geometry_msgs/Vector3 linear    # 선속도 [m/s]
    geometry_msgs/Vector3 angular   # 각속도 [rad/s]
  float64[36] covariance
```

```bash
# 위치만 확인
ros2 topic echo /odom --field pose.pose.position
#   x: 0.123
#   y: 0.456
#   z: 0.000
```

### Odometry 한계 이해

- **단점:** 바퀴 슬립이 발생하면 오차 누적 (drift)
- **극복 방법:** LIDAR + IMU 융합 = SLAM
- **TurtleBot3 정밀도:** 약 1-3% 오차 (바닥 상태에 따라 다름)

---

## 11.4 IMU (Inertial Measurement Unit)

OpenCR 보드에 내장된 IMU 센서:

```bash
ros2 interface show sensor_msgs/Imu
```

```bash
# IMU 데이터 확인
ros2 topic echo /imu --once
```

**IMU 활용:**
- **가속도계:** 선형 가속도 측정 (중력 방향 포함)
- **자이로스코프:** 각속도 측정 (회전 속도)
- **자기장계(일부):** 지자기 방향

> 실제 TurtleBot3 Burger의 OpenCR IMU는 자이로 + 가속도계만 제공합니다.

---

## 11.5 TF (Transform) 트리

**TF**는 ROS2에서 모든 좌표계 간 변환 관계를 관리하는 시스템입니다.

### TurtleBot3 좌표계 트리

```
map ─► odom ─► base_footprint ─► base_link ─► base_scan
                                        ├──► left_wheel_link
                                        └──► right_wheel_link
```

| 프레임 | 설명 |
|--------|------|
| `map` | 전역 지도 좌표계 (SLAM 사용 시) |
| `odom` | Odometry 기준 좌표계 |
| `base_footprint` | 바닥 투영 기준 (x: 전방, y: 좌측, z: 상단) |
| `base_link` | 로봇 중심 좌표계 |
| `base_scan` | LIDAR 좌표계 |
| `left_wheel_link` | 좌측 바퀴 |
| `right_wheel_link` | 우측 바퀴 |

```bash
# TF 트리 확인
ros2 run tf2_tools view_frames.py

# 생성된 frames.pdf 확인
evince frames.pdf

# 특정 프레임 간 변환 확인
ros2 run tf2_ros tf2_echo base_link base_scan
```

---

## 11.6 RViz2 고급 시각화

### RViz2 런치 파일로 실행

```bash
# TurtleBot3 전용 RViz2 실행
ros2 launch turtlebot3_bringup rviz2.launch.py

# 또는 직접 실행
rviz2
```

### 추천 Display 설정

| Display | 설정 | 목적 |
|---------|------|------|
| `RobotModel` | Description Topic: `/robot_description` | 3D 로봇 모델 |
| `LaserScan` | Topic: `/scan`, Style: Points | LIDAR 데이터 |
| `Odometry` | Topic: `/odom`, Color: 녹색 | 위치 추정 경로 |
| `TF` | Frame Timeout: 15 | 좌표계 표시 |
| `Grid` | Plane Cell Count: 30 | 바닥 격자 |
| `Path` | Topic: `/odom` nav_msgs/Path | 이동 경로 |

### 카메라 뷰 (선택)

USB 카메라가 설치된 경우:

```bash
# RPi에서
ros2 run v4l2_camera v4l2_camera_node

# PC의 RViz2에서 Image display 추가 → Topic: /image_raw
```

---

## 11.7 rqt_plot으로 시각화

```bash
# 실시간 그래프
rqt_plot

# 좌측 상단 + 버튼 → Topic 입력:
# /odom/pose/pose/position/x
# /odom/pose/pose/position/y
# /odom/twist/twist/linear/x
# /scan/ranges[180]  (정면값)
```

### rqt_multiplot

더 정교한 그래프:

```bash
# 설치
sudo apt install -y ros-humble-rqt-multiplot

# 실행
ros2 run rqt_multiplot rqt_multiplot
```

---

## 11.8 ROS2 Bag으로 데이터 기록

```bash
# 모든 토픽 녹음 (5초간)
ros2 bag record -a -o test_bag &
sleep 5
kill %1

# 녹음된 파일 확인
ls -la test_bag/

# 정보 확인
ros2 bag info test_bag/

# 재생
ros2 bag play test_bag/
```

### 특정 토픽만 기록

```bash
# LIDAR, Odometry, TF만 기록
ros2 bag record /scan /odom /tf -o sensor_bag
```

---

## 📝 연습 문제

1. **거리 측정:** RViz2의 `LaserScan` 데이터를 보고 방 안의 물체 5개의 거리를 추정하고, 실제 줄자로 측정한 값과 비교하세요
2. **Odometry drift:** 로봇을 정사각형(1m x 1m)으로 주행시킨 후, 시작점과 종점의 odometry 오차를 기록하세요
3. **TF 실습:** `tf2_echo base_footprint base_scan`으로 LIDAR와 로봇 중심 간의 변환 관계를 확인하세요
4. **Bag 기록:** 30초간의 센서 데이터를 bag 파일로 기록하고, `ros2 bag info`로 내용을 확인한 후 재생하세요
5. **종합 시각화:** RViz2에 RobotModel + LaserScan + Odometry + Path + TF를 모두 표시한 설정을 저장하세요

---

## ⚠️ 트러블슈팅

| 문제 | 해결 방법 |
|------|----------|
| RViz2에 RobotModel 안 보임 | `RobotModel` display의 Description Topic이 `/robot_description`인지 확인 |
| LaserScan이 360°로 안 보임 | RVIZ의 `LaserScan` display에서 `Style`을 `Points`로 변경 |
| TF 오류 | `view_frames.py` 실행 후 PDF 확인, 누락된 프레임 식별 |
| Bag 재생 시 속도 이상 | `ros2 bag play --rate 0.5 test_bag/`로 느리게 재생 |
| rqt_plot 응답 없음 | `rqt --force-discover`로 새로고침 후 재시도 |
