# 🐢 TurtleBot3 Burger + ROS2 Humble — 3주 완성 커리큘럼

> Raspberry Pi 4B (8GB) 환경에서 ROS2 Humble을 구축하고, TurtleBot3 Burger를 제어하며 SLAM/네비게이션까지 마스터하는 3주 집중 코스

---

## 📋 개요

이 커리큘럼은 **TurtleBot3 Burger** 로봇에 **Raspberry Pi 4B 8GB**를 SBC로 탑재하고, **ROS2 Humble Hawksbill** 환경에서 최종 프로젝트까지 완성하는 것을 목표로 합니다.

| 항목 | 내용 |
|------|------|
| **총 기간** | 3주 (21일) |
| **난이도** | 초중급 (Linux 기초 지식 필요) |
| **ROS 버전** | ROS2 Humble Hawksbill |
| **타겟 보드** | Raspberry Pi 4B 8GB |
| **로봇** | TurtleBot3 Burger |
| **Remote PC OS** | Ubuntu 22.04 LTS Desktop |
| **RPi OS** | Ubuntu Server 22.04 LTS (arm64) |

---

## 🛠 사전 준비물

### 하드웨어

| 항목 | 비고 |
|------|------|
| TurtleBot3 Burger | OpenCR 1.0, DYNAMIXEL XL430-W250 x2, 배터리 포함 |
| Raspberry Pi 4B 8GB | SBC로 사용 |
| MicroSD 카드 | 32GB 이상 (64GB 권장, Class 10 U3) |
| MicroSD 리더기 | OS 플래싱용 |
| USB-C 전원 어댑터 (5V 3A) | RPi 전원용 |
| Remote PC (노트북/데스크탑) | Ubuntu 22.04 Desktop 설치 |
| WiFi 공유기 | 동일 네트워크 (5GHz 권장) |
| USB 카메라 (선택) | TurtleBot3 Burger 마운트 가능한 모델 |
| 방열판 + 케이스 팬 | RPi 4B 발열 필수 대책 |

### 소프트웨어

| 항목 | 버전 |
|------|------|
| Raspberry Pi Imager | 최신 버전 |
| Ubuntu Server | 22.04 LTS (Jammy Jellyfish) arm64 |
| Ubuntu Desktop | 22.04 LTS (Jammy Jellyfish) amd64 |
| ROS 2 | Humble Hawksbill |
| Gazebo | Classic 11 (시뮬레이션) |

---

## 📚 주차별 학습 로드맵

```
Week 1 ────┬── Day 1~2: OS + ROS2 설치
           ├── Day 3~4: Remote PC 세팅 + ROS2 개념
           ├── Day 5~6: 첫 패키지 작성 + 복습
           └── Day 7: [미니 프로젝트] 온도 모니터링 노드
                   
Week 2 ────┬── Day 8~9: TurtleBot3 SBC + PC 패키지
           ├── Day 10~11: Bringup + 센서 시각화
           ├── Day 12~13: PID 제어 + URDF/RViz2
           └── Day 14: [미니 프로젝트] LIDAR 장애물 회피

Week 3 ────┬── Day 15~16: SLAM 이론 + 실제 매핑
           ├── Day 17~18: Navigation2 + 자율주행
           └── Day 19~21: [최종 프로젝트] 자율 순찰 로봇
```

---

## 📂 디렉토리 구조

```
turtlebot3-ros2-curriculum/
├── README.md                          # 이 파일 (전체 개요)
│
├── week1/                             # 1주차: 환경 구축 & ROS2 기초
│   ├── README.md                      # 1주차 개요
│   ├── day01_os_installation.md       # RPi OS 설치
│   ├── day02_ros2_installation.md     # ROS2 Humble 설치 (RPi)
│   ├── day03_remote_pc_setup.md       # Remote PC 세팅
│   ├── day04_ros2_concepts.md         # ROS2 핵심 개념
│   ├── day05_first_package.md         # 첫 ROS2 패키지 작성
│   ├── day06_review_troubleshooting.md# 복습 & 트러블슈팅
│   └── day07_mini_project_temp.md     # 미니 프로젝트: 온도 모니터링
│
├── week2/                             # 2주차: TurtleBot3 구동 & 센서/제어
│   ├── README.md                      # 2주차 개요
│   ├── day08_turtlebot3_sbc_setup.md  # SBC 패키지 설치
│   ├── day09_remote_pc_tb3_pkg.md     # Remote PC 패키지 설치
│   ├── day10_bringup_teleop.md        # Bringup & Teleoperation
│   ├── day11_sensor_visualization.md  # 센서 데이터 & RViz2
│   ├── day12_pid_control_basics.md    # PID 제어 & 로봇 구동
│   ├── day13_rviz2_urdf.md           # RViz2 심화 & URDF
│   └── day14_mini_project_obstacle.md # 미니 프로젝트: 장애물 회피
│
├── week3/                             # 3주차: SLAM, Navigation & 최종 프로젝트
│   ├── README.md                      # 3주차 개요
│   ├── day15_slam_theory.md           # SLAM 이론 & Cartographer
│   ├── day16_slam_mapping.md          # 실제 SLAM 매핑 실습
│   ├── day17_navigation2_concepts.md  # Nav2 개념 & 설정
│   ├── day18_nav2_autonomous.md       # Nav2 자율주행 실습
│   ├── day19_final_project_plan.md    # 최종 프로젝트 기획
│   ├── day20_final_project_impl.md    # 최종 프로젝트 구현
│   └── day21_final_project_final.md   # 최종 프로젝트 완성
│
├── exercises/                         # 연습문제 모음
│   ├── week1_exercises.md             # 1주차 연습문제
│   ├── week2_exercises.md             # 2주차 연습문제
│   └── week3_exercises.md             # 3주차 연습문제
│
└── appendices/                        # 부록
    ├── troubleshooting.md             # 문제 해결 가이드
    ├── ros2_cheatsheet.md             # ROS2 명령어 치트시트
    └── references.md                  # 참고 자료 & 링크
```

---

## 🎯 학습 목표

이 커리큘럼을 완료하면 다음을 할 수 있습니다:

1. ✅ Raspberry Pi 4B에 Ubuntu Server + ROS2 Humble을 설치하고 설정한다
2. ✅ ROS2의 핵심 개념 (Node, Topic, Service, Action, Parameter)을 이해하고 코드로 구현한다
3. ✅ TurtleBot3 Burger를 bringup하고 키보드/조이스틱으로 제어한다
4. ✅ LIDAR, Odometry, IMU 센서 데이터를 읽고 RViz2로 시각화한다
5. ✅ SLAM (Cartographer/slam_toolbox)으로 실제 환경의 지도를 생성한다
6. ✅ Navigation2 (Nav2)를 이용해 자율주행을 구현한다
7. ✅ 최종 프로젝트로 자율 순찰 로봇 시스템을 완성한다

---

## 🚀 시작하기

**Step 1:** 이 리포지토리를 클론합니다

```bash
git clone https://github.com/your-username/turtlebot3-ros2-curriculum.git
cd turtlebot3-ros2-curriculum
```

**Step 2:** 주차별로 순서대로 진행합니다

```
week1/ → week2/ → week3/
```

각 Day 폴더의 `.md` 파일을 읽고 명령어를 따라 실행하면 됩니다.

**Step 3:** 연습문제로 복습합니다

각 주차별 연습문제는 `exercises/` 디렉토리에 있습니다.

---

## 📝 라이선스

이 교육 자료는 **MIT License**로 제공됩니다. 자유롭게 수정하고 배포하세요.

---

## 🙏 참고 자료

- [ROBOTIS TurtleBot3 e-Manual](https://emanual.robotis.com/docs/en/platform/turtlebot3/overview/)
- [ROS2 Humble Documentation](https://docs.ros.org/en/humble/)
- [Navigation2 Documentation](https://navigation.ros.org/)
- [SLAM Toolbox](https://github.com/SteveMacenski/slam_toolbox)
