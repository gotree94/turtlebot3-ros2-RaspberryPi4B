# Day 12 — PID 제어 & 로봇 구동 원리

> **목표:** TurtleBot3 Burger의 DYNAMIXEL 모터 구동 원리를 이해하고, 직접 cmd_vel 명령으로 주행 제어를 실습한다.

---

## 12.1 DYNAMIXEL XL430-W250 모터

TurtleBot3 Burger는 2개의 **DYNAMIXEL XL430-W250** 서보 모터를 사용합니다.

### 사양

| 항목 | 값 |
|------|-----|
| 모델명 | XL430-W250 |
| 통신 프로토콜 | Dynamixel Protocol 2.0 |
| 해상도 | 4096 (12-bit) |
| 동작 모드 | 속도 제어(Velocity Control), 위치 제어(Position Control) |
| 최대 토크 | 1.0 N·m |
| 최대 속도 | 77 rpm (약 0.22 m/s, Burger 기준) |
| 피드백 | 위치, 속도, 부하, 온도, 전압 |

### Differential Drive (차동 구동)

```
      좌측 바퀴 (L)
        ▲            ▲ 우측 바퀴 (R)
        │            │
        └──────┬─────┘
               │
            본체 (OpenCR)
```

**속도 관계:**
- 직진: L = R (같은 속도, 같은 방향)
- 회전: L = -R (반대 방향)
- 선회: L ≠ R (서로 다른 속도)

---

## 12.2 cmd_vel 상세

```bash
geometry_msgs/Twist 메시지:
  linear:
    x: 0.22  # 전진 속도 [m/s] (최대)
    y: 0.0   # 횡방향 (Burger는 미지원)
    z: 0.0
  angular:
    x: 0.0
    y: 0.0
    z: 2.84  # 회전 속도 [rad/s] (최대)
```

### 속도 ↔ 모터 RPM 변환

```
TurtleBot3 Burger 제원:
  바퀴 반경 (r): 0.033 m
  바퀴 간격 (w): 0.160 m

  모터 속도 → cmd_vel:
    linear.x = (RPM_L + RPM_R) / 2 * r * 2π / 60
  
  cmd_vel → 모터 속도:
    RPM_L = (linear.x - angular.z * w / 2) / r * 60 / (2π)
    RPM_R = (linear.x + angular.z * w / 2) / r * 60 / (2π)
```

---

## 12.3 PID 제어 원리

OpenCR은 내부적으로 DYNAMIXEL 모터에 PID 제어를 적용합니다.

```
목표 속도 ──► [오차] ──► P: Kp * e(t) ──┐
                    │                     ├──► [합산] ──► 모터 PWM
                    └──► I: Ki * ∫e(t)dt ─┘
                    └──► D: Kd * de(t)/dt ─┘
                        ▲
                        │
                   센서 피드백 (엔코더)
```

### PID 게인 확인

```bash
# OpenCR의 PID 파라미터 확인
ros2 param get /turtlebot3_core kp
ros2 param get /turtlebot3_core ki
ros2 param get /turtlebot3_core kd
```

---

## 12.4 직접 cmd_vel 제어 실습

### Python으로 정밀 주행 제어

PC에 저장: `~/ros2_ws/src/tb3_control/tb3_control/precision_drive.py`:

```python
#!/usr/bin/env python3
import rclpy
from rclpy.node import Node
from geometry_msgs.msg import Twist
import time


class PrecisionDrive(Node):
    """TurtleBot3 정밀 주행 제어"""

    def __init__(self):
        super().__init__('precision_drive')
        self.publisher_ = self.create_publisher(Twist, '/cmd_vel', 10)
        self.get_logger().info('Precision Drive Node Ready')

    def stop(self):
        """정지"""
        msg = Twist()
        self.publisher_.publish(msg)

    def move_forward(self, speed=0.1, duration=2.0):
        """지정 시간 동안 직진"""
        msg = Twist()
        msg.linear.x = speed
        self._publish_duration(msg, duration)
        self.get_logger().info(f'Moved forward {speed:.2f} m/s for {duration:.1f}s')

    def turn(self, angular_speed=0.5, duration=1.0):
        """지정 시간 동안 회전"""
        msg = Twist()
        msg.angular.z = angular_speed
        self._publish_duration(msg, duration)
        self.get_logger().info(f'Turned at {angular_speed:.2f} rad/s for {duration:.1f}s')

    def drive_square(self, side_length=0.5):
        """정사각형 주행"""
        speed = 0.1
        move_time = side_length / speed
        turn_time = 1.57 / 0.5  # 90° = 1.57 rad

        for i in range(4):
            self.get_logger().info(f'Square: side {i+1}')
            self.move_forward(speed, move_time)
            self.stop()
            time.sleep(0.5)
            self.turn(0.5, turn_time)
            self.stop()
            time.sleep(0.5)

    def _publish_duration(self, msg, duration):
        """지정 시간 동안 메시지를 지속 발행"""
        rate = self.create_rate(50)  # 50 Hz
        start_time = time.time()
        while time.time() - start_time < duration:
            self.publisher_.publish(msg)
            rate.sleep()
        self.stop()

    def drive_pattern(self, pattern):
        """패턴 주행"""
        for cmd in pattern:
            if cmd['type'] == 'forward':
                self.move_forward(cmd['speed'], cmd['duration'])
            elif cmd['type'] == 'turn':
                self.turn(cmd['speed'], cmd['duration'])
            elif cmd['type'] == 'stop':
                self.stop()
                time.sleep(cmd['duration'])


def main(args=None):
    rclpy.init(args=args)
    drive = PrecisionDrive()

    # 패턴 예시: 사각형
    pattern = [
        {'type': 'forward', 'speed': 0.1, 'duration': 2.0},
        {'type': 'turn', 'speed': 0.5, 'duration': 3.14},  # ~180°
        {'type': 'forward', 'speed': 0.1, 'duration': 2.0},
    ]

    drive.drive_pattern(pattern)
    drive.get_logger().info('Pattern complete!')
    drive.destroy_node()
    rclpy.shutdown()


if __name__ == '__main__':
    main()
```

실행:

```bash
cd ~/ros2_ws
colcon build --symlink-install
source install/setup.bash
ros2 run tb3_control precision_drive
```

---

## 12.5 Go-to-Goal 구현

간단한 Go-to-Goal 알고리즘:

```python
#!/usr/bin/env python3
import rclpy
from rclpy.node import Node
from geometry_msgs.msg import Twist
from nav_msgs.msg import Odometry
import math


class GoToGoal(Node):
    """Odometry 기반 목표 지점 이동"""

    def __init__(self, goal_x=1.0, goal_y=1.0):
        super().__init__('go_to_goal')
        self.publisher_ = self.create_publisher(Twist, '/cmd_vel', 10)
        self.subscription = self.create_subscription(
            Odometry, '/odom', self.odom_callback, 10)

        self.goal_x = goal_x
        self.goal_y = goal_y
        self.current_x = 0.0
        self.current_y = 0.0
        self.current_theta = 0.0

        # 제어 게인
        self.kp_linear = 0.5
        self.kp_angular = 1.0
        self.position_tolerance = 0.05
        self.angle_tolerance = 0.05

        self.timer = self.create_timer(0.1, self.control_loop)

    def odom_callback(self, msg):
        self.current_x = msg.pose.pose.position.x
        self.current_y = msg.pose.pose.position.y

        # Quaternion → Euler
        q = msg.pose.pose.orientation
        siny_cosp = 2 * (q.w * q.z + q.x * q.y)
        cosy_cosp = 1 - 2 * (q.y * q.y + q.z * q.z)
        self.current_theta = math.atan2(siny_cosp, cosy_cosp)

    def control_loop(self):
        # 목표까지 거리와 각도 계산
        dx = self.goal_x - self.current_x
        dy = self.goal_y - self.current_y
        distance = math.sqrt(dx ** 2 + dy ** 2)
        target_angle = math.atan2(dy, dx)

        angle_error = target_angle - self.current_theta
        # 각도 정규화 (-π ~ π)
        angle_error = math.atan2(math.sin(angle_error), math.cos(angle_error))

        cmd = Twist()

        if distance > self.position_tolerance:
            if abs(angle_error) > self.angle_tolerance:
                # 회전 우선
                cmd.angular.z = self.kp_angular * angle_error
            else:
                # 직진
                cmd.linear.x = min(self.kp_linear * distance, 0.2)
                cmd.angular.z = self.kp_angular * angle_error
        else:
            self.get_logger().info(f'🎯 Goal reached! (x={self.current_x:.2f}, y={self.current_y:.2f})')
            self.timer.cancel()

        self.publisher_.publish(cmd)

        # 상태 출력
        self.get_logger().info(
            f'Pos: ({self.current_x:.2f}, {self.current_y:.2f}) '
            f'Dist: {distance:.2f}m '
            f'Angle error: {math.degrees(angle_error):.1f}°'
        )


def main(args=None):
    rclpy.init(args=args)
    goal_x = float(input('Goal X (m): ') or 1.0)
    goal_y = float(input('Goal Y (m): ') or 1.0)
    node = GoToGoal(goal_x, goal_y)
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()
```

---

## 12.6 OpenCR 파라미터 튜닝

```bash
# 현재 PID 게인 확인
ros2 param get /turtlebot3_core kp
ros2 param get /turtlebot3_core ki
ros2 param get /turtlebot3_core kd

# PID 게인 변경 (런타임)
ros2 param set /turtlebot3_core kp 500.0
ros2 param set /turtlebot3_core ki 0.0
ros2 param set /turtlebot3_core kd 0.0

# 변경 후 주행 특성 변화 확인
```

---

## 📝 연습 문제

1. **정사각형 주행:** TurtleBot3를 0.5m x 0.5m 정사각형 경로로 정확히 주행시키고, 종점 오차를 측정하세요
2. **8자 주행:** 좌회전과 우회전을 번갈아가며 ∞(8자) 패턴을 주행시키는 코드를 작성하세요
3. **PID 튜닝:** `kp` 값을 100, 500, 1000으로 변경하며 주행 안정성의 차이를 관찰하고 기록하세요
4. **Go-to-Goal:** (0,0)에서 (1.0, 1.5)까지 자동 주행하는 Go-to-Goal을 실행하고, 실제 경로를 RViz2의 Path로 시각화하세요
5. **부하 측정:** 로봇이 정지 상태일 때와 주행 중일 때 DYNAMIXEL 모터의 부하(load) 값을 `/joint_states`에서 확인하고 비교하세요

---

## ⚠️ 트러블슈팅

| 문제 | 해결 방법 |
|------|----------|
| 모터에서 이상한 소음 | DYNAMIXEL 기어 손상 가능성, OpenCR 재부팅 후 확인 |
| 로봇이 직선으로 안 감 | 바퀴 장착 상태 확인, 양쪽 모터 부하 균일한지 확인 |
| Go-to-Goal이 목표에 수렴 안 함 | `kp_linear`와 `kp_angular` 게인 조정 |
| `/joint_states` 데이터 없음 | `ros2 topic list`에 `/joint_states`가 있는지 먼저 확인 |
| 모터 과열 | 연속 주행 시 5분 간격으로 1분 휴식 권장 |
