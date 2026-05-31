# Day 10 — TurtleBot3 Bringup & Teleoperation

> **목표:** 실제 TurtleBot3 Burger를 구동하고(bringup), LIDAR 데이터를 확인하며 키보드로 로봇을 직접 움직인다.

---

## 10.1 준비 사항

### 하드웨어 체크리스트

```
□ TurtleBot3 Burger 완전 조립 상태
□ Raspberry Pi 4B 장착 및 케이블 연결
□ OpenCR USB 케이블 — RPi에 연결
□ LDS USB 케이블 — RPi에 연결 (또는 OpenCR 경유)
□ 배터리 충전 완료 (녹색 LED)
□ MicroSD 카드 삽입
□ 방열판 부착 확인
```

### 소프트웨어 체크리스트

```bash
# RPi에서 확인
ssh ubuntu@turtlebot-pi.local

# ROS2 환경 확인
echo $ROS_DOMAIN_ID     # 30
echo $TURTLEBOT3_MODEL  # burger
echo $LDS_MODEL         # LDS-01 or LDS-02

# Day 8 패키지 설치 완료 확인
ros2 pkg list | grep turtlebot3
```

---

## 10.2 Bringup 실행

> **Bringup** = 로봇의 모든 하드웨어 드라이버를 시작하는 과정

### Step 1: OpenCR 및 LDS 연결 확인 (RPi)

```bash
# USB 장치 목록
lsusb

# OpenCR (STMicroelectronics STM32)
# LDS (Silicon Labs CP210x or similar) 가 보여야 함

# 포트 확인
ls -la /dev/ttyACM*  # OpenCR (개수에 따라 ACM0 또는 ACM1)
ls -la /dev/ttyUSB*  # LDS (USB-to-UART)
```

> ⚠️ **LDS 연결 확인:** LDS에서 "삑" 소리와 함께 레이저가 회전하면 정상입니다.

### Step 2: Bringup Launch (RPi)

```bash
# RPi 터미널
ros2 launch turtlebot3_bringup robot.launch.py
```

**정상 부팅 시 출력 예시:**
```
[INFO] [1700000000.123456789] [turtlebot3_core]: OpenCR connected on /dev/ttyACM0
[INFO] [1700000000.234567890] [turtlebot3_core]: Firmware version: 2.0.0
[INFO] [1700000001.123456789] [turtlebot3_core]: Battery voltage: 11.4V
[INFO] [1700000001.234567890] [turtlebot3_lidar]: LDS connected
[INFO] [1700000001.345678901] [turtlebot3_lidar]: Start scanning...
```

### Step 3: 토픽 확인 (RPi 또는 PC)

```bash
# PC 터미널 (bringup이 잘 되고 있는지 확인)
ros2 topic list
```

**정상 출력:**
```
/battery_state
/cmd_vel
/diagnostics
/imu
/joint_states
/odom
/scan
/tf
/tf_static
```

---

## 10.3 LIDAR 데이터 확인

```bash
# LIDAR 데이터 구조 확인
ros2 interface show sensor_msgs/LaserScan

# 실제 데이터 보기
ros2 topic echo /scan

# 발행 주기 확인
ros2 topic hz /scan
# 약 5-10 Hz (LDS 모델에 따라 다름)
```

### LIDAR 데이터 이해

```
sensor_msgs/LaserScan 메시지:
  - angle_min: -3.14 (시작 각도, rad)
  - angle_max: 3.14 (끝 각도, rad)
  - angle_increment: 0.0175 (1° 단위)
  - range_min: 0.12 (최소 거리, m)
  - range_max: 3.5 (최대 거리, m)
  - ranges: [...]  (360개 거리 값)
  - intensities: [...] (반사 강도)
```

---

## 10.4 Teleoperation (키보드 원격 제어)

### PC에서 실행

```bash
# PC 터미널
export TURTLEBOT3_MODEL=burger
ros2 run turtlebot3_teleop teleop_keyboard
```

```
Control Your TurtleBot3!
---------------------------
Moving around:
        w
   a    s    d
        x

w/x : increase/decrease linear velocity (전진/후진)
a/d : increase/decrease angular velocity (좌/우회전)
space : force stop (강제 정지)

CTRL+C to quit
```

### 속도 증가 모드

```bash
# shift + 방향키 → 속도 2배
# 더블 클릭 → 가속
```

---

## 10.5 RViz2로 실시간 시각화

### PC에서 RViz2 실행

```bash
# PC에서 (bringup이 RPi에서 실행 중인 상태)
rviz2
```

### RViz2 설정

1. **Fixed Frame** → `odom` 또는 `base_footprint` 설정
2. **Add (Ctrl+A)** → `RobotModel` 추가
3. **Add** → `LaserScan` 추가 → Topic: `/scan`
4. **Add** → `Odometry` 추가 → Topic: `/odom`
5. **Add** → `TF` 추가

> 💡 **프로 팁:** 설정을 저장하려면 `File → Save Config As...` → `~/tb3.rviz`

---

## 10.6 토픽 모니터링 팁

```bash
# PC에서 배터리 상태 확인
ros2 topic echo /battery_state
# voltage: 11.4V
# percentage: 0.85  (없을 수 있음)

# IMU 데이터 확인
ros2 topic echo /imu

# Odometry 확인
ros2 topic echo /odom --once
```

---

## 10.7 Teleop 없이 직접 cmd_vel 발행

```bash
# 직진 (0.1 m/s)
ros2 topic pub /cmd_vel geometry_msgs/Twist \
  "{linear: {x: 0.1, y: 0.0, z: 0.0}, angular: {x: 0.0, y: 0.0, z: 0.0}}" \
  -r 10

# 3초 후 정지
ros2 topic pub /cmd_vel geometry_msgs/Twist \
  "{linear: {x: 0.0, y: 0.0, z: 0.0}, angular: {x: 0.0, y: 0.0, z: 0.0}}" \
  -r 10

# 제자리 회전 (0.5 rad/s)
ros2 topic pub /cmd_vel geometry_msgs/Twist \
  "{linear: {x: 0.0, y: 0.0, z: 0.0}, angular: {x: 0.0, y: 0.0, z: 0.5}}" \
  -r 10
```

---

## 10.8 Odometry 기반 이동 거리 측정

```bash
# 2초간 직진 후 이동 거리 확인
ros2 topic pub /cmd_vel geometry_msgs/Twist \
  "{linear: {x: 0.1, y: 0.0, z: 0.0}, angular: {x: 0.0, y: 0.0, z: 0.0}}" \
  -r 10 &
PID=$!
sleep 2
kill $PID

# 이동 거리 확인
ros2 topic echo /odom --once
# pose.pose.position.x 값 확인 (약 0.2m)
```

---

## 📝 연습 문제

1. **벽 감지:** `/scan` 데이터에서 전방(0°)의 거리 값을 확인하고, 벽까지의 거리를 측정하세요
2. **주행 테스트:** 로봇을 1m 직진시키고, 실제 이동 거리와 odometry 값을 비교하여 오차를 기록하세요
3. **회전 테스트:** 로봇을 360° 회전시키고, 실제 회전 각도와 odometry의 차이를 측정하세요
4. **배터리 소모:** 텔레옵으로 5분간 연속 주행 후 배터리 전압 변화를 기록하세요
5. **장애물 매핑:** 로봇을 천천히 회전시키며 `/scan` 데이터를 연속 수집하고, 주변 환경을 그려보세요

---

## ⚠️ 트러블슈팅

| 문제 | 해결 방법 |
|------|----------|
| OpenCR 연결 실패 | `ls /dev/ttyACM*` 확인, `sudo chmod 666 /dev/ttyACM0` 후 재시도 |
| LDS 회전 안 함 | LDS 케이블 연결 확인, 전원 공급 확인 |
| teleop_keyboard 반응 없음 | 터미널이 포커스 상태인지 확인, `ros2 topic echo /cmd_vel`로 명령 발행 확인 |
| RViz2에 LaserScan 안 보임 | `Global Options > Fixed Frame`을 `base_scan`으로 변경 |
| 배터리 부족 경고 | 10.5V 이하이면 충전 필요 (로봇 동작 불안정) |
| `robot.launch.py` 실행 후 바로 종료 | OpenCR 포트 권한 문제, `dmesg | grep tty`로 로그 확인 |
