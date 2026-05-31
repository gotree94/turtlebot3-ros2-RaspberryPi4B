# Day 7 — 미니 프로젝트: CPU 온도 모니터링 노드

> **목표:** Week 1에서 배운 ROS2 Publisher/Subscriber 개념을 활용하여 RPi의 CPU 온도를 모니터링하는 시스템을 만든다.

---

## 7.1 프로젝트 개요

Raspberry Pi의 CPU 온도를 읽어 ROS2 토픽으로 발행하고, Remote PC에서 실시간으로 모니터링하며 온도가 임계치를 초과하면 경고를 출력하는 시스템을 구축합니다.

```
┌─────────────────────────┐    /cpu_temp     ┌─────────────────────────┐
│  RPi (Publisher Node)   │ ──────────────►  │  PC (Subscriber Node)   │
│                         │                   │                         │
│  - CPU 온도 읽기         │     Float64      │  - 온도 그래프 출력       │
│  - 1초마다 publish       │                   │  - 70°C 초과 경고       │
│  - rqt_plot으로 전송     │                   │  - CSV 파일 저장         │
└─────────────────────────┘                   └─────────────────────────┘
```

---

## 7.2 준비 사항

- RPi에 ROS2 Humble 설치 완료
- Remote PC에 ROS2 Humble 설치 완료
- RPi ↔ PC ROS_DOMAIN_ID 통일
- Week 1 Day 5 내용 숙지

---

## 7.3 구현: RPi Publisher Node

### Step 1: 패키지 생성 (RPi에서)

```bash
# 이미 ~/ros2_ws가 있으면 그대로 사용
mkdir -p ~/ros2_ws/src
cd ~/ros2_ws/src

# Python 패키지 생성
ros2 pkg create temp_monitor \
  --build-type ament_python \
  --dependencies rclpy std_msgs
```

### Step 2: Publisher 노드 작성

`~/ros2_ws/src/temp_monitor/temp_monitor/cpu_temp_publisher.py`:

```python
#!/usr/bin/env python3
import rclpy
from rclpy.node import Node
from std_msgs.msg import Float64
import subprocess
import re


class CPUTempPublisher(Node):
    """
    Raspberry Pi의 CPU 온도를 읽어 /cpu_temp 토픽으로 발행하는 노드
    """

    def __init__(self):
        super().__init__('cpu_temp_publisher')
        self.publisher_ = self.create_publisher(Float64, '/cpu_temp', 10)
        self.timer_ = self.create_timer(1.0, self.publish_temperature)
        self.threshold_high = 80.0  # 경고 임계 온도 (°C)
        self.get_logger().info('CPU Temperature Publisher started!')

    def read_cpu_temp(self):
        """RPi의 CPU 온도를 읽어 섭씨 온도로 반환"""
        try:
            # RPi의 온도 센서 파일 읽기
            result = subprocess.run(
                ['cat', '/sys/class/thermal/thermal_zone0/temp'],
                capture_output=True,
                text=True,
                timeout=2
            )
            if result.returncode == 0:
                # millidegree Celsius → Celsius 변환
                temp_raw = result.stdout.strip()
                temp_celsius = float(temp_raw) / 1000.0
                return temp_celsius
            else:
                self.get_logger().error('Failed to read temperature')
                return None
        except Exception as e:
            self.get_logger().error(f'Error reading temperature: {e}')
            return None

    def publish_temperature(self):
        """온도를 읽어 publish하고 조건에 따라 로그 출력"""
        temp = self.read_cpu_temp()

        if temp is not None:
            msg = Float64()
            msg.data = temp
            self.publisher_.publish(msg)

            # 로그 출력
            if temp > self.threshold_high:
                self.get_logger().warn(
                    f'🔥 HIGH TEMPERATURE: {temp:.1f}°C (threshold: {self.threshold_high}°C)'
                )
            else:
                self.get_logger().info(f'CPU Temperature: {temp:.1f}°C')
        else:
            self.get_logger().error('Could not read CPU temperature!')


def main(args=None):
    rclpy.init(args=args)
    node = CPUTempPublisher()
    try:
        rclpy.spin(node)
    except KeyboardInterrupt:
        node.get_logger().info('Temperature Publisher shutting down...')
    finally:
        node.destroy_node()
        rclpy.shutdown()


if __name__ == '__main__':
    main()
```

### Step 3: setup.py 수정

`~/ros2_ws/src/temp_monitor/setup.py`의 `entry_points`:

```python
entry_points={
    'console_scripts': [
        'cpu_temp_publisher = temp_monitor.cpu_temp_publisher:main',
        'temp_monitor = temp_monitor.temp_monitor:main',  # 다음 단계에서 추가
    ],
},
```

### Step 4: 빌드 (RPi)

```bash
cd ~/ros2_ws
colcon build --symlink-install --parallel-workers 1
source ~/ros2_ws/install/setup.bash
```

### Step 5: 테스트 실행 (RPi)

```bash
ros2 run temp_monitor cpu_temp_publisher
```

출력 예시:
```
[INFO] [1700000000.123456789] [cpu_temp_publisher]: CPU Temperature: 45.3°C
[INFO] [1700000001.123456789] [cpu_temp_publisher]: CPU Temperature: 45.5°C
```

---

## 7.4 구현: Remote PC Subscriber Node

### Step 1: PC에서 동일 패키지 작성

```bash
# PC에서도 같은 구조로 패키지 생성
mkdir -p ~/ros2_ws/src
cd ~/ros2_ws/src

# 패키지 생성
ros2 pkg create temp_monitor \
  --build-type ament_python \
  --dependencies rclpy std_msgs
```

### Step 2: Subscriber 노드 작성

`~/ros2_ws/src/temp_monitor/temp_monitor/temp_monitor.py`:

```python
#!/usr/bin/env python3
import rclpy
from rclpy.node import Node
from std_msgs.msg import Float64
import csv
import os
from datetime import datetime


class TempMonitor(Node):
    """
    /cpu_temp 토픽을 구독하여:
    1. 실시간 온도 출력
    2. 70°C 초과 시 경고
    3. CSV 파일 저장
    """

    def __init__(self):
        super().__init__('temp_monitor')
        self.subscription = self.create_subscription(
            Float64,
            '/cpu_temp',
            self.temp_callback,
            10
        )
        self.warning_threshold = 70.0
        self.critical_threshold = 80.0
        self.temp_history = []
        self.csv_filename = f'cpu_temp_log_{datetime.now().strftime("%Y%m%d_%H%M%S")}.csv'

        # CSV 헤더 작성
        self.init_csv()

        self.get_logger().info('🌡️  Temperature Monitor started!')
        self.get_logger().info(f'Warning threshold: {self.warning_threshold}°C')
        self.get_logger().info(f'Critical threshold: {self.critical_threshold}°C')
        self.get_logger().info(f'Logging to: {self.csv_filename}')

    def init_csv(self):
        """CSV 파일 초기화 (헤더 작성)"""
        try:
            with open(self.csv_filename, 'w', newline='') as f:
                writer = csv.writer(f)
                writer.writerow(['timestamp', 'temperature_celsius'])
        except Exception as e:
            self.get_logger().error(f'CSV init failed: {e}')

    def temp_callback(self, msg):
        """온도 데이터 수신 시 처리"""
        temp = msg.data
        timestamp = datetime.now().strftime('%H:%M:%S')

        # CSV 저장
        try:
            with open(self.csv_filename, 'a', newline='') as f:
                writer = csv.writer(f)
                writer.writerow([datetime.now().isoformat(), f'{temp:.2f}'])
        except Exception as e:
            self.get_logger().error(f'CSV write failed: {e}')

        # 온도에 따른 상태 표시
        if temp >= self.critical_threshold:
            icon = '🔥🔥🔥'
            level = 'CRITICAL'
        elif temp >= self.warning_threshold:
            icon = '⚠️'
            level = 'WARNING'
        else:
            icon = '✅'
            level = 'NORMAL'

        # 10회마다 통계 출력
        self.temp_history.append(temp)
        if len(self.temp_history) >= 10:
            avg_temp = sum(self.temp_history) / len(self.temp_history)
            max_temp = max(self.temp_history)
            min_temp = min(self.temp_history)
            self.get_logger().info(
                f'{icon} [{timestamp}] {level} - Current: {temp:.1f}°C | '
                f'Avg: {avg_temp:.1f}°C | Min: {min_temp:.1f}°C | Max: {max_temp:.1f}°C'
            )
            self.temp_history.clear()
        else:
            self.get_logger().info(
                f'{icon} [{timestamp}] Temp: {temp:.1f}°C ({level})'
            )

        if temp >= self.warning_threshold:
            self.get_logger().warn(
                f'{icon} HIGH TEMPERATURE DETECTED: {temp:.1f}°C! '
                f'Check RPi cooling!'
            )


def main(args=None):
    rclpy.init(args=args)
    node = TempMonitor()
    try:
        rclpy.spin(node)
    except KeyboardInterrupt:
        node.get_logger().info('Temperature Monitor shutting down...')
    finally:
        node.destroy_node()
        rclpy.shutdown()


if __name__ == '__main__':
    main()
```

### Step 3: 빌드 (PC)

```bash
cd ~/ros2_ws
colcon build --symlink-install
source ~/ros2_ws/install/setup.bash
```

---

## 7.5 실행 & 테스트

### Step 1: RPi에서 Publisher 실행

```bash
# RPi SSH
source ~/ros2_ws/install/setup.bash
ros2 run temp_monitor cpu_temp_publisher
```

### Step 2: PC에서 Subscriber 실행

```bash
# PC 터미널
source ~/ros2_ws/install/setup.bash
ros2 run temp_monitor temp_monitor
```

### Step 3: rqt_plot으로 그래프 보기

```bash
# PC에서
rqt_plot /cpu_temp
```

### Step 4: 부하 테스트

RPi에서 CPU 부하를 주어 온도 변화 확인:

```bash
# RPi에서 새로운 SSH 세션
sudo apt install -y stress
stress --cpu 4 --timeout 60 &

# 온도 모니터링 노드가 어떻게 반응하는지 관찰
# 약 30-60초 후 온도 상승 확인
```

> **예상 결과:** idle 45-55°C → stress 후 70-80°C (방열판/팬 상태에 따라 다름)

---

## 7.6 확장 아이디어

### 아이디어 1: 통계 노드 추가

```python
# 5분, 30분, 1시간 평균 온도를 별도 토픽으로 발행
/average_temp_5min
/average_temp_30min
/max_temp_today
```

### 아이디어 2: 알림 시스템

```python
# 온도가 75°C 초과 시 PC 바탕화면 알림
import subprocess
subprocess.run(['notify-send', 'RPi Temperature Warning!', f'{temp}°C'])
```

### 아이디어 3: 팬 제어

```python
# RPi GPIO로 냉각 팬 자동 제어
if temp > 70:
    GPIO.output(FAN_PIN, GPIO.HIGH)  # 팬 ON
elif temp < 50:
    GPIO.output(FAN_PIN, GPIO.LOW)   # 팬 OFF
```

### 아이디어 4: Web Dashboard

```python
# Flask나 FastAPI로 웹 대시보드에 온도 표시
# /cpu_temp 토픽 subscribe → WebSocket으로 브라우저 전송
```

---

## 7.7 프로젝트 회고

### 배운 점

- ROS2 Publisher/Subscriber 패턴을 실제 문제에 적용
- 시스템 파일 읽기와 ROS2 메시지 변환
- 분산 시스템에서 RPi↔PC 통신
- CSV 로깅으로 데이터 지속성 확보

### 제출 요구사항

```
📁 temp_monitor/
├── package.xml
├── setup.py
├── temp_monitor/
│   ├── __init__.py
│   ├── cpu_temp_publisher.py    # RPi 노드
│   └── temp_monitor.py          # PC 노드
└── logs/
    └── cpu_temp_log_*.csv       # 저장된 온도 로그
```

---

## 📝 심화 연습 문제

1. **온도 히스토그램:** 1시간치 온도 데이터를 수집하고, 온도 구간별 빈도를 계산하는 분석 노드를 만드세요
2. **네트워크 지연 측정:** CPU 온도와 함께 publish-to-subscribe 지연 시간(latency)도 측정하여 토픽에 포함시키세요
3. **이중 Publisher:** 온도가 갑자기 10°C 이상 변하면 `/temp_event` (String, "급격한 온도 변화 감지")를 별도로 publish하는 노드를 추가하세요
4. **Launch 파일:** publisher + monitor + rqt_plot을 한 번에 실행하는 launch 파일을 작성하세요
5. **Lifecycle 노드:** ROS2 Lifecycle 노드로 변환하여, 온도가 안전 범위일 때만 active 상태가 되도록 구현해보세요

---

## 🎉 Week 1 완료!

축하합니다! 7일 동안 ROS2의 핵심 개념을 모두 학습하고 실제 미니 프로젝트까지 완료했습니다.

**Week 1에서 배운 것:**
- ✅ Raspberry Pi 4B + Ubuntu Server + ROS2 Humble 설치
- ✅ Remote PC 환경 구축
- ✅ ROS2 노드, 토픽, 서비스, 액션, 파라미터
- ✅ Python/C++ Publisher/Subscriber 노드 작성
- ✅ Launch 파일 작성
- ✅ DDS 통신 이해
- ✅ 미니 프로젝트: CPU 온도 모니터링

**이제 Week 2로 넘어가서 실제 TurtleBot3를 구동해봅시다! 🚀**
