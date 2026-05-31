# Day 15 — SLAM 이론 & Cartographer

> **목표:** SLAM(Simultaneous Localization And Mapping)의 핵심 개념을 이해하고, ROS2에서 사용 가능한 SLAM 도구를 학습한다.

---

## 15.1 SLAM이란?

**SLAM** = Simultaneous Localization And Mapping
- **Localization:** "지금 나는 어디에 있는가?" (위치 추정)
- **Mapping:** "내 주변은 어떻게 생겼는가?" (환경 지도 작성)
- **동시에** 이 두 문제를 해결하는 것이 SLAM

### 닭과 달걀 문제

```
지도 ←→ 위치
  ▲       ▲
  │       │
  └── 의존 ──┘
```

- 위치를 알려면 지도가 필요하다
- 지도를 만들려면 위치를 알아야 한다
→ **SLAM은 이 순환 문제를 동시에 해결한다**

---

## 15.2 SLAM 알고리즘 분류

### 1) EKF SLAM (Extended Kalman Filter)

- **원리:** 로봇의 자세 + 랜드마크 위치를 하나의 상태 벡터로 관리
- **장점:** 계산량이 적음, 실시간성 좋음
- **단점:** 선형화 오차, 특징점 기반 (LIDAR에는 부적합)
- **사용:** 소형 로봇, 제한된 환경

### 2) FastSLAM (Particle Filter)

- **원리:** 파티클(입자)로 로봇 자세의 확률 분포 표현
- **장점:** 비선형 시스템에 강함, 멀티모달 분포 표현 가능
- **단점:** 파티클 수에 따라 성능 좌우 (너무 많으면 느림)
- **사용:** 소규모 환경

### 3) GraphSLAM / Pose Graph SLAM

- **원리:** 로봇의 자세를 그래프 노드로, 관측을 엣지로 표현
  - **노드(Node):** 로봇의 자세 (pose)
  - **엣지(Edge):** 노드 간의 상대 변환 (odometry, loop closure)
- **장점:** 전체 지도를 최적화 (global optimization), 대규모 환경에 적합
- **단점:** 계산량이 많지만 SPA(Sparse Pose Adjustment)로 최적화
- **사용:** TurtleBot3 + Cartographer (GraphSLAM 기반)

### 비교표

| 특성 | EKF SLAM | FastSLAM | GraphSLAM |
|------|----------|----------|-----------|
| 정확도 | 중간 | 좋음 | 매우 좋음 |
| 대규모 환경 | 부적합 | 보통 | 적합 |
| 계산량 | 적음 | 중간 | 많음 (최적화 필요) |
| Loop Closure | 어려움 | 가능 | 핵심 기능 |
| ROS2 도구 | robot_localization | - | Cartographer, slam_toolbox |

---

## 15.3 GraphSLAM 상세

```
실제 경로:  A ── B ── C ── D ── E
            │              │
            └── F ── G ────┘  (loop closure)

그래프:
    ┌─── odometry edge ──┐
    A ── B ── C ── D ──── E
    │                     │
    └── loop closure ─────┘  ← 같은 장소 인식!
```

### 작동 과정

1. **Odometry Edge:** 바퀴 엔코더로 추정한 로봇 이동 (단기적 정확, 장기적 오차)
2. **Observation Edge:** LIDAR로 관측한 주변 환경
3. **Loop Closure Edge:** 같은 장소를 재방문했을 때 오차 보정

→ 모든 엣지의 오차를 최소화하는 최적화 수행 (SPA: Sparse Pose Adjustment)

---

## 15.4 SLAM Toolbox (ROS2)

` slam_toolbox`는 ROS2에서 널리 사용되는 SLAM 라이브러리입니다.

```bash
# 설치
sudo apt install -y ros-humble-slam-toolbox
```

### 주요 특징

- **online_async:** 매핑 중에도 데이터를 비동기로 처리 (기본 모드)
- **online_sync:** 동기 모드 (정확도 높음, 속도 느림)
- **offline:** 저장된 bag 파일로 매핑
- **localization_only:** 기존 지도에서 위치 추정만 (매핑 없음)
- **Serialize/Deserialize:** 지도 저장/로딩

### Cartographer (Google)

Google이 개발한 실시간 SLAM 라이브러리:

```bash
# 설치
sudo apt install -y ros-humble-cartographer ros-humble-cartographer-ros
```

**Cartographer 특징:**
- 2D/3D LIDAR 모두 지원
- Submap 기반의 Scan-to-Map 매칭
- IMU + Odometry + LIDAR 센서 융합
- 실시간 Loop Closure
- TurtleBot3와 완벽 호환 (공식 지원)

---

## 15.5 TurtleBot3에서 SLAM 실행 구조

```
┌──────────┐    /scan     ┌──────────────┐    /map      ┌──────────┐
│ TurtleBot3│───────────►│ SLAM Toolbox │────────────►│   RViz2  │
│ (LIDAR)   │            │ Cartographer │    지도      │ (지도 표시)│
│           │    /tf     │              │              │          │
│ (Odometry)│───────────►│ /map → /odom │              │          │
│           │            │  TF 변환     │              │          │
└───────────┘            └──────────────┘              └──────────┘
     ▲                                                        │
     │                    /cmd_vel                             │
     └────────────────────────────────────────────────────────┘
                    (teleop_keyboard)
```

### 필요한 데이터

SLAM 정확도를 위해 필요한 입력:

```
✅ LIDAR scan data   (/scan)     ← 필수
✅ Odometry          (/odom)     ← 필수 (바퀴 엔코더)
✅ TF                (/tf)       ← 필수 (base_scan → base_link)
✅ IMU               (/imu)      ← 선택 (정확도 향상)
```

---

## 15.6 SLAM 성능에 영향을 미치는 요소

| 요소 | 영향 | 권장사항 |
|------|------|---------|
| **주행 속도** | 너무 빠르면 scan mismatch | 0.1~0.15 m/s로 천천히 |
| **Loop Closure** | 같은 공간 재방문 시 정확도↑ | 최소 1회 재방문 |
| **조명** | LIDAR는 무관하나 카메라는 영향 큼 | 실내 고정 조명 권장 |
| **바닥 재질** | 바퀴 슬립 → odometry 오차 | 단단한 평지 권장 |
| **LIDAR 높이** | 너무 낮으면 다리만, 너무 높으면 벽 상단 | TurtleBot3 기본값으로 충분 |
| **공간 크기** | 30평 이하: 5분, 100평: 15-20분 | 크면 천천히 여러 번 왕복 |

---

## 15.7 주요 SLAM 파라미터

### slam_toolbox 파라미터

```yaml
# burger.yaml (중요 파라미터)
slam_toolbox:
  ros__parameters:
    # Solver
    solver_plugin: "ceres_linear_solver"
    
    # Map parameters
    map_file_name: ""
    map_start_at_dock: false
    
    # Scan matcher
    minimum_travel_distance: 0.1    # 최소 이동 거리 (m)
    minimum_travel_heading: 0.1     # 최소 회전 각도 (rad)
    minimum_travel_distance_resubmit: 0.05
    scan_buffer_size: 10            # 스캔 버퍼 크기
    scan_buffer_maximum_scan_distance: 5.0
    
    # Loop closure
    loop_search_maximum_distance: 3.0
    loop_match_minimum_chain_size: 10
    loop_match_maximum_variance_skip: 5
    do_loop_closing: true
    loop_closure_duration: 1000
    
    # Correlation parameters
    correlation_search_space_dimension: 0.5
    correlation_search_space_resolution: 0.01
    correlation_search_space_smear_deviation: 0.03
```

---

## 📝 연습 문제

1. **SLAM 분류:** EKF SLAM, FastSLAM, GraphSLAM의 차이점을 장단점 위주로 A4 용지 1장에 요약하세요
2. **GraphSLAM 그래프:** 10m x 10m 정사각형 경로를 주행할 때 GraphSLAM의 노드와 엣지를 그림으로 표현하세요
3. **Cartographer 문서:** Cartographer의 ROS2 위키 페이지에서 파라미터 설명을 읽고, 중요 파라미터 5개를 선정하여 설명하세요
4. **SLAM vs Navigation:** SLAM과 Navigation의 차이점을 설명하고, 각각 필요한 센서 데이터가 무엇인지 기술하세요
5. **Loop Closure:** Loop Closure가 필요한 이유를 설명하고, Loop Closure 실패 시 어떤 일이 발생하는지 예측하세요

---

## ⚠️ 트러블슈팅

| 문제 | 해결 방법 |
|------|----------|
| SLAM 지도가 흐릿함 | 이동 속도를 줄이고(0.1 m/s), 동일 경로 재방문 |
| Loop Closure 안 됨 | `loop_search_maximum_distance` 증가, 천천히 재방문 |
| Cartographer 크래시 | `ros-humble-cartographer`와 `ros-humble-cartographer-ros` 재설치 |
| 지도가 뒤틀림 | odometry drift 의심, `minimum_travel_distance` 감소 |
| SLAM 시작 안 됨 | `/scan`, `/odom`, `/tf` 토픽이 모두 발행되는지 확인 |
