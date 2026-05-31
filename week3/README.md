# Week 3 — SLAM, Navigation & 최종 프로젝트

> **목표:** SLAM으로 환경 지도를 만들고, Navigation2로 자율주행을 구현하며, 최종 프로젝트를 완성한다.

---

## 📅 주간 일정

| 일차 | 주제 | 핵심 내용 |
|------|------|----------|
| Day 15 | SLAM 이론 & Cartographer | SLAM 개념, GraphSLAM, Cartographer 아키텍처 |
| Day 16 | 실제 SLAM 매핑 실습 | slam_toolbox로 실제 환경 매핑, 지도 저장 |
| Day 17 | Navigation2 개념 & 설정 | Nav2 아키텍처, Costmap, Planner, AMCL |
| Day 18 | Nav2 자율주행 실습 | 실제 자율주행, 파라미터 튜닝 |
| Day 19 | 최종 프로젝트 기획 | 자율 순찰 로봇 설계, Waypoint 시스템 |
| Day 20 | 최종 프로젝트 구현 | 핵심 기능 개발, 통합 테스트 |
| Day 21 | 최종 프로젝트 완성 | 최종 통합, 데모, 문서화 |

---

## ✅ Week 3 완료 조건

- [ ] SLAM 개념 이해 (EKF, GraphSLAM, Particle Filter)
- [ ] `slam_toolbox`로 실제 환경 지도 생성 완료
- [ ] 생성된 지도를 파일로 저장 (pgm + yaml)
- [ ] Navigation2 자율주행 성공 (2D Pose Estimate → 2D Nav Goal)
- [ ] Nav2 파라미터 기본 튜닝 완료
- [ ] 최종 프로젝트 (자율 순찰 로봇) 완성 및 데모

---

## ⚠️ 주의사항

- **SLAM 매핑 시** 로봇을 천천히 움직여야 합니다. 급회전/급가속은 매핀 정확도를 떨어뜨립니다
- **Loop closure**를 위해 같은 공간을 재방문하면 지도 정확도가 크게 향상됩니다
- **Nav2 자율주행**은 매핑이 완료된 환경에서만 실행하세요
- 매핑 중 배터리 방전에 주의 — 중간에 배터리가 나가면 지도가 손실됩니다
- Nav2 파라미터 튜닝은 실제 환경에 맞춰 점진적으로 진행하세요
