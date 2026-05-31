# Day 5 — 첫 ROS2 패키지 작성

> **목표:** 직접 Publisher 노드와 Subscriber 노드를 Python과 C++로 작성하고 실행한다.

---

## 5.1 개요

ROS2에서 패키지는 노드, 라이브러리, 설정 파일 등을 묶는 단위입니다.
오늘은 가장 기본적인 **Publisher** (데이터 발행)와 **Subscriber** (데이터 수신) 노드를 직접 만들어봅니다.

---

## 5.2 ROS2 패키지 구조

```
my_first_package/              # 패키지 루트
├── package.xml                # 패키지 메타정보 (의존성)
├── setup.py                   # Python 패키지 설정
├── setup.cfg                  # 설치 설정
├── resource/
│   └── my_first_package       # 마커 파일 (패키지 이름)
├── launch/                    # 런치 파일 (선택)
│   └── demo.launch.py
├── msg/                       # 커스텀 메시지 (선택)
│   └── Custom.msg
├── srv/                       # 커스텀 서비스 (선택)
│   └── Custom.srv
└── my_first_package/          # Python 모듈
    └── __init__.py
```

---

## 5.3 Python Publisher 노드 만들기

### Step 1: 패키지 생성

```bash
# 워크스페이스 생성 (이미 있으면 건너뜀)
mkdir -p ~/ros2_ws/src
cd ~/ros2_ws/src

# Python 패키지 생성
ros2 pkg create my_first_package \
  --build-type ament_python \
  --dependencies rclpy std_msgs \
  --node-name publisher_node
```

### Step 2: 작성된 코드 확인

```bash
# 생성된 파일 확인
ls -la ~/ros2_ws/src/my_first_package/my_first_package/
```

### Step 3: Publisher 노드 수정

`~/ros2_ws/src/my_first_package/my_first_package/publisher_node.py`:

```python
import rclpy
from rclpy.node import Node
from std_msgs.msg import String


class SimplePublisher(Node):
    """1초마다 Hello 메시지를 발행하는 Publisher 노드"""

    def __init__(self):
        super().__init__('simple_publisher')
        self.publisher_ = self.create_publisher(String, '/hello_topic', 10)
        self.timer_ = self.create_timer(1.0, self.timer_callback)
        self.counter_ = 0
        self.get_logger().info('Publisher node has been started!')

    def timer_callback(self):
        msg = String()
        msg.data = f'Hello ROS2 from Sisyphus: #{self.counter_}'
        self.publisher_.publish(msg)
        self.get_logger().info(f'Publishing: "{msg.data}"')
        self.counter_ += 1


def main(args=None):
    rclpy.init(args=args)
    node = SimplePublisher()
    try:
        rclpy.spin(node)  # 노드가 종료될 때까지 대기
    except KeyboardInterrupt:
        node.get_logger().info('Publisher node shutting down...')
    finally:
        node.destroy_node()
        rclpy.shutdown()


if __name__ == '__main__':
    main()
```

### Step 4: 빌드 및 실행

```bash
# 워크스페이스 빌드
cd ~/ros2_ws
colcon build --symlink-install

# 환경 설정
source install/setup.bash

# 노드 실행
ros2 run my_first_package publisher_node
```

---

## 5.4 Python Subscriber 노드 추가

### Step 1: subscriber_node.py 생성

`~/ros2_ws/src/my_first_package/my_first_package/subscriber_node.py`:

```python
import rclpy
from rclpy.node import Node
from std_msgs.msg import String


class SimpleSubscriber(Node):
    """/hello_topic 토픽을 구독하는 Subscriber 노드"""

    def __init__(self):
        super().__init__('simple_subscriber')
        self.subscription_ = self.create_subscription(
            String,
            '/hello_topic',
            self.listener_callback,
            10
        )
        self.get_logger().info('Subscriber node has been started!')

    def listener_callback(self, msg):
        self.get_logger().info(f'Received: "{msg.data}"')


def main(args=None):
    rclpy.init(args=args)
    node = SimpleSubscriber()
    try:
        rclpy.spin(node)
    except KeyboardInterrupt:
        node.get_logger().info('Subscriber node shutting down...')
    finally:
        node.destroy_node()
        rclpy.shutdown()


if __name__ == '__main__':
    main()
```

### Step 2: setup.py에 노드 등록

`~/ros2_ws/src/my_first_package/setup.py`에서 `entry_points` 수정:

```python
entry_points={
    'console_scripts': [
        'publisher_node = my_first_package.publisher_node:main',
        'subscriber_node = my_first_package.subscriber_node:main',
    ],
},
```

### Step 3: 빌드 및 테스트

```bash
cd ~/ros2_ws
colcon build --symlink-install
source install/setup.bash

# 터미널 1: publisher 실행
ros2 run my_first_package publisher_node

# 터미널 2: subscriber 실행
source ~/ros2_ws/install/setup.bash
ros2 run my_first_package subscriber_node
```

---

## 5.5 (선택) C++ 버전으로 동일 노드 작성

### Step 1: C++ 패키지 생성

```bash
cd ~/ros2_ws/src
ros2 pkg create my_cpp_package \
  --build-type ament_cmake \
  --dependencies rclcpp std_msgs
```

### Step 2: C++ Publisher

`~/ros2_ws/src/my_cpp_package/src/publisher_node.cpp`:

```cpp
#include <chrono>
#include <memory>
#include <string>

#include "rclcpp/rclcpp.hpp"
#include "std_msgs/msg/string.hpp"

using namespace std::chrono_literals;

class SimplePublisher : public rclcpp::Node
{
public:
  SimplePublisher() : Node("simple_publisher_cpp"), counter_(0)
  {
    publisher_ = this->create_publisher<std_msgs::msg::String>("/hello_topic_cpp", 10);
    timer_ = this->create_wall_timer(
      1s, std::bind(&SimplePublisher::timer_callback, this));
    RCLCPP_INFO(this->get_logger(), "C++ Publisher node started!");
  }

private:
  void timer_callback()
  {
    auto msg = std_msgs::msg::String();
    msg.data = "Hello from C++: #" + std::to_string(counter_++);
    publisher_->publish(msg);
    RCLCPP_INFO(this->get_logger(), "Publishing: '%s'", msg.data.c_str());
  }

  rclcpp::Publisher<std_msgs::msg::String>::SharedPtr publisher_;
  rclcpp::TimerBase::SharedPtr timer_;
  int counter_;
};

int main(int argc, char* argv[])
{
  rclcpp::init(argc, argv);
  rclcpp::spin(std::make_shared<SimplePublisher>());
  rclcpp::shutdown();
  return 0;
}
```

### Step 3: CMakeLists.txt 수정

`~/ros2_ws/src/my_cpp_package/CMakeLists.txt`에 추가:

```cmake
add_executable(publisher_node src/publisher_node.cpp)
ament_target_dependencies(publisher_node rclcpp std_msgs)

install(TARGETS
  publisher_node
  DESTINATION lib/${PROJECT_NAME}
)
```

### Step 4: 빌드 및 실행

```bash
cd ~/ros2_ws
colcon build --symlink-install
source install/setup.bash
ros2 run my_cpp_package publisher_node
```

---

## 5.6 Launch 파일로 여러 노드 실행

`~/ros2_ws/src/my_first_package/launch/demo.launch.py`:

```python
from launch import LaunchDescription
from launch_ros.actions import Node


def generate_launch_description():
    return LaunchDescription([
        Node(
            package='my_first_package',
            executable='publisher_node',
            name='publisher',
            output='screen'
        ),
        Node(
            package='my_first_package',
            executable='subscriber_node',
            name='subscriber',
            output='screen'
        ),
    ])
```

`setup.py`에 launch 파일 포함:

```python
from setuptools import setup
import os
from glob import glob

# setup() 내부에 추가:
data_files=[
    (os.path.join('share', 'my_first_package', 'launch'),
     glob(os.path.join('launch', '*launch.[pxy][yma]*')))
]
```

실행:

```bash
cd ~/ros2_ws
colcon build --symlink-install
source install/setup.bash
ros2 launch my_first_package demo.launch.py
```

---

## 5.7 핵심 패턴 이해

### Node 작성 패턴 (Python)

```python
import rclpy
from rclpy.node import Node

class MyNode(Node):
    def __init__(self):
        super().__init__('node_name')  # 노드 이름
        # Publisher, Subscriber, Timer, Service 등을 여기에 생성

    def callback_function(self):
        pass

def main(args=None):
    rclpy.init(args=args)
    node = MyNode()
    rclpy.spin(node)  # 콜백 함수가 실행되도록 무한 루프
    node.destroy_node()
    rclpy.shutdown()

if __name__ == '__main__':
    main()
```

---

## 📝 연습 문제

1. **카운터 노드:** 0.5초마다 1씩 증가하는 Int32 메시지를 `/counter` 토픽으로 발행하는 노드를 만드세요
2. **제곱 노드:** `/counter` 토픽을 구독하여 값을 제곱한 결과를 `/squared` 토픽으로 다시 발행하는 노드를 만드세요
3. **런치 파일:** 위 두 노드를 한 번에 실행하는 런치 파일을 작성하세요
4. **멀티 토픽:** `/temperature`, `/humidity` 두 개의 토픽을 동시에 발행하는 노드를 만드세요
5. **C++ 도전:** Python과 동일한 기능의 Subscriber 노드를 C++로도 작성해보세요

---

## ⚠️ 트러블슈팅

| 문제 | 해결 방법 |
|------|----------|
| `ModuleNotFoundError` | `setup.py`에 `entry_points`가 올바른지 확인 |
| 빌드 후 `find_package()` 에러 | `package.xml`에 모든 의존성이 선언되었는지 확인 |
| `colcon build` 실패 | `colcon build --symlink-install --cmake-clean-cache`로 정리 후 재시도 |
| 노드 실행 시 "package not found" | `source install/setup.bash` 실행 확인 |
| launch 파일 실행 안 됨 | `setup.py`에 `data_files` 설정 확인 |
