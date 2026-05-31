# Week 1 — 환경 구축 & ROS2 기초

> **목표:** SD 카드부터 ROS2 통신까지 전부 세팅하고 ROS2 개념에 익숙해지기

---

## 📅 주간 일정

| 일차 | 주제 | 핵심 내용 |
|------|------|----------|
| Day 1 | RPi OS 설치 | Ubuntu Server 22.04 LTS 플래싱, SSH/WiFi 설정 |
| Day 2 | ROS2 Humble 설치 (RPi) | ROS2 Base 설치, locale 설정, apt repository |
| Day 3 | Remote PC 세팅 | Ubuntu 22.04 + ROS2 Desktop + Gazebo 설치 |
| Day 4 | ROS2 핵심 개념 | Nodes, Topics, Services, Actions, Parameters |
| Day 5 | 첫 ROS2 패키지 | Publisher/Subscriber 노드 작성 (Python & C++) |
| Day 6 | 복습 & 트러블슈팅 | DDS 통신 문제, RMW 설정, 일반적인 오류 대처 |
| Day 7 | 미니 프로젝트 | CPU 온도 모니터링 노드 만들기 |

---

## ✅ Week 1 완료 조건

- [ ] Raspberry Pi에 Ubuntu Server 22.04 + ROS2 Humble base 설치 완료
- [ ] Remote PC에 Ubuntu 22.04 + ROS2 Humble desktop 설치 완료
- [ ] RPi ↔ PC 간 ROS_DOMAIN_ID 통일 및 통신 확인
- [ ] ROS2 기본 명령어 사용 가능 (ros2 run, topic, service, action, param)
- [ ] 직접 만든 Publisher/Subscriber 노드 실행 성공
- [ ] 미니 프로젝트 (온도 모니터링) 완성

---

## ⚠️ 사전 주의사항

- 모든 `sudo apt update` / `sudo apt upgrade`는 최신 상태 유지를 위해 매일 처음에 실행합니다
- RPi에서 빌드 시 발열 관리에 주의하세요 (방열판 + 팬 필수)
- 배터리가 아닌 유선 전원으로 RPi에 전원 공급하며 작업합니다
- Remote PC와 RPi는 **반드시 동일한 WiFi 네트워크**에 연결되어야 합니다
