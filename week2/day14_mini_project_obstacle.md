# Day 14 — 미니 프로젝트: LIDAR 기반 장애물 회피

> **목표:** 2주차 동안 배운 모든 지식을 활용하여 LIDAR 기반 장애물 회피 주행 로봇을 구현한다.

---

## 14.1 프로젝트 개요

TurtleBot3 Burger의 LIDAR(`/scan`) 데이터를 실시간으로 분석하여, 장애물을 피하면서 무작위로 환경을 탐험하는 자율 주행 로봇을 만듭니다.

```
                     ┌─────────────────────┐
                     │   /scan (LIDAR)      │
                     │   거리 데이터 360°    │
                     └──────────┬──────────┘
                                │
                    ┌───────────▼───────────┐
                    │  Obstacle Avoidance   │
                    │  Node                 │
                    │                       │
                    │  1. 전방 스캔 분석     │
                    │  2. 장애물 거리 계산   │
                    │  3. 회피 방향 결정     │
                    │  4. cmd_vel 발행       │
                    └───────────┬───────────┘
                                │
                    ┌───────────▼───────────┐
                    │   /cmd_vel (Twist)    │
                    │   로봇 속도 명령       │
                    └───────────────────────┘
```

---

## 14.2 프로젝트 요구사항

### 필수 기능

1. ✅ 전방 180° 범위의 LIDAR 데이터를 실시간 분석
2. ✅ 가장 가까운 장애물까지의 거리 측정
3. ✅ 장애물이 임계 거리(0.3m) 이내이면 회피 동작
4. ✅ 장애물이 없으면 직진 (목표 속도 0.15 m/s)
5. ✅ 상태 정보를 로그로 출력

### 선택 기능

1. 여러 방향(전방/좌측/우측)의 거리 비교 → 최적 방향으로 회전
2. RViz2에 Markers로 장애물 위치 표시
3. odometry 기반 무작위 방향 전환 (일정 시간마다)

---

## 14.3 구현: 장애물 회피 노드

`~/ros2_ws/src/tb3_control/tb3_control/obstacle_avoidance.py`:

```python
#!/usr/bin/env python3
"""
TurtleBot3 LIDAR 기반 장애물 회피 주행 노드

동작 원리:
1. /scan 토픽에서 LIDAR 데이터 수신
2. 전방 180°를 3개 구역(좌/중앙/우)으로 분할
3. 각 구역의 최소 거리 계산
4. 가장 안전한 방향으로 회전 + 직진
"""

import rclpy
from rclpy.node import Node
from sensor_msgs.msg import LaserScan
from geometry_msgs.msg import Twist
from visualization_msgs.msg import Marker
import math


class ObstacleAvoidance(Node):
    """장애물 회피 주행 노드"""

    # 상수
    FRONT_ANGLE = 30  # 정면 판단 각도 (±30°)
    SIDE_ANGLE = 60   # 측면 판단 각도 (30°~90°)
    SAFE_DIST = 0.35  # 안전 거리 [m]
    DANGER_DIST = 0.2 # 위험 거리 [m]
    FORWARD_SPEED = 0.15
    TURN_SPEED = 0.5
    SCAN_TIMEOUT = 0.5  # 스캔 타임아웃 [s]

    def __init__(self):
        super().__init__('obstacle_avoidance')

        # Publisher & Subscriber
        self.cmd_pub = self.create_publisher(Twist, '/cmd_vel', 10)
        self.marker_pub = self.create_publisher(Marker, '/obstacle_marker', 10)
        self.scan_sub = self.create_subscription(
            LaserScan, '/scan', self.scan_callback, 10)

        # 상태 변수
        self.last_scan_time = self.get_clock().now()
        self.current_scan = None
        self.state = 'FORWARD'
        self.turn_direction = 0  # -1: left, 1: right

        # 타이머 (제어 루프 20Hz)
        self.control_timer = self.create_timer(0.05, self.control_loop)

        # 로그
        self.get_logger().info('🛡️  Obstacle Avoidance Node Started!')
        self.get_logger().info(
            f'Safe Dist: {self.SAFE_DIST}m, Danger Dist: {self.DANGER_DIST}m'
        )

    def scan_callback(self, msg):
        """LIDAR 데이터 수신"""
        self.current_scan = msg
        self.last_scan_time = self.get_clock().now()

    def get_range_data(self):
        """스캔 데이터에서 구역별 최소 거리 계산"""
        if self.current_scan is None:
            return None, None, None

        scan = self.current_scan
        total_points = len(scan.ranges)
        center_idx = total_points // 2

        # 각도 증분 계산
        angle_inc = scan.angle_increment
        front_range_deg = int(self.FRONT_ANGLE / math.degrees(angle_inc))
        side_range_deg = int(self.SIDE_ANGLE / math.degrees(angle_inc))

        def safe_range(r):
            """inf 또는 nan을 안전값으로 변환"""
            if math.isinf(r) or math.isnan(r):
                return scan.range_max
            return r

        # 정면 (중앙 ± FRONT_ANGLE)
        front_start = max(0, center_idx - front_range_deg)
        front_end = min(total_points, center_idx + front_range_deg)
        front_ranges = [safe_range(r) for r in scan.ranges[front_start:front_end]]
        front_min = min(front_ranges) if front_ranges else scan.range_max

        # 좌측 (중앙 + FRONT_ANGLE ~ +FRONT_ANGLE+SIDE_ANGLE)
        left_start = min(total_points, center_idx + front_range_deg)
        left_end = min(total_points, center_idx + front_range_deg + side_range_deg)
        left_ranges = [safe_range(r) for r in scan.ranges[left_start:left_end]]
        left_min = min(left_ranges) if left_ranges else scan.range_max

        # 우측 (중앙 - FRONT_ANGLE-SIDE_ANGLE ~ -FRONT_ANGLE)
        right_end = max(0, center_idx - front_range_deg)
        right_start = max(0, center_idx - front_range_deg - side_range_deg)
        right_ranges = [safe_range(r) for r in scan.ranges[right_start:right_end]]
        right_min = min(right_ranges) if right_ranges else scan.range_max

        return front_min, left_min, right_min

    def control_loop(self):
        """메인 제어 루프"""
        # 스캔 타임아웃 체크
        elapsed = (self.get_clock().now() - self.last_scan_time).nanoseconds / 1e9
        if elapsed > self.SCAN_TIMEOUT:
            self.get_logger().warn('LIDAR scan timeout! Stopping.')
            self.stop_robot()
            return

        front_dist, left_dist, right_dist = self.get_range_data()
        if front_dist is None:
            return

        cmd = Twist()

        # 상태 머신
        if front_dist < self.DANGER_DIST:
            # 위험: 급회전
            self.state = 'DANGER'
            cmd.linear.x = 0.0

            # 더 넓은 쪽으로 회전
            if left_dist > right_dist:
                cmd.angular.z = self.TURN_SPEED * 1.5
                self.get_logger().warn(f'🔥 DANGER! Front={front_dist:.2f}m, Hard LEFT')
            else:
                cmd.angular.z = -self.TURN_SPEED * 1.5
                self.get_logger().warn(f'🔥 DANGER! Front={front_dist:.2f}m, Hard RIGHT')

        elif front_dist < self.SAFE_DIST:
            # 안전 거리 이내: 회피
            self.state = 'AVOID'
            cmd.linear.x = 0.05  # 천천히 전진

            if left_dist >= right_dist:
                cmd.angular.z = self.TURN_SPEED
                self.get_logger().info(f'⚠️  AVOID: Front={front_dist:.2f}m, Turn LEFT')
            else:
                cmd.angular.z = -self.TURN_SPEED
                self.get_logger().info(f'⚠️  AVOID: Front={front_dist:.2f}m, Turn RIGHT')

        else:
            # 안전: 직진
            self.state = 'FORWARD'
            cmd.linear.x = self.FORWARD_SPEED
            cmd.angular.z = 0.0

            # 일정 시간마다 무작위 방향 전환 (탐험 모드)
            if self.get_clock().now().nanoseconds % 5000000000 < 50000000:
                import random
                cmd.angular.z = random.uniform(-0.3, 0.3)
                self.get_logger().debug('Random direction change')

        # cmd_vel 발행
        self.cmd_pub.publish(cmd)

        # 마커 발행 (장애물 위치 표시)
        self.publish_obstacle_marker(front_dist, left_dist, right_dist)

    def publish_obstacle_marker(self, front, left, right):
        """RViz2에 장애물 마커 표시"""
        marker = Marker()
        marker.header.frame_id = 'base_scan'
        marker.header.stamp = self.get_clock().now().to_msg()
        marker.ns = 'obstacles'
        marker.id = 0
        marker.type = Marker.SPHERE_LIST
        marker.action = Marker.ADD

        marker.scale.x = 0.05
        marker.scale.y = 0.05
        marker.scale.z = 0.05
        marker.color.a = 0.8
        marker.color.g = 1.0

        # 정면 장애물
        from geometry_msgs.msg import Point
        if front < self.SAFE_DIST:
            p = Point()
            p.x = front
            p.y = 0.0
            p.z = 0.0
            marker.points.append(p)
            marker.color.r = 1.0  # 빨간색

        self.marker_pub.publish(marker)

    def stop_robot(self):
        """비상 정지"""
        cmd = Twist()
        self.cmd_pub.publish(cmd)
        self.get_logger().warn('🛑 EMERGENCY STOP')

    def destroy_node(self):
        self.stop_robot()
        super().destroy_node()


def main(args=None):
    rclpy.init(args=args)
    node = ObstacleAvoidance()
    try:
        rclpy.spin(node)
    except KeyboardInterrupt:
        node.get_logger().info('🛑 Shutting down...')
    finally:
        node.destroy_node()
        rclpy.shutdown()


if __name__ == '__main__':
    main()
```

---

## 14.4 패키지 설정

`package.xml`에 의존성 추가:

```xml
<depend>rclpy</depend>
<depend>std_msgs</depend>
<depend>sensor_msgs</depend>
<depend>geometry_msgs</depend>
<depend>visualization_msgs</depend>
```

`setup.py` entry_points:

```python
entry_points={
    'console_scripts': [
        'obstacle_avoidance = tb3_control.obstacle_avoidance:main',
        'precision_drive = tb3_control.precision_drive:main',
        'go_to_goal = tb3_control.go_to_goal:main',
    ],
},
```

---

## 14.5 실행 방법

### PC에서

```bash
# 1. 빌드
cd ~/ros2_ws
colcon build --symlink-install
source install/setup.bash

# 2. RPi에서 로봇 Bringup (SSH)
ssh ubuntu@turtlebot-pi.local
ros2 launch turtlebot3_bringup robot.launch.py

# 3. PC에서 장애물 회피 노드 실행
ros2 run tb3_control obstacle_avoidance

# 4. (선택) RViz2로 모니터링
rviz2
```

### 시뮬레이션으로 테스트

```bash
# PC에서 Gazebo 실행 (로봇 없을 때)
ros2 launch turtlebot3_gazebo turtlebot3_world.launch.py

# 장애물 회피 노드 실행
ros2 run tb3_control obstacle_avoidance
```

---

## 14.6 성능 튜닝 가이드

### 파라미터 최적화

| 파라미터 | 기본값 | 영향 | 튜닝 방향 |
|---------|--------|------|----------|
| `SAFE_DIST` | 0.35m | 회피 시작 거리 | 좁은 공간: 0.25, 넓은 공간: 0.5 |
| `DANGER_DIST` | 0.2m | 긴급 회피 거리 | 기본 유지 |
| `FORWARD_SPEED` | 0.15 m/s | 직진 속도 | 초보: 0.1, 숙련: 0.2 |
| `TURN_SPEED` | 0.5 rad/s | 회전 속도 | 천천히: 0.3, 빠르게: 0.8 |
| `FRONT_ANGLE` | 30° | 정면 인식 각도 | 넓게: 45, 좁게: 15 |

### 시나리오별 설정

```python
# 좁은 복도 주행
config = {
    'SAFE_DIST': 0.25,
    'FORWARD_SPEED': 0.1,
    'FRONT_ANGLE': 20,
}

# 넓은 공간 탐험
config = {
    'SAFE_DIST': 0.5,
    'FORWARD_SPEED': 0.2,
    'FRONT_ANGLE': 45,
}
```

---

## 14.7 확장 아이디어

### 1) 벽 따라가기 (Wall Follower)

```python
# 오른쪽 벽을 따라가는 로직
if right_dist < TARGET_DIST:
    cmd.angular.z = -0.2  # 왼쪽으로
elif right_dist > TARGET_DIST:
    cmd.angular.z = 0.2   # 오른쪽으로
```

### 2) 다중 센서 융합

```python
# IMU + Odometry + LIDAR 융합
# IMU로 회전 각도 보정, odometry로 위치 추정
```

### 3) 탐험 완료 판단

```python
# 일정 시간 동안 새 영역을 발견하지 못하면 종료
unvisited_coverage = estimate_coverage(odom_poses)
if unvisited_coverage > 0.95:
    self.get_logger().info('✅ Exploration complete!')
    self.stop_robot()
```

### 4) 동적 장애물 대응

```python
# 이전 스캔과 비교하여 움직이는 물체 감지
diff = current_scan - previous_scan
moving_objects = diff > MOVEMENT_THRESHOLD
```

---

## 14.8 테스트 시나리오

### 시나리오 1: 단일 장애물

```
환경: 로봇 정면 0.5m에 상자 하나
예상 동작: 장애물 감지 → 좌/우 회전 → 회피 → 직진 재개
```

### 시나리오 2: 복도 주행

```
환경: 양쪽 벽 사이 (폭 1m)
예상 동작: 직진 중 양쪽 벽 감지 → 중앙 유지 → 전방 막히면 회전
```

### 시나리오 3: 막다른 골목

```
환경: 3면이 벽으로 둘러싸인 공간
예상 동작: 전방 위험 → 회전 → 좌우도 위험 → 180° 회전 → 탈출
```

---

## 📝 연습 문제

1. **파라미터 튜닝:** `SAFE_DIST`를 0.5m, 0.3m, 0.15m로 변경하며 주행 패턴의 차이를 기록하고, 최적값을 찾으세요
2. **벽 따라가기:** 오른쪽 벽을 따라 주행하는 Wall Follower 모드를 추가하세요
3. **통계 기록:** 주행 중 장애물 회피 횟수, 총 이동 거리, 평균 속도를 기록하는 분석 노드를 만드세요
4. **시뮬레이션 vs 실제:** Gazebo 시뮬레이션에서 먼저 테스트한 후, 실제 로봇에서 동일하게 동작하는지 비교하세요
5. **다중 장애물:** 방 안에 5개 이상의 장애물을 배치하고, 로봇이 모든 장애물을 회피하며 2분 이상 주행할 수 있는지 테스트하세요

---

## 🎉 Week 2 완료!

축하합니다! 14일 동안 ROS2의 핵심 개념부터 실제 TurtleBot3 제어까지 마스터했습니다.

**Week 2에서 배운 것:**
- ✅ TurtleBot3 패키지 설치 및 설정
- ✅ Bringup 및 Teleoperation
- ✅ LIDAR, Odometry, IMU, TF 데이터 이해
- ✅ RViz2 시각화
- ✅ PID 제어 및 Go-to-Goal
- ✅ URDF 구조 이해
- ✅ 미니 프로젝트: LIDAR 장애물 회피

**이제 Week 3로 넘어가서 SLAM과 자율주행에 도전합시다! 🚀**
