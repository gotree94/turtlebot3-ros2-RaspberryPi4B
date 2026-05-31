# Day 21 — 최종 프로젝트 완성 & 발표

> **목표:** 전체 시스템을 통합하고, 최종 데모를 준비하며, 3주 과정을 마무리한다.

---

## 21.1 최종 체크리스트

### 하드웨어

```
□ TurtleBot3 Burger 완전 조립 및 배터리 충전
□ Raspberry Pi 4B 장착 및 방열판 부착
□ MicroSD 카드 여유 공간 확인 (df -h)
□ OpenCR, LDS USB 연결 확인
□ 모든 케이블 정리
□ 테스트 공간 장애물 배치 완료
```

### 소프트웨어

```
□ RPi: ROS2 Humble ros-base 정상 동작
□ RPi: turtlebot3_bringup 정상 실행
□ PC: ROS2 Humble desktop 정상 동작
□ PC: SLAM 지도 생성 완료 (my_map.yaml)
□ PC: Nav2 자율주행 사전 테스트 완료
□ PC: patrol_bot 패키지 빌드 완료
□ Waypoint YAML 파일 최종 확정
```

### 네트워크

```
□ RPi ↔ PC 동일 WiFi 연결
□ RPi 고정 IP 또는 hostname 확인
□ ROS_DOMAIN_ID=30 양쪽 일치
□ ping 지연 5ms 이하
```

---

## 21.2 최종 데모 시나리오

### 데모 1: SLAM 매핑 (선택)

```
목표: 새로운 환경에서 지도 생성
절차:
  1. RPi: ros2 launch turtlebot3_bringup robot.launch.py
  2. PC: slam_toolbox 실행
  3. 텔레옵으로 환경 탐색 (약 5분)
  4. 지도 저장 (map_saver_cli)
시간: 약 7분
```

### 데모 2: 자율 순찰 (메인)

```
목표: 3개의 Waypoint를 순찰하는 자율주행
절차:
  1. RPi: Bringup 실행
  2. PC: ros2 launch patrol_bot patrol.launch.py
  3. RViz2에서 "2D Pose Estimate"로 초기 위치 설정
  4. Waypoint Navigator가 자동으로 순찰 시작
  5. RViz2에서 경로 주행 관찰
  6. Patrol Monitor의 상태 메시지 확인
  7. 모든 Waypoint 도착 확인
시간: 약 5분 (거리에 따라 변동)
```

### 데모 3: 장애물 대응 (선택)

```
목표: 순찰 중 장애물 회피
절차:
  1. 순찰 실행 중 로봇 경로 위에 장애물 배치
  2. Nav2가 장애물을 회피하는 경로 재계획
  3. Patrol Monitor가 장애물 회피 로그 기록
시간: 약 3분
```

---

## 21.3 최종 실행 명령어 정리

### 터미널 구성

```
┌─────────────────────────────────┐
│  Terminal 1: RPi SSH (Bringup)  │
│  $ ros2 launch turtlebot3_      │
│      bringup robot.launch.py    │
├─────────────────────────────────┤
│  Terminal 2: PC (Nav2 + Patrol) │
│  $ ros2 launch patrol_bot       │
│      patrol.launch.py           │
├─────────────────────────────────┤
│  Terminal 3: PC (RViz2)         │
│  $ rviz2                        │
├─────────────────────────────────┤
│  Terminal 4: PC (모니터링)      │
│  $ ros2 topic echo /patrol_status│
└─────────────────────────────────┘
```

### Quick-start 스크립트

`~/patrol_bot/start_patrol.sh`:

```bash
#!/bin/bash
# TurtleBot3 자율 순찰 빠른 시작 스크립트
# PC에서 실행

set -e

echo "╔══════════════════════════════════════╗"
echo "║   TurtleBot3 Autonomous Patrol       ║"
echo "╚══════════════════════════════════════╝"

# 환경 변수
export TURTLEBOT3_MODEL=burger
export ROS_DOMAIN_ID=30

# 사전 체크
echo "🔍 Checking environment..."
echo "ROS_DOMAIN_ID: $ROS_DOMAIN_ID"
echo "TURTLEBOT3_MODEL: $TURTLEBOT3_MODEL"

# RPi 연결 확인
if ! ping -c 1 -W 1 turtlebot-pi.local &> /dev/null; then
    echo "❌ Cannot reach turtlebot-pi.local. Check WiFi and RPi status."
    exit 1
fi
echo "✅ RPi connected"

# 맵 파일 확인
MAP_FILE=~/my_map.yaml
if [ ! -f "$MAP_FILE" ]; then
    echo "⚠️  Map file not found: $MAP_FILE"
    echo "   Please complete mapping first (Day 16)"
    exit 1
fi

echo "✅ Map file found: $MAP_FILE"

# Waypoint 확인
WP_FILE=~/patrol_bot/config/waypoints.yaml
if [ ! -f "$WP_FILE" ]; then
    echo "⚠️  Waypoint file not found: $WP_FILE"
    echo "   Creating sample waypoints..."
    mkdir -p ~/patrol_bot/config
    cat > "$WP_FILE" << 'EOF'
waypoints:
  - name: "Start"
    position: {x: 0.0, y: 0.0, z: 0.0}
    orientation: {yaw: 0.0}
    actions: [{type: "wait", duration: 2.0}, {type: "log", message: "Starting patrol"}]
  - name: "Point_A"
    position: {x: 1.0, y: 1.0, z: 0.0}
    orientation: {yaw: 90.0}
    actions: [{type: "wait", duration: 3.0}, {type: "log"}]
  - name: "Point_B"
    position: {x: -0.5, y: 1.5, z: 0.0}
    orientation: {yaw: -90.0}
    actions: [{type: "wait", duration: 3.0}, {type: "log"}]
patrol_config:
  repeat_count: 2
  wait_between_cycles: 5
  battery_check_interval: 5
EOF
fi
echo "✅ Waypoints loaded"

echo ""
echo "🚀 Starting patrol system..."
echo ""
echo "❗ Instructions:"
echo "   1. Wait for Nav2 to start"
echo "   2. Set initial pose in RViz2 (2D Pose Estimate)"
echo "   3. Watch the autonomous patrol begin!"
echo ""

# 패트롤 실행
ros2 launch patrol_bot patrol.launch.py
```

---

## 21.4 예상 문제 및 대처

| 문제 | 증상 | 대처 |
|------|------|------|
| **Nav2 시작 실패** | "No map received" | `map_server`가 맵 파일을 찾는지 확인 |
| **AMCL 위치 이탈** | 로봇이 엉뚱한 곳에 있다고 인식 | RViz2에서 2D Pose Estimate 재설정 |
| **Waypoint 도달 불가** | Nav2가 경로를 찾지 못함 | Waypoint가 장애물 영역에 없는지 확인 |
| **배터리 부족** | 순찰 중 갑자기 멈춤 | 배터리 충전 후 재시작 (충분히 충전했는지 확인) |
| **WiFi 끊김** | RPi와 PC 통신 두절 | WiFi 공유기와 가까운 위치에서 테스트 |
| **LIDAR 데이터 이상** | /scan에서 nan 또는 inf | LDS 연결 상태 확인, USB 케이블 재연결 |

---

## 21.5 프로젝트 확장 아이디어

이 프로젝트를 더 발전시킬 수 있는 아이디어들:

### 1) 카메라 기반 객체 탐지

```python
# USB 카메라로 순찰 중 객체 탐지
# YOLO 또는 OpenCV로 사람/물체 인식
# 탐지 결과를 /patrol_status에 포함
```

### 2) Web Dashboard

```python
# Flask/FastAPI 웹 서버
# 실시간 지도 + 로봇 위치 표시
# Waypoint 설정 Web UI
# 원격에서 순찰 시작/중지
```

### 3) Telegram/Discord 알림

```python
# 순찰 완료, 배터리 부족 등을 메시지로 전송
# 원격 모니터링 가능
```

### 4) 다중 Waypoint 최적화

```python
# Traveling Salesman Problem (TSP) 알고리즘
# 최단 경로로 Waypoint 방문 순서 최적화
```

### 5) 자동 충전 도킹

```python
# 배터리 부족 시 충전소 위치로 자동 귀환
# 충전 후 순찰 재개
# IR 센서 또는 AprilTag로 도킹
```

### 6) 패트롤 스케줄링

```python
# 특정 시간에 자동 순찰 시작 (cron)
# 순찰 이력을 데이터베이스에 저장
# 주간/야간 모드 전환
```

---

## 21.6 3주 과정 회고

### 배운 내용 총정리

```
Week 1: ROS2 기초
  ✅ Raspberry Pi 4B + Ubuntu Server 설치
  ✅ ROS2 Humble 설치 및 설정
  ✅ ROS2 5대 개념 (Node, Topic, Service, Action, Parameter)
  ✅ 첫 ROS2 패키지 작성 (Python + C++)
  ✅ DDS 통신 이해 (ROS_DOMAIN_ID, RMW)

Week 2: TurtleBot3 제어
  ✅ TurtleBot3 패키지 설치 및 빌드
  ✅ Bringup & Teleoperation
  ✅ LIDAR, Odometry, IMU, TF 트리
  ✅ RViz2 시각화
  ✅ PID 제어 및 Go-to-Goal
  ✅ URDF 구조 이해
  ✅ LIDAR 장애물 회피

Week 3: SLAM + Navigation + 최종 프로젝트
  ✅ SLAM 이론 (EKF, FastSLAM, GraphSLAM)
  ✅ slam_toolbox로 실제 환경 매핑
  ✅ Navigation2 (Nav2) 아키텍처 이해
  ✅ AMCL 위치 추정
  ✅ Nav2 자율주행 및 파라미터 튜닝
  ✅ 자율 순찰 로봇 시스템 구축
```

### 추가 학습 방향

| 분야 | 추천 리소스 |
|------|------------|
| ROS2 심화 | ROS2 공식 튜토리얼, ROS2 Design 문서 |
| 로봇 공학 | Modern Robotics (Lynch & Park), Robotics, Vision and Control (Corke) |
| SLAM 심화 | Probabilistic Robotics (Thrun, Burgard, Fox) |
| 컴퓨터 비전 | OpenCV ROS2 패키지, YOLO + ROS2 |
| 강화 학습 | ROS2 + Gym, Stable Baselines3 |

---

## 21.7 최종 제출 체크리스트

### GitHub 저장소 구성

```
📦 turtlebot3-ros2-curriculum/
├── README.md                    # 프로젝트 개요
├── week1/                       # 1주차 교육 자료
├── week2/                       # 2주차 교육 자료
├── week3/                       # 3주차 교육 자료
├── exercises/                   # 연습문제
├── appendices/                  # 부록
└── src/                         # (선택) 코드 저장소
    └── patrol_bot/
        ├── patrol_bot/
        │   ├── waypoint_navigator.py
        │   ├── battery_monitor.py
        │   └── patrol_monitor.py
        ├── launch/
        │   └── patrol.launch.py
        ├── config/
        │   └── waypoints.yaml
        ├── package.xml
        └── setup.py
```

### 데모 영상 체크리스트

```
□ 1. 시스템 부팅 (RPi + PC)
□ 2. SLAM 매핑 과정 (선택)
□ 3. 자율 순찰 실행 (3개 Waypoint)
□ 4. 장애물 회피 (선택)
□ 5. Patrol Monitor 상태 출력
□ 6. 최종 보고서 출력
```

### 발표 자료 (선택)

```
□ 시스템 아키텍처 슬라이드
□ 주요 구현 사항
□ 시연 영상
□ 트러블슈팅 경험 공유
□ 향후 개선 방향
```

---

## 21.8 마무리

축하합니다! 🎉 **3주 동안 TurtleBot3 Burger + Raspberry Pi 4B + ROS2 Humble** 환경에서 SLAM, Navigation, 자율주행까지 완전히 마스터했습니다.

### 이제 당신은:

- ✅ ROS2 시스템을 설치하고 설정할 수 있습니다
- ✅ ROS2 노드, 토픽, 서비스, 액션을 자유자재로 다룹니다
- ✅ TurtleBot3를 bringup하고 센서 데이터를 분석합니다
- ✅ SLAM으로 환경 지도를 생성합니다
- ✅ Navigation2로 자율주행을 구현합니다
- ✅ Waypoint 기반 자율 순찰 시스템을 구축합니다

### 이 다음 단계:

이 커리큘럼의 지식을 기반으로:
- **멀티 로봇 시스템**으로 확장
- **카메라 기반 인식** 추가 (AprilTag, YOLO)
- **클라우드 연동** (AWS RoboMaker, Azure)
- **실제 산업 현장 적용** (물류, 검사, 순찰)

---

> **"로봇 공학의 길은 끝이 없습니다. 이제 첫걸음을 떼었습니다."**

---

## 📝 최종 점검 문제

1. **종합:** 시스템을 완전히 재부팅하고, 단계별로 모든 서비스를 시작하는 절차를 스크립트 없이 수동으로 수행해보세요
2. **문제 해결:** 순찰 중 "/odom" 토픽이 갑자기 끊겼습니다. 원인을 진단하고 해결하는 절차를 서술하세요
3. **확장 설계:** 4층 건물에서 층별로 순찰 로봇을 운영하려면 시스템을 어떻게 확장해야 할지 설계하세요
4. **최적화:** 10개의 Waypoint를 방문할 때, 최소 이동 거리를 계산하는 알고리즘을 설계하고 구현해보세요
5. **자기 평가:** 3주 동안 배운 내용 중 가장 어려웠던 부분과 가장 흥미로웠던 부분을 각각 3개씩 적고, 그 이유를 설명하세요
