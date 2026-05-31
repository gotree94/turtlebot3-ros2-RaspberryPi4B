# Day 20 — 최종 프로젝트 구현

> **목표:** 자율 순찰 로봇의 핵심 노드를 구현하고 통합 테스트를 진행한다.

---

## 20.1 패키지 생성

```bash
# PC에서
cd ~/ros2_ws/src
ros2 pkg create patrol_bot \
  --build-type ament_python \
  --dependencies rclpy std_msgs geometry_msgs nav_msgs sensor_msgs visualization_msgs
```

---

## 20.2 구현: Waypoint Navigator

`~/ros2_ws/src/patrol_bot/patrol_bot/waypoint_navigator.py`:

```python
#!/usr/bin/env python3
"""
TurtleBot3 자율 순찰 - Waypoint Navigator

Nav2 액션 클라이언트를 사용하여 Waypoint를 순차적으로 방문
"""

import rclpy
from rclpy.node import Node
from rclpy.action import ActionClient
from nav2_msgs.action import NavigateToPose
from geometry_msgs.msg import PoseStamped
import yaml
import os
import math
from datetime import datetime


class WaypointNavigator(Node):
    """Waypoint 기반 자율 순찰 내비게이터"""

    def __init__(self):
        super().__init__('waypoint_navigator')

        # Nav2 액션 클라이언트
        self.nav_client = ActionClient(self, NavigateToPose, '/navigate_to_pose')

        # 상태 변수
        self.waypoints = []
        self.current_index = 0
        self.patrol_complete = False
        self.repeat_count = 1
        self.current_cycle = 0
        self.status_pub = self.create_publisher(
            std_msgs.msg.String, '/patrol_status', 10
        )

        # Config 로드
        self.load_config()

        self.get_logger().info('🚀 Waypoint Navigator initialized')
        self.get_logger().info(f'Loaded {len(self.waypoints)} waypoints')
        self.get_logger().info(f'Repeat count: {self.repeat_count}')

    def load_config(self):
        """YAML config 파일 로드"""
        config_path = os.path.expanduser('~/patrol_bot/config/waypoints.yaml')

        if not os.path.exists(config_path):
            self.get_logger().error(f'Config not found: {config_path}')
            self.create_sample_config(config_path)
            return

        try:
            with open(config_path, 'r') as f:
                config = yaml.safe_load(f)

            # Waypoint 리스트
            for wp in config.get('waypoints', []):
                self.waypoints.append({
                    'name': wp.get('name', f'WP_{len(self.waypoints)}'),
                    'position': {
                        'x': wp['position']['x'],
                        'y': wp['position']['y'],
                        'z': wp['position'].get('z', 0.0)
                    },
                    'yaw': math.radians(wp.get('orientation', {}).get('yaw', 0.0)),
                    'actions': wp.get('actions', [])
                })

            # 순찰 설정
            self.repeat_count = config.get('patrol_config', {}).get('repeat_count', 1)
            self.wait_between_cycles = config.get('patrol_config', {}).get(
                'wait_between_cycles', 10
            )
            self.battery_check_interval = config.get('patrol_config', {}).get(
                'battery_check_interval', 5
            )

        except Exception as e:
            self.get_logger().error(f'Config load error: {e}')

    def create_sample_config(self, path):
        """샘플 config 생성"""
        os.makedirs(os.path.dirname(path), exist_ok=True)
        sample = {
            'waypoints': [
                {
                    'name': 'Point_1',
                    'position': {'x': 1.0, 'y': 0.0, 'z': 0.0},
                    'orientation': {'yaw': 0.0},
                    'actions': [{'type': 'wait', 'duration': 2.0}]
                },
                {
                    'name': 'Point_2',
                    'position': {'x': 0.0, 'y': 1.0, 'z': 0.0},
                    'orientation': {'yaw': 90.0},
                    'actions': [{'type': 'wait', 'duration': 2.0}]
                }
            ],
            'patrol_config': {
                'repeat_count': 1,
                'wait_between_cycles': 5,
                'battery_check_interval': 5
            }
        }
        with open(path, 'w') as f:
            yaml.dump(sample, f, default_flow_style=False)
        self.get_logger().info(f'Created sample config: {path}')

    def start_patrol(self):
        """순찰 시작"""
        if not self.waypoints:
            self.get_logger().error('No waypoints to patrol!')
            return

        self.get_logger().info(f'=== Starting Patrol Cycle {self.current_cycle + 1} ===')
        self.send_next_goal()

    def send_next_goal(self):
        """다음 Waypoint로 goal 전송"""
        if self.current_index >= len(self.waypoints):
            self.current_cycle += 1
            if self.current_cycle < self.repeat_count:
                self.current_index = 0
                self.get_logger().info(
                    f'Cycle {self.current_cycle}/{self.repeat_count} complete. '
                    f'Waiting {self.wait_between_cycles}s before next cycle...'
                )
                self.create_timer(self.wait_between_cycles, self.start_patrol)
                return
            else:
                self.patrol_complete = True
                self.get_logger().info('✅ All patrol cycles complete!')
                return

        wp = self.waypoints[self.current_index]
        self.get_logger().info(f'📍 Navigating to Waypoint {self.current_index}: {wp["name"]}')

        # Goal 메시지 생성
        goal_msg = NavigateToPose.Goal()
        goal_msg.pose.header.frame_id = 'map'
        goal_msg.pose.header.stamp = self.get_clock().now().to_msg()
        goal_msg.pose.pose.position.x = wp['position']['x']
        goal_msg.pose.pose.position.y = wp['position']['y']
        goal_msg.pose.pose.position.z = wp['position']['z']

        # Yaw → Quaternion
        goal_msg.pose.pose.orientation.z = math.sin(wp['yaw'] / 2.0)
        goal_msg.pose.pose.orientation.w = math.cos(wp['yaw'] / 2.0)

        # 상태 발행
        self.publish_status(f'Navigating to {wp["name"]}')

        # Goal 전송
        self.send_goal_future = self.nav_client.send_goal_async(
            goal_msg,
            feedback_callback=self.feedback_callback
        )
        self.send_goal_future.add_done_callback(self.goal_response_callback)

    def goal_response_callback(self, future):
        """Goal 응답 콜백"""
        goal_handle = future.result()
        if not goal_handle.accepted:
            self.get_logger().error('Goal rejected by Nav2!')
            self.current_index += 1
            self.send_next_goal()
            return

        self.get_logger().info('Goal accepted, waiting for result...')
        self.result_future = goal_handle.get_result_async()
        self.result_future.add_done_callback(self.goal_result_callback)

    def goal_result_callback(self, future):
        """Goal 결과 콜백"""
        result = future.result().result
        wp = self.waypoints[self.current_index]

        if result:
            self.get_logger().info(f'✅ Reached waypoint: {wp["name"]}')

            # Waypoint 액션 실행
            self.execute_wp_actions(wp['actions'])

            # 다음 Waypoint
            self.current_index += 1
            self.send_next_goal()
        else:
            self.get_logger().error(f'❌ Failed to reach: {wp["name"]}')
            self.current_index += 1
            self.send_next_goal()

    def feedback_callback(self, feedback_msg):
        """Nav2 피드백 콜백"""
        feedback = feedback_msg.feedback
        # 현재 위치와 목표까지 거리 출력 (5회에 1번)
        if hasattr(feedback, 'distance_remaining'):
            dist = feedback.distance_remaining
            if dist is not None:
                self.get_logger().debug(f'Distance to goal: {dist:.2f}m')

    def execute_wp_actions(self, actions):
        """Waypoint 도착 후 액션 실행"""
        for action in actions:
            if action['type'] == 'wait':
                self.get_logger().info(f'Waiting for {action["duration"]}s...')
                # rclpy spin을 방해하지 않고 대기
                start = self.get_clock().now()
                while (self.get_clock().now() - start).nanoseconds / 1e9 < action['duration']:
                    rclpy.spin_once(self, timeout_sec=0.1)

            elif action['type'] == 'log':
                self.get_logger().info(f'📝 {action.get("message", "Waypoint visited")}')

            elif action['type'] == 'scan':
                # 현재 스캔 데이터 기록 (확장 포인트)
                self.get_logger().info('📡 Scan data logging (placeholder)')

    def publish_status(self, status_text):
        """상태 메시지 발행"""
        msg = String()
        msg.data = f'[{datetime.now().strftime("%H:%M:%S")}] {status_text}'
        self.status_pub.publish(msg)

    def stop(self):
        """안전 정지"""
        self.patrol_complete = True
        self.get_logger().info('🛑 Patrol stopped')


def main(args=None):
    rclpy.init(args=args)
    navigator = WaypointNavigator()

    try:
        navigator.start_patrol()
        rclpy.spin(navigator)
    except KeyboardInterrupt:
        navigator.stop()
        navigator.get_logger().info('Patrol interrupted by user')
    finally:
        navigator.destroy_node()
        rclpy.shutdown()


if __name__ == '__main__':
    main()
```

---

## 20.3 구현: Battery Monitor

`~/ros2_ws/src/patrol_bot/patrol_bot/battery_monitor.py`:

```python
#!/usr/bin/env python3
"""
TurtleBot3 배터리 모니터 노드

배터리 전압을 지속적으로 모니터링하고,
저전압 상황에서 귀환 명령을 발행
"""

import rclpy
from rclpy.node import Node
from sensor_msgs.msg import BatteryState
from std_msgs.msg import String
from geometry_msgs.msg import PoseStamped


class BatteryMonitor(Node):
    """배터리 모니터링 및 비상 대응"""

    def __init__(self):
        super().__init__('battery_monitor')

        # 파라미터
        self.declare_parameter('low_threshold', 11.0)
        self.declare_parameter('critical_threshold', 10.5)
        self.declare_parameter('charger_position_x', 0.0)
        self.declare_parameter('charger_position_y', 0.0)

        # Publisher & Subscriber
        self.battery_sub = self.create_subscription(
            BatteryState, '/battery_state', self.battery_callback, 10)
        self.status_pub = self.create_publisher(String, '/patrol_status', 10)

        # 상태
        self.voltage = 0.0
        self.low_warning_issued = False

        self.get_logger().info('🔋 Battery Monitor started')
        self.get_logger().info(f'Low threshold: {self.low_threshold}V')
        self.get_logger().info(f'Critical threshold: {self.critical_threshold}V')

    @property
    def low_threshold(self):
        return self.get_parameter('low_threshold').value

    @property
    def critical_threshold(self):
        return self.get_parameter('critical_threshold').value

    def battery_callback(self, msg):
        """배터리 상태 업데이트"""
        self.voltage = msg.voltage

        if self.voltage <= 0:
            return  # 아직 데이터 없음

        if self.voltage < self.critical_threshold:
            self.emergency_stop()
        elif self.voltage < self.low_threshold:
            if not self.low_warning_issued:
                self.low_warning_issued = True
                self.request_return_to_charger()
        else:
            self.low_warning_issued = False

    def request_return_to_charger(self):
        """충전소 귀환 요청"""
        msg = String()
        msg.data = f'BATTERY_LOW:{self.voltage:.1f}V'
        self.status_pub.publish(msg)
        self.get_logger().warn(
            f'⚠️  Battery low ({self.voltage:.1f}V)! Returning to charger...'
        )

    def emergency_stop(self):
        """비상 정지"""
        msg = String()
        msg.data = f'BATTERY_CRITICAL:{self.voltage:.1f}V'
        self.status_pub.publish(msg)
        self.get_logger().error(
            f'🚨 BATTERY CRITICAL ({self.voltage:.1f}V)! EMERGENCY STOP!'
        )


def main(args=None):
    rclpy.init(args=args)
    node = BatteryMonitor()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()
```

---

## 20.4 구현: Patrol Monitor

`~/ros2_ws/src/patrol_bot/patrol_bot/patrol_monitor.py`:

```python
#!/usr/bin/env python3
"""
TurtleBot3 순찰 모니터 노드

순찰 상태를 실시간으로 모니터링하고 통계 기록
"""

import rclpy
from rclpy.node import Node
from std_msgs.msg import String
from nav_msgs.msg import Odometry
from datetime import datetime
import csv
import os


class PatrolMonitor(Node):
    """순찰 상태 모니터링 및 로깅"""

    def __init__(self):
        super().__init__('patrol_monitor')

        # Subscriber
        self.status_sub = self.create_subscription(
            String, '/patrol_status', self.status_callback, 10)
        self.odom_sub = self.create_subscription(
            Odometry, '/odom', self.odom_callback, 10)

        # 통계
        self.stats = {
            'start_time': datetime.now(),
            'waypoints_visited': 0,
            'total_distance': 0.0,
            'max_distance_from_start': 0.0,
            'status_messages': [],
            'battery_alerts': []
        }
        self.last_position = None
        self.start_position = None
        self.log_file = os.path.expanduser(f'~/patrol_bot/logs/patrol_{datetime.now():%Y%m%d_%H%M}.csv')

        # CSV 헤더
        self.init_csv()

        # 상태 발행 타이머 (5초)
        self.timer = self.create_timer(5.0, self.report_status)

        self.get_logger().info('📊 Patrol Monitor started')

    def init_csv(self):
        os.makedirs(os.path.dirname(self.log_file), exist_ok=True)
        with open(self.log_file, 'w', newline='') as f:
            writer = csv.writer(f)
            writer.writerow(['timestamp', 'event', 'details'])

    def log_event(self, event, details=''):
        with open(self.log_file, 'a', newline='') as f:
            writer = csv.writer(f)
            writer.writerow([datetime.now().isoformat(), event, details])

    def status_callback(self, msg):
        """상태 메시지 수신"""
        self.stats['status_messages'].append(msg.data)
        self.get_logger().info(f'📡 {msg.data}')
        self.log_event('status', msg.data)

        if 'reached' in msg.data.lower():
            self.stats['waypoints_visited'] += 1
        elif 'battery' in msg.data.lower():
            self.stats['battery_alerts'].append(msg.data)

    def odom_callback(self, msg):
        """Odometry 데이터 기록"""
        x = msg.pose.pose.position.x
        y = msg.pose.pose.position.y

        if self.start_position is None:
            self.start_position = (x, y)

        if self.last_position is not None:
            dx = x - self.last_position[0]
            dy = y - self.last_position[1]
            dist = (dx ** 2 + dy ** 2) ** 0.5
            self.stats['total_distance'] += dist

        self.last_position = (x, y)

        # 시작점으로부터 최대 거리
        if self.start_position:
            dist_from_start = ((x - self.start_position[0]) ** 2 +
                               (y - self.start_position[1]) ** 2) ** 0.5
            self.stats['max_distance_from_start'] = max(
                self.stats['max_distance_from_start'], dist_from_start
            )

    def report_status(self):
        """5초마다 상태 보고"""
        elapsed = (datetime.now() - self.stats['start_time']).total_seconds()
        self.get_logger().info(
            f'⏱️  Elapsed: {elapsed:.0f}s | '
            f'📍 Waypoints: {self.stats["waypoints_visited"]} | '
            f'📏 Distance: {self.stats["total_distance"]:.1f}m'
        )

    def generate_final_report(self):
        """최종 보고서 생성"""
        elapsed = (datetime.now() - self.stats['start_time']).total_seconds()
        report = f"""
╔══════════════════════════════════════╗
║     Patrol Mission Report            ║
╠══════════════════════════════════════╣
║ Duration:           {elapsed:.0f}s
║ Waypoints Visited:  {self.stats['waypoints_visited']}
║ Total Distance:     {self.stats['total_distance']:.1f}m
║ Max Range:          {self.stats['max_distance_from_start']:.1f}m
║ Battery Alerts:     {len(self.stats['battery_alerts'])}
║ Status Messages:    {len(self.stats['status_messages'])}
╚══════════════════════════════════════╝
"""
        self.get_logger().info(report)
        self.log_event('report', report)
        return report

    def destroy_node(self):
        self.generate_final_report()
        super().destroy_node()


def main(args=None):
    rclpy.init(args=args)
    node = PatrolMonitor()
    try:
        rclpy.spin(node)
    except KeyboardInterrupt:
        pass
    finally:
        node.destroy_node()
        rclpy.shutdown()


if __name__ == '__main__':
    main()
```

---

## 20.5 통합 Launch 파일

`~/ros2_ws/src/patrol_bot/launch/patrol.launch.py`:

```python
from launch import LaunchDescription
from launch_ros.actions import Node
from launch.actions import ExecuteProcess
import os


def generate_launch_description():
    # Nav2 실행 (저장된 맵 기반)
    nav2_cmd = ExecuteProcess(
        cmd=[
            'ros2', 'launch', 'turtlebot3_navigation2', 'navigation2.launch.py',
            f'map:={os.path.expanduser("~/my_map.yaml")}',
            'use_sim_time:=false'
        ],
        output='screen'
    )

    # Waypoint Navigator
    waypoint_node = Node(
        package='patrol_bot',
        executable='waypoint_navigator',
        name='waypoint_navigator',
        output='screen'
    )

    # Battery Monitor
    battery_node = Node(
        package='patrol_bot',
        executable='battery_monitor',
        name='battery_monitor',
        parameters=[{
            'low_threshold': 11.0,
            'critical_threshold': 10.5,
            'charger_position_x': 0.0,
            'charger_position_y': 0.0
        }],
        output='screen'
    )

    # Patrol Monitor
    monitor_node = Node(
        package='patrol_bot',
        executable='patrol_monitor',
        name='patrol_monitor',
        output='screen'
    )

    return LaunchDescription([
        nav2_cmd,
        waypoint_node,
        battery_node,
        monitor_node,
    ])
```

---

## 20.6 통합 테스트 절차

### Phase 1: 단위 테스트

```bash
# 각 노드 개별 테스트
# 1. Waypoint Navigator (단독)
ros2 run patrol_bot waypoint_navigator

# 2. Battery Monitor (단독)
ros2 run patrol_bot battery_monitor

# 3. Patrol Monitor (단독)
ros2 run patrol_bot patrol_monitor
```

### Phase 2: 통합 테스트

```bash
# 1. RPi에서 Bringup
ssh ubuntu@turtlebot-pi.local
ros2 launch turtlebot3_bringup robot.launch.py

# 2. PC에서 SLAM 실행 (매핑되지 않은 경우)
ros2 launch slam_toolbox online_async_launch.py \
  slam_params_file:=/opt/ros/humble/share/turtlebot3_navigation2/param/burger.yaml \
  use_sim_time:=false

# 3. PC에서 Nav2 + Patrol 실행
ros2 launch patrol_bot patrol.launch.py
```

### Phase 3: 실제 테스트

```bash
# 2-3개의 Waypoint로 시작
# 각 Waypoint 도착 확인
# 배터리 부족 시나리오 테스트 (의도적으로 배터리 부족 상태)
# 다양한 장애물 배치 후 테스트
```

---

## 📝 오늘의 연습 문제

1. **Waypoint 경로:** 자신의 환경에 맞는 3개 Waypoint YAML 파일을 작성하고 순찰을 실행하세요
2. **실시간 모니터링:** RViz2로 로봇의 위치를 실시간으로 추적하면서 Waypoint Navigator가 정상 동작하는지 확인하세요
3. **에러 처리:** Waypoint에 도달할 수 없는 경우 (예: 맵 밖의 좌표) 로그를 확인하고 시스템이 어떻게 반응하는지 기록하세요
4. **성능 측정:** 3개 Waypoint 순회 완료 시간과 총 이동 거리를 측정하세요
5. **배터리 시뮬레이션:** 배터리 임계값을 임시로 12.5V로 설정하고, 정상 주행 중에도 귀환 명령이 발행되는지 테스트하세요 (테스트 후 원복)
