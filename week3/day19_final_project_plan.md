# Day 19 — 최종 프로젝트: 자율 순찰 로봇 (기획 & 설계)

> **목표:** 3주 동안 배운 모든 지식을 종합하여 자율 순찰 로봇 시스템을 설계한다.

---

## 19.1 프로젝트 개요

**프로젝트명:** TurtleBot3 자율 순찰 로봇 (Autonomous Patrol Bot)

**목표:** 사용자가 설정한 여러 Waypoint를 순차적으로 방문하며 순찰하고, 이상 상황(장애물, 배터리 부족)에 대응하는 자율 로봇 시스템 구축

---

## 19.2 시스템 아키텍처

```
┌─────────────────────────────────────────────────────────┐
│                   Remote PC                              │
│                                                          │
│  ┌─────────────────────┐     ┌──────────────────────┐   │
│  │  Waypoint Manager   │     │  Patrol Monitor       │   │
│  │  (Node)             │     │  (Node)               │   │
│  │  - YAML waypoint 로드│     │  - 진행 상황 표시      │   │
│  │  - 순차적 goal 전송  │     │  - 통계 기록          │   │
│  │  - Nav2 API 호출    │     │  - 로그 저장          │   │
│  └─────────┬───────────┘     └──────────────────────┘   │
│            │                                             │
│  ┌─────────▼────────────────────────────────────────┐   │
│  │              Nav2 Stack                          │   │
│  │  (Global Planner + Local Planner + AMCL + BT)   │   │
│  └─────────┬────────────────────────────────────────┘   │
│            │                                             │
│  ┌─────────▼────────────────────────────────────────┐   │
│  │              RViz2 (모니터링)                     │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────┬───────────────────────────────┘
                          │ ROS_DOMAIN_ID=30
                          │ (WiFi)
┌─────────────────────────▼───────────────────────────────┐
│                   TurtleBot3 (RPi)                       │
│                                                          │
│  ┌──────────────────────────────────────────────────┐   │
│  │  turtlebot3_bringup (robot.launch.py)             │   │
│  │  - OpenCR (모터 제어)                              │   │
│  │  - LDS (LIDAR)                                    │   │
│  │  - Odometry                                       │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

---

## 19.3 노드 설계

### 1) Waypoint Navigator Node

**역할:** Waypoint를 순차적으로 방문

```python
# 의사 코드
class WaypointNavigator(Node):
    def __init__(self):
        # YAML에서 waypoint 로드
        self.waypoints = load_waypoints('waypoints.yaml')
        self.current_wp = 0
        self.nav_client = Nav2Client()  # Nav2 액션 클라이언트

    def start_patrol(self):
        while self.current_wp < len(self.waypoints):
            target = self.waypoints[self.current_wp]
            result = self.nav_client.send_goal(target)
            if result.success:
                self.log(f"Waypoint {self.current_wp} reached")
                self.current_wp += 1
                self.wait_at_waypoint(2.0)  # 2초 대기
            else:
                self.handle_failure(result)

    def handle_failure(self, result):
        if result == 'blocked':
            self.replan_or_skip()
        elif result == 'battery_low':
            self.return_to_charger()
```

### 2) Patrol Monitor Node

**역할:** 순찰 상태 모니터링 및 기록

```python
class PatrolMonitor(Node):
    def __init__(self):
        self.sub_battery = create_subscription(/battery_state, ...)
        self.sub_odom = create_subscription(/odom, ...)
        self.stats = {
            'total_distance': 0.0,
            'waypoints_visited': 0,
            'obstacles_avoided': 0,
            'start_time': None,
            'battery_history': []
        }

    def log_status(self):
        # 10초마다 상태 출력
        pass

    def generate_report(self):
        # 최종 보고서 생성
        pass
```

### 3) Battery Monitor Node

**역할:** 배터리 상태 감시 및 귀환 결정

```python
class BatteryMonitor(Node):
    def __init__(self):
        self.low_threshold = 11.0  # 11V 이하 → 충전소 귀환
        self.critical_threshold = 10.5  # 10.5V 이하 → 비상 정지

    def check_battery(self):
        voltage = self.get_battery_voltage()
        if voltage < self.critical_threshold:
            self.emergency_stop()
        elif voltage < self.low_threshold:
            self.request_return_to_charger()
```

---

## 19.4 Waypoint 파일 형식

```yaml
# waypoints.yaml
waypoints:
  - name: "Entry Door"
    position:
      x: 1.5
      y: 2.0
      z: 0.0
    orientation:
      yaw: 0.0  # 도착 후 바라볼 방향 (degrees)
    actions:
      - type: "wait"
        duration: 3.0  # 3초 대기
      - type: "scan"     # 주변 스캔 기록
      - type: "log"      # 로그 출력

  - name: "Kitchen"
    position:
      x: 3.0
      y: -1.0
      z: 0.0
    orientation:
      yaw: 90.0
    actions:
      - type: "wait"
        duration: 5.0
      - type: "log"

  - name: "Living Room"
    position:
      x: -1.0
      y: 2.5
      z: 0.0
    orientation:
      yaw: 180.0
    actions:
      - type: "wait"
        duration: 3.0
      - type: "log"

  - name: "Charging Station"
    position:
      x: 0.0
      y: 0.0
      z: 0.0
    orientation:
      yaw: 0.0
    actions:
      - type: "dock"  # 충전소 도킹 (선택)

patrol_config:
  repeat_count: 3          # 순찰 반복 횟수 (0 = 무한)
  wait_between_cycles: 10  # 순환 간 대기 시간 (초)
  battery_check_interval: 5  # 배터리 체크 주기 (초)
  log_file: "patrol_log.csv"
```

---

## 19.5 개발 일정

### Day 19: 기획 및 설계 (오늘)

- [ ] 시스템 아키텍처 설계 완료
- [ ] Waypoint 파일 작성
- [ ] 각 노드의 인터페이스 정의
- [ ] 실패 조건 정의 (배터리, 장애물, 경로 이탈)

### Day 20: 핵심 기능 구현

- [ ] Waypoint Navigator Node 구현
- [ ] Patrol Monitor Node 구현
- [ ] Battery Monitor Node 구현
- [ ] 통합 테스트
- [ ] 디버깅 및 수정

### Day 21: 최종 완성

- [ ] 전체 시스템 통합
- [ ] Launch 파일 작성 (단일 명령으로 실행)
- [ ] 최종 데모
- [ ] 문서화

---

## 19.6 사전 준비

### 필요한 패키지

```bash
# PC에서
sudo apt install -y \
  ros-humble-nav2-simple-commander \
  ros-humble-nav2-waypoint-follower \
  python3-yaml
```

### Waypoint 디렉토리

```bash
mkdir -p ~/patrol_bot/config
mkdir -p ~/patrol_bot/logs
mkdir -p ~/patrol_bot/launch
```

---

## 📝 기획 문제

1. **Waypoint 설계:** 자신의 방/사무실 기준으로 5개 이상의 Waypoint를 설정하고 YAML 파일을 작성하세요
2. **실패 시나리오:** 다음 상황에서 로봇이 어떻게 대응해야 하는지 설계하세요
   - 배터리가 11V 이하일 때
   - Waypoint 경로가 완전히 막혔을 때
   - LIDAR 통신이 끊겼을 때
3. **효율성:** 10개의 방이 있는 사무실에서 최소 이동 거리로 모든 방을 순찰하는 경로를 설계하세요
4. **안전:** 순찰 중 사람이 로봇 앞에 갑자기 나타났을 때의 안전 로직을 설계하세요
5. **확장:** (선택) USB 카메라를 추가하여 각 Waypoint에서 사진을 촬영하고 저장하는 기능을 설계에 포함하세요

---

## ⚠️ 주의사항

- **Nav2**를 사용하므로 매핑이 먼저 완료되어야 합니다 (Day 16)
- **AMCL 초기 위치 설정**이 정확하지 않으면 순찰이 실패합니다
- 배터리가 부족한 상태에서는 절대 긴 경로를 설정하지 마세요
- Waypoint는 Costmap의 lethal 영역을 피해서 설정하세요
- 첫 번째 테스트는 2-3개의 Waypoint로 간단히 시작하세요
