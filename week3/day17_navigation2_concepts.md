# Day 17 — Navigation2 (Nav2) 개념 & 설정

> **목표:** ROS2 Navigation2 (Nav2) 아키텍처를 이해하고, TurtleBot3의 Nav2 설정 방법을 학습한다.

---

## 17.1 Navigation2 개요

**Nav2** = ROS2의 공식 네비게이션 프레임워크
- 주어진 지도에서 출발지 → 목적지까지 안전하게 이동
- 동적 장애물 회피
- 행동 트리(Behavior Tree) 기반 태스크 관리

### Nav2 버전

| ROS2 버전 | Nav2 버전 |
|-----------|-----------|
| Humble | Nav2 1.x (안정) |

---

## 17.2 Nav2 아키텍처

```
┌─────────────────────────────────────────────────────┐
│                    Nav2 Stack                       │
├─────────────────────────────────────────────────────┤
│  ┌──────────────────┐  ┌────────────────────────┐   │
│  │ Behavior Trees   │  │ Lifecycle Manager      │   │
│  │ (BT Navigator)   │  │ (노드 lifecycle 관리)   │   │
│  └────────┬─────────┘  └────────────────────────┘   │
│           │                                          │
│  ┌────────▼─────────────────────────────────┐        │
│  │         Planners                         │        │
│  │  ┌────────────┐  ┌────────────────┐     │        │
│  │  │ Global     │  │ Local          │     │        │
│  │  │ Planner    │  │ Planner        │     │        │
│  │  │ (전체 경로)│  │ (국부 경로)   │     │        │
│  │  └────────────┘  └────────────────┘     │        │
│  └──────────────────────────────────────────┘        │
│           │                                          │
│  ┌────────▼─────────────────────────────────┐        │
│  │         Controllers                       │        │
│  │  ┌────────────┐  ┌────────────────┐     │        │
│  │  │ DWB        │  │ 추종 제어기    │     │        │
│  │  │ Controller │  │               │     │        │
│  │  └────────────┘  └────────────────┘     │        │
│  └──────────────────────────────────────────┘        │
│           │                                          │
│  ┌────────▼─────────────────────────────────┐        │
│  │         Costmaps                          │        │
│  │  ┌────────────┐  ┌────────────────┐     │        │
│  │  │ Global     │  │ Local          │     │        │
│  │  │ Costmap    │  │ Costmap        │     │        │
│  │  └────────────┘  └────────────────┘     │        │
│  └──────────────────────────────────────────┘        │
│           │                                          │
│  ┌────────▼─────────────────────────────────┐        │
│  │         Recovery Behaviors               │        │
│  │  회전, 후진, 경로 재계획 등              │        │
│  └──────────────────────────────────────────┘        │
└─────────────────────────────────────────────────────┘
```

---

## 17.3 핵심 구성 요소

### 1) Global Planner (전역 경로 계획)

- **역할:** 시작점 → 목적지까지의 최적 경로를 지도상에서 계산
- **알고리즘:** NavFn (Dijkstra/A* hybrid)
- **출력:** `nav_msgs/Path` (전역 경로)
- **특징:** Costmap 기반, 정적 장애물 고려

```bash
# 사용 가능한 Global Planner 확인
ros2 pkg list | grep planner
```

### 2) Local Planner (지역 경로 계획)

- **역할:** Global Planner의 경로를 따라가며 동적 장애물 회피
- **알고리즘:** DWB (Dynamic Window Approach 기반)
- **출력:** `geometry_msgs/Twist` (속도 명령)
- **특징:** 실시간 센서 데이터 반영, 동적 장애물 즉시 회피

```bash
# DWB 파라미터 확인
ros2 param describe /local_costmap.local_costmap plugins
```

### 3) Costmap (비용 지도)

- **역할:** 장애물을 비용(cost)으로 변환한 2D 그리드 맵
- **종류:** Global Costmap, Local Costmap
- **레이어:**
  - **Static Layer:** SLAM 지도 기반 (영구 장애물)
  - **Obstacle Layer:** LIDAR 데이터 기반 (실시간 장애물)
  - **Inflation Layer:** 장애물 주변으로 안전 영역 확장

```
비용 값 의미:
  0     = Free space (자유 공간)
  1-55  = Inflated (위험 구역, 접근 가능)
  56-99 = Lethal obstacle (장애물 근접)
  100   = Lethal (충돌 불가피)
  255   = Unknown (미탐색 영역)
```

### 4) AMCL (Adaptive Monte Carlo Localization)

- **역할:** 기존 지도 위에서 현재 로봇 위치 추정
- **방법:** 파티클 필터 (입자로 위치 확률 표현)
- **초기화:** "2D Pose Estimate"로 초기 위치 지정

```
파티클 분포가 의미하는 것:
  뭉쳐 있음 → 위치를 정확히 알고 있음
  퍼져 있음 → 위치 불확실성이 큼
  분리됨 → 여러 위치 가능성 (kidnapped robot 문제)
```

### 5) Behavior Tree (BT)

- **역할:** 네비게이션 전체 흐름 제어
- **구조:** XML 기반 트리 (Fallback, Sequence, Parallel 노드)

```xml
<!-- 단순 네비게이션 BT 예시 -->
<root>
  <BehaviorTree>
    <Sequence>
      <ComputePathToPose/>
      <FollowPath/>
    </Sequence>
  </BehaviorTree>
</root>
```

실제 Nav2 BT 구조:

```
NavigateToPose
├── PipelineSequence
│   ├── ComputePathToPose  ← Global Planner
│   ├── FollowPath          ← Local Planner + Controller
│   └── RecoveryNode        ← 복구 동작
│       ├── Spin            ← 제자리 회전
│       ├── Backup          ← 후진
│       └── ClearCostmap    ← Costmap 초기화
```

---

## 17.4 TurtleBot3 Nav2 패키지 구조

```bash
# TurtleBot3 Nav2 패키지 위치
ls /opt/ros/humble/share/turtlebot3_navigation2/

# 구성 파일들:
# param/
#   burger.yaml       ← Nav2 파라미터
# launch/
#   navigation2.launch.py   ← Nav2 실행 런치
#   rviz2.launch.py          ← RViz2 실행
# maps/
#   burger_house.yaml        ← 예제 맵
#   burger_house.pgm
```

### Params 파일 구조

```yaml
# burger.yaml 의 주요 섹션
controller_server:            # 컨트롤러 설정
local_costmap:
  local_costmap:              # 로컬 코스트맵
    ...
global_costmap:
  global_costmap:             # 글로벌 코스트맵
    ...
planner_server:               # 경로 계획기
    ...
behavior_tree_config:         # 행동 트리
    ...
bt_navigator:                 # BT 네비게이터
    ...
```

---

## 17.5 AMCL 파라미터

```yaml
amcl:
  ros__parameters:
    # 파티클 필터
    min_particles: 6          # 최소 파티클 수
    max_particles: 2000       # 최대 파티클 수
    particles: 500            # 기본 파티클 수
    
    # 위치 추정
    alpha1: 0.2               # 회전 중 회전 오차
    alpha2: 0.2               # 이동 중 회전 오차
    alpha3: 0.2               # 회전 중 이동 오차
    alpha4: 0.2               # 이동 중 이동 오차
    alpha5: 0.2               # 센서 오차
    
    # 초기 위치
    initial_pose_x: 0.0
    initial_pose_y: 0.0
    initial_pose_z: 0.0
    initial_pose_yaw: 0.0
    
    # 업데이트 간격
    update_min_d: 0.2         # 최소 이동 거리 (m)
    update_min_a: 0.2         # 최소 회전 각도 (rad)
    resample_interval: 1      # 리샘플링 간격
```

---

## 17.6 데모 실행

매핑 없이 Nav2 기본 데모:

```bash
# 방법 1: TurtleBot3 시뮬레이션에서 Nav2
ros2 launch turtlebot3_gazebo turtlebot3_world.launch.py

# 방법 2: 저장된 맵으로 Nav2 실행
ros2 launch turtlebot3_navigation2 navigation2.launch.py \
  map:=/opt/ros/humble/share/turtlebot3_navigation2/maps/burger_house.yaml

# RViz2가 자동 실행되며 "2D Pose Estimate"와 "2D Nav Goal" 사용 가능
```

---

## 📝 연습 문제

1. **구조도 작성:** Nav2 아키텍처의 모든 구성 요소와 그 관계를 블록 다이어그램으로 그리고, 각 요소의 역할을 한 줄로 설명하세요
2. **코스트맵 이해:** inflation layer의 radius가 0.1m, 0.3m, 0.5m일 때 각각의 차이점을 설명하세요
3. **AMCL 시나리오:** 로봇이 "kidnapped" (갑자기 다른 위치로 옮겨짐)되었을 때 AMCL이 어떻게 반응하는지 서술하세요
4. **BT 분석:** Nav2의 기본 Behavior Tree XML 파일을 찾아 읽고, 각 노드의 의미를 분석하세요
5. **Planner 비교:** NavFn의 Dijkstra와 A* 알고리즘의 차이점을 경로 품질과 계산 시간 측면에서 비교하세요

---

## ⚠️ 트러블슈팅

| 문제 | 해결 방법 |
|------|----------|
| "No map received" 에러 | `map_server`가 실행 중인지 확인 |
| AMCL 위치 수렴 안 됨 | 초기 위치를 더 정확히 지정, `max_particles` 증가 |
| Global Planner 경로 없음 | 목적지가 Costmap의 lethal 영역인지 확인 |
| 로봇이 빙글빙글 돔 | Local Planner 파라미터 튜닝 필요 |
| Nav2 노드가 실행 안 됨 | `ros2 lifecycle list`로 모든 노드의 상태 확인 |
