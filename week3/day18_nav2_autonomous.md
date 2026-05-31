# Day 18 — Nav2 자율주행 실습

> **목표:** 실제 TurtleBot3에서 Navigation2를 실행하여 자율주행을 구현하고, 파라미터를 튜닝한다.

---

## 18.1 실행 순서

### Step 1: RPi — Bringup

```bash
# RPi SSH
ssh ubuntu@turtlebot-pi.local
export TURTLEBOT3_MODEL=burger
ros2 launch turtlebot3_bringup robot.launch.py
```

### Step 2: PC — SLAM 또는 지도 로드

**옵션 A: 새로 매핑 (SLAM 실행)**

```bash
# PC
ros2 launch slam_toolbox online_async_launch.py \
  slam_params_file:=/opt/ros/humble/share/turtlebot3_navigation2/param/burger.yaml \
  use_sim_time:=false
```

**옵션 B: 저장된 지도 로드**

```bash
# PC (지도를 미리 저장했다면)
export TURTLEBOT3_MODEL=burger
ros2 launch turtlebot3_navigation2 navigation2.launch.py \
  map:=/home/ubuntu/my_map.yaml
```

> ⚠️ **주의:** SLAM과 Nav2는 동시에 실행할 수 없습니다. 매핑이 먼저, 네비게이션은 그 후에!

### Step 3: PC — RViz2

```bash
# PC (navigation2.launch.py로 이미 실행되었다면 생략)
rviz2
```

### Step 4: 초기 위치 설정

1. RViz2에서 **"2D Pose Estimate"** 버튼 클릭
2. 지도 위에서 로봇의 대략적인 위치 클릭
3. 방향 화살표 드래그 (로봇이 바라보는 방향)

> **AMCL 파티클 관찰:** 화살표 형태의 파티클 분포 확인
> - 넓게 퍼짐 → 위치 불확실
> - 좁게 모임 → 위치 확실

### Step 5: 목적지 설정

1. RViz2에서 **"2D Nav Goal"** 버튼 클릭
2. 목표 지점 클릭
3. 로봇이 도착했을 때의 방향 드래그

---

## 18.2 전체 실행 스크립트

모든 터미널을 관리하는 스크립트 (`nav2_start.sh`):

```bash
#!/bin/bash
# TurtleBot3 Navigation2 실행 스크립트
# PC에서 실행

# 환경 변수
export TURTLEBOT3_MODEL=burger
export ROS_DOMAIN_ID=30

# 기존 세션 정리
echo "🧹 Cleaning up..."
killall -9 gzserver gzclient 2>/dev/null
killall -9 ros2 2>/dev/null
sleep 2

# Nav2 실행
echo "🚀 Starting Navigation2..."
ros2 launch turtlebot3_navigation2 navigation2.launch.py \
  map:=~/my_map.yaml \
  use_sim_time:=false
```

---

## 18.3 자율주행 관찰 포인트

### 주행 단계별 확인

```
1. 목적지 설정
   → RViz2에 전역 경로 (초록색 선) 표시
   
2. 경로 추종 시작
   → 로봇이 전역 경로를 따라 이동
   → Local Costmap에 동적 장애물 반영

3. 장애물 회피
   → Local Planner가 경로 재계획
   → Local Costmap이 실시간 업데이트

4. 목적지 도착
   → 로봇 정지
   → "Goal reached!" 메시지 출력
```

### 토픽 모니터링

```bash
# 전역 경로 확인
ros2 topic echo /plan --once

# Costmap 확인 (RViz2에서 시각적으로)
# Global Costmap: /global_costmap/costmap
# Local Costmap: /local_costmap/costmap

# 현재 속도 확인
ros2 topic echo /cmd_vel --once

# 네비게이션 상태
ros2 topic echo /bt_navigator/transition_event
```

---

## 18.4 Nav2 파라미터 튜닝

### 주요 튜닝 파라미터

burger.yaml의 중요한 파라미터들:

```yaml
# ============================================
# 1. 로봇 속도 관련
# ============================================
controller_server:
  ros__parameters:
    # DWB Controller 설정
    min_vel_x: -0.05           # 최소 후진 속도
    max_vel_x: 0.22            # 최대 전진 속도 (Burger 한계)
    min_vel_theta: -1.0        # 최소 회전 속도
    max_vel_theta: 2.0         # 최대 회전 속도
    min_speed_xy: 0.0          # 최소 xy 속도
    max_speed_xy: 0.22         # 최대 xy 속도
    
    # 가속도
    acc_lim_x: 2.5             # 선가속도
    acc_lim_y: 0.0             # 횡가속도 (Burger 미지원)
    acc_lim_theta: 3.2         # 각가속도
    decel_lim_x: -2.5          # 감속도

# ============================================
# 2. Costmap 관련
# ============================================
local_costmap:
  local_costmap:
    ros__parameters:
      # 로컬 코스트맵 크기 (로봇 주변)
      width: 3                 # 3m x 3m 영역
      height: 3
      resolution: 0.05         # 5cm 해상도

      # Obstacle Layer
      observation_sources: scan
      scan:
        topic: /scan
        max_obstacle_height: 0.5
        clearing: true
        marking: true

      # Inflation Layer
      inflation_layer:
        inflation_radius: 0.3  # 안전 여유 공간

global_costmap:
  global_costmap:
    ros__parameters:
      # 글로벌 코스트맵 (지도 전체 사용)
      width: 10
      height: 10
      resolution: 0.05

# ============================================
# 3. AMCL 관련
# ============================================
amcl:
  ros__parameters:
    min_particles: 6
    max_particles: 2000
    particles: 500
    
    # 위치 추정 불확실성
    alpha1: 0.2
    alpha2: 0.2
    alpha3: 0.2
    alpha4: 0.2
    
    # 초기 위치
    initial_pose_x: 0.0
    initial_pose_y: 0.0
    initial_pose_yaw: 0.0
```

### 튜닝 순서

```
Step 1: 속도 튜닝
  max_vel_x, max_vel_theta 조정
  → 너무 빠르면 충돌 위험, 너무 느리면 비효율

Step 2: Costmap 튜닝
  inflation_radius 조정
  → 너무 크면 좁은 통로 못 지나감
  → 너무 작으면 벽에 부딪힘

Step 3: AMCL 튜닝
  alpha1~4 조정
  → odometry 불확실성이 크면 값 증가

Step 4: Recovery 튜닝
  Spin, Backup, ClearCostmap 행동 확인
```

---

## 18.5 동적 장애물 테스트

```python
#!/usr/bin/env python3
"""
Nav2 자율주행 중 동적 장애물 회피 테스트
로봇이 주행 중일 때 장애물을 갑자기 경로 위에 놓고 반응 관찰
"""

import rclpy
from rclpy.node import Node
from geometry_msgs.msg import Twist
import time


class DynamicObstacleTest(Node):
    """자율주행 중 동적 장애물 테스트"""

    def __init__(self):
        super().__init__('dynamic_obstacle_test')
        self.cmd_pub = self.create_publisher(Twist, '/cmd_vel', 10)
        self.get_logger().info('=== Nav2 Dynamic Obstacle Test ===')
        self.get_logger().info('1. Set a Nav Goal in RViz2')
        self.get_logger().info('2. Wait for robot to start moving')
        self.get_logger().info('3. Place obstacle in robot path')
        self.get_logger().info('4. Observe: does robot avoid it?')
        self.get_logger().info('5. Robot should replan path')

    def test_manual_intervention(self):
        """자율주행 중 수동 개입"""
        self.get_logger().info('Testing manual interruption...')
        time.sleep(3)

        # 자율주행 중 갑자기 cmd_vel 발행
        cmd = Twist()
        cmd.angular.z = 1.0
        self.cmd_pub.publish(cmd)
        time.sleep(1)

        cmd = Twist()  # 정지
        self.cmd_pub.publish(cmd)
        self.get_logger().info('Check if Nav2 recovers after interruption')


def main():
    rclpy.init()
    test = DynamicObstacleTest()
    test.test_manual_intervention()
    test.destroy_node()
    rclpy.shutdown()
```

---

## 18.6 복구 동작 (Recovery Behaviors)

Nav2는 문제 발생 시 자동 복구를 시도합니다:

| 복구 동작 | 설명 | 트리거 조건 |
|-----------|------|------------|
| **Spin** | 제자리 360° 회전 | 경로 막힘, costmap 재평가 |
| **Backup** | 후진 | 경로 재시도 |
| **Clear Costmap** | costmap 초기화 | 오래된 장애물 제거 |
| **경로 재계획** | Global Planner 재실행 | 위 복구 실패 시 |

```bash
# 복구 동작 모니터링
ros2 topic echo /bt_navigator/transition_event
```

---

## 18.7 Waypoint Navigation

여러 목적지를 순차적으로 방문:

```bash
# Waypoint Following 패키지
sudo apt install -y ros-humble-nav2-waypoint-follower

# waypoint 파일 형식 (YAML)
cat > ~/waypoints.yaml << 'EOF'
waypoints:
  - position: {x: 1.0, y: 0.0, z: 0.0}
    orientation: {x: 0.0, y: 0.0, z: 0.0, w: 1.0}
  - position: {x: 1.0, y: 1.0, z: 0.0}
    orientation: {x: 0.0, y: 0.0, z: 0.707, w: 0.707}
  - position: {x: 0.0, y: 1.0, z: 0.0}
    orientation: {x: 0.0, y: 0.0, z: 1.0, w: 0.0}
EOF
```

---

## 📝 연습 문제

1. **자율주행 기본:** RViz2에서 3개의 다른 목적지를 설정하고, 각각의 경로 길이와 소요 시간을 기록하세요
2. **장애물 회피:** 자율주행 중 로봇 경로 위에 상자를 갑자기 놓고, 로봇의 반응을 관찰하고 기록하세요
3. **속도 튜닝:** `max_vel_x`를 0.1, 0.15, 0.22로 변경하면서 주행의 안정성과 속도를 비교하세요
4. **Inflation 튜닝:** `inflation_radius`를 0.1, 0.3, 0.5로 변경하고 좁은 통로 통과 여부를 테스트하세요
5. **AMCL 리셋:** 로봇을 갑자기 다른 위치로 옮긴 후(kidnapped), AMCL이 다시 위치를 찾는 시간과 과정을 관찰하세요

---

## ⚠️ 트러블슈팅

| 문제 | 해결 방법 |
|------|----------|
| 로봇이 목적지로 가지 않음 | "2D Pose Estimate"로 초기 위치 정확히 설정 |
| 경로 중간에 멈춤 | `inflation_radius`가 너무 큰지 확인 |
| 빙글빙글 돔 | `max_vel_theta` 감소, `acc_lim_theta` 증가 |
| AMCL 파티클이 수렴 안 됨 | `min_particles`와 `max_particles` 증가 |
| Costmap이 갑자기 빨갛게 변함 | 장애물이 LIDAR에 너무 가까이 있음. 로봇 주변 정리 |
| Nav2 "Received invalid goal" | 목적지가 Costmap의 lethal 영역 내에 있음 |
| `navigation2.launch.py`에 지도 경로 오류 | `map:=` 파라미터에 절대 경로 사용 |
