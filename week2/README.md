# Week 2 — TurtleBot3 구동 & 센서/제어

> **목표:** 실제 TurtleBot3 Burger를 구동하고, LIDAR/Odometry 센서 데이터를 읽으며 키보드와 조이스틱으로 제어한다.

---

## 📅 주간 일정

| 일차 | 주제 | 핵심 내용 |
|------|------|----------|
| Day 8 | TurtleBot3 SBC 패키지 설치 (RPi) | turtlebot3 패키지 빌드, OpenCR 설정 |
| Day 9 | Remote PC TurtleBot3 패키지 | PC에 TurtleBot3 패키지 + 시뮬레이션 |
| Day 10 | Bringup & Teleoperation | 로봇 구동, LDS 확인, 키보드 제어 |
| Day 11 | 센서 데이터 & RViz2 시각화 | LIDAR, Odometry, TF 트리 이해 |
| Day 12 | PID 제어 & 로봇 구동 원리 | DYNAMIXEL 모터, cmd_vel, Go-to-Goal |
| Day 13 | RViz2 심화 & URDF | URDF 분석, Joint State, Marker |
| Day 14 | 미니 프로젝트 | LIDAR 기반 장애물 회피 주행 |

---

## ✅ Week 2 완료 조건

- [ ] RPi에 TurtleBot3 패키지 빌드 완료 (OpenCR udev 규칙 포함)
- [ ] Remote PC에 TurtleBot3 패키지 설치 및 환경 변수 설정
- [ ] `ros2 launch turtlebot3_bringup robot.launch.py` 성공
- [ ] 키보드로 TurtleBot3 전/후/좌/우 움직임 제어
- [ ] `/scan` (LIDAR) 데이터를 `rqt_plot` 또는 RViz2로 확인
- [ ] `/odom`, `/tf` 토픽 이해
- [ ] TurtleBot3 URDF 구조 이해
- [ ] 미니 프로젝트 (LIDAR 장애물 회피) 완성

---

## ⚠️ 주의사항

- **배터리 잔량을 항상 확인** — 작업 시작 전 완충 권장
- **LDS 레이저는 눈에 직접 조사하지 않도록 주의** (Class 1이지만 안전 수칙 준수)
- 바닥이 평평한 장소에서 테스트 (카펫/매트 위에서는 주행 정확도 떨어짐)
- 첫 구동 시 OpenCR 펌웨어 버전 확인 필수
- **LDS-01 vs LDS-02 모델을 반드시 확인**하고 환경 변수 설정 (`export LDS_MODEL=LDS-02`)
