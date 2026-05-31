# Day 13 — RViz2 심화 & URDF

> **목표:** TurtleBot3의 URDF 모델 구조를 이해하고, RViz2의 고급 기능을 마스터한다.

---

## 13.1 URDF (Unified Robot Description Format)

URDF는 로봇의 **기구학적/시각적 모델**을 XML로 표현한 파일입니다.

### TurtleBot3 Burger URDF 구조

```xml
<robot name="turtlebot3_burger">
  <link name="base_link">
    <visual>
      <geometry><mesh filename="base_link.stl"/></geometry>
    </visual>
    <collision>
      <geometry><mesh filename="base_link.stl"/></geometry>
    </collision>
    <inertial>
      <mass value="0.5"/>
      <inertia ixx="..."/>
    </inertial>
  </link>

  <joint name="wheel_left_joint" type="continuous">
    <parent link="base_link"/>
    <child link="wheel_left_link"/>
    <origin xyz="0 0.08 0" rpy="0 0 0"/>
    <axis xyz="0 1 0"/>
  </joint>

  <link name="wheel_left_link">
    <visual>
      <geometry><mesh filename="wheel.stl"/></geometry>
    </visual>
  </link>
  <!-- ... 오른쪽 바퀴, LDS, IMU 등 ... -->
</robot>
```

### URDF 확인

```bash
# RPi의 URDF 파일 경로
roscd turtlebot3_description/urdf/
# 또는 직접 경로
ls /opt/ros/humble/share/turtlebot3_description/urdf/

# URDF 내용 보기
cat /opt/ros/humble/share/turtlebot3_description/urdf/turtlebot3_burger.urdf
```

---

## 13.2 URDF → RViz2 연결 과정

```
URDF 파일
    │
    ▼
robot_state_publisher (URDF 파싱 → TF 발행)
    │
    ├── /robot_description (URDF 텍스트)
    ├── /tf (joint → link 변환)
    └── /joint_states (조인트 상태)
            │
            ▼
        RViz2 (3D 시각화)
```

### robot_state_publisher 실행

```bash
# robot_state_publisher 단독 실행
ros2 run robot_state_publisher robot_state_publisher \
  --model /opt/ros/humble/share/turtlebot3_description/urdf/turtlebot3_burger.urdf
```

---

## 13.3 joint_state_publisher GUI

조인트 값을 수동으로 조작:

```bash
# GUI로 조인트 값 조작
sudo apt install -y ros-humble-joint-state-publisher-gui

ros2 launch turtlebot3_bringup rviz2.launch.py &

# 새 터미널
ros2 run joint_state_publisher_gui joint_state_publisher_gui

# 슬라이더로 바퀴 회전값 변경 가능
```

---

## 13.4 RViz2 고급 기능

### 1) Custom Display Plugins

```bash
# 설치된 플러그인 확인
rqt --list-plugins | grep rviz
```

### 2) 특정 Display 집중 학습

| Display | 용도 | 팁 |
|---------|------|-----|
| **Marker** | 임의의 3D 객체 표시 | AR 마커, 경로점 표시 |
| **MarkerArray** | 다중 마커 | Path 시각화 |
| **Polygon** | 다각형 영역 | 금지 구역 표시 |
| **Path** | 이동 경로 | 네비게이션 경로 확인 |
| **PoseArray** | 파티클 필터 분포 | AMCL 입자 분포 확인 |
| **PointCloud2** | 3D 포인트 클라우드 | (Burger는 미지원) |
| **Range** | 초음파/적외선 | 장애물 거리 |
| **Camera** | 카메라 이미지 | USB 카메라 |

### 3) 마커 발행 예제

`marker_publisher.py`:

```python
#!/usr/bin/env python3
import rclpy
from rclpy.node import Node
from visualization_msgs.msg import Marker
from geometry_msgs.msg import Point
import math


class MarkerPublisher(Node):
    """RViz2에 마커 표시"""

    def __init__(self):
        super().__init__('marker_publisher')
        self.publisher_ = self.create_publisher(Marker, '/visualization_marker', 10)
        self.timer_ = self.create_timer(1.0, self.publish_markers)
        self.count_ = 0

    def publish_markers(self):
        # 목표 지점 마커 (구)
        goal_marker = Marker()
        goal_marker.header.frame_id = "odom"
        goal_marker.header.stamp = self.get_clock().now().to_msg()
        goal_marker.ns = "goals"
        goal_marker.id = self.count_
        goal_marker.type = Marker.SPHERE
        goal_marker.action = Marker.ADD

        # 마커 위치: 원형으로 배치
        angle = self.count_ * 0.5
        goal_marker.pose.position.x = 1.0 * math.cos(angle)
        goal_marker.pose.position.y = 1.0 * math.sin(angle)
        goal_marker.pose.position.z = 0.0
        goal_marker.pose.orientation.w = 1.0

        goal_marker.scale.x = 0.1
        goal_marker.scale.y = 0.1
        goal_marker.scale.z = 0.1
        goal_marker.color.a = 1.0
        goal_marker.color.r = 1.0
        goal_marker.color.g = 0.0
        goal_marker.color.b = 0.0

        self.publisher_.publish(goal_marker)

        # 경로 표시 (LineStrip)
        if self.count_ > 2:
            path_marker = Marker()
            path_marker.header.frame_id = "odom"
            path_marker.header.stamp = self.get_clock().now().to_msg()
            path_marker.ns = "paths"
            path_marker.id = 0
            path_marker.type = Marker.LINE_STRIP
            path_marker.action = Marker.ADD

            path_marker.scale.x = 0.03  # 선 두께
            path_marker.color.a = 1.0
            path_marker.color.r = 0.0
            path_marker.color.g = 1.0
            path_marker.color.b = 0.0

            for i in range(self.count_ + 1):
                a = i * 0.5
                p = Point()
                p.x = 1.0 * math.cos(a)
                p.y = 1.0 * math.sin(a)
                p.z = 0.0
                path_marker.points.append(p)

            self.publisher_.publish(path_marker)

        self.count_ += 1
        self.get_logger().info(f'Published marker #{self.count_}')


def main(args=None):
    rclpy.init(args=args)
    node = MarkerPublisher()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()
```

---

## 13.5 RViz2 설정 저장 & 재사용

```bash
# 현재 설정 저장
# RViz2 메뉴: File → Save Config As... → ~/tb3.rviz

# 저장된 설정으로 실행
rviz2 -d ~/tb3.rviz

# 또는 launch 파일에서 지정
ros2 launch turtlebot3_bringup rviz2.launch.py \
  rviz_config:=/path/to/custom.rviz
```

---

## 13.6 RViz2 Plugins 탐방

### Marker Types

```
ARROW          = 0    # 화살표
CUBE           = 1    # 정육면체
SPHERE         = 2    # 구
CYLINDER       = 3    # 원기둥
LINE_STRIP     = 4    # 선 연결
LINE_LIST      = 5    # 독립 선분
CUBE_LIST      = 6    # 큐브 리스트
SPHERE_LIST    = 7    # 구 리스트
POINTS         = 8    # 포인트
TEXT_VIEW_FACING = 9  # 텍스트
MESH_RESOURCE  = 10   # 메시
TRIANGLE_LIST  = 11   # 삼각형
```

---

## 13.7 RViz2 vs rqt_tools

| 도구 | 용도 | 특징 |
|------|------|------|
| **RViz2** | 3D 로봇/센서 시각화 | RobotModel, LaserScan, TF, Marker |
| **rqt_graph** | 노드 통신 그래프 | 토픽/서비스/액션 관계 |
| **rqt_plot** | 2D 데이터 플롯 | 수치 데이터 시계열 |
| **rqt_console** | 로그 메시지 뷰어 | 필터링, 심각도 |
| **rqt_topic** | 토픽 모니터 | 발행 주기, 대역폭 |
| **rqt_reconfigure** | 동적 파라미터 | 런타임 값 변경 |

---

## 📝 연습 문제

1. **URDF 분석:** TurtleBot3 Burger의 URDF에서 모든 link와 joint의 이름, parent-child 관계를 그려보세요
2. **Marker 실습:** LIDAR 장애물 위치에 SPHERE 마커를 동적으로 표시하는 노드를 작성하세요
3. **Text Marker:** RViz2에 현재 배터리 전압을 TEXT_VIEW_FACING 마커로 표시하는 노드를 만드세요
4. **경로 시각화:** Go-to-Goal 주행 시 이동 경로를 LINE_STRIP 마커로 실시간 표시하세요
5. **Color Coding:** 마커의 색상을 로봇 속도에 따라 변경하세요 (정지=녹색, 이동=파란색, 급정지=빨간색)

---

## ⚠️ 트러블슈팅

| 문제 | 해결 방법 |
|------|----------|
| URDF 로딩 실패 | `check_urdf` 명령어로 URDF 문법 검증 |
| RViz2에서 마커 안 보임 | `Global Options > Fixed Frame`이 `odom` 또는 `map`인지 확인 |
| Marker 색상이 적용 안 됨 | `color.a = 1.0` (알파값)이 설정되었는지 확인 |
| joint_state_publisher GUI 안 뜸 | `sudo apt install -y ros-humble-joint-state-publisher-gui` |
| RViz2 느림 | `Grid` display의 Cell Count 줄이기, LaserScan `Decay Time` 줄이기 |
