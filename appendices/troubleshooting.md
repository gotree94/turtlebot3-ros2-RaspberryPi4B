# 부록 A: 문제 해결 가이드 (Troubleshooting)

> **ROS2 Humble + TurtleBot3 Burger + Raspberry Pi 4B 환경에서 자주 발생하는 문제와 해결 방법**

---

## 1. 설치 관련 문제

### Q1: ROS2 Humble 설치 중 GPG 키 오류

**증상:** `sudo apt update` 실행 시 GPG 키 관련 오류 발생

**해결:**
```bash
# 키가 만료되었거나 잘못된 경우 재설치
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key \
  -o /usr/share/keyrings/ros-archive-keyring.gpg
sudo apt update
```

### Q2: colcon 명령어를 찾을 수 없음

**해결:**
```bash
sudo apt install -y python3-colcon-common-extensions
```

### Q3: `ros2: command not found`

**해결:**
```bash
# ROS2 환경이 source되지 않음
source /opt/ros/humble/setup.bash

# 영구 적용
echo 'source /opt/ros/humble/setup.bash' >> ~/.bashrc
source ~/.bashrc
```

### Q4: RPi에서 colcon build 중 메모리 부족

**증상:** 빌드 도중 `Killed` 메시지와 함께 종료

**해결:**
```bash
# 단일 코어로 빌드
colcon build --symlink-install --parallel-workers 1

# 또는 SWAP 메모리 추가 (8GB 모델에서는 불필요하지만 안전장치)
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

---

## 2. 통신 문제

### Q5: RPi ↔ PC 간 토픽이 보이지 않음

**증상:** `ros2 topic list`에 상대방의 토픽이 표시되지 않음

**단계별 진단:**

```bash
# 1단계: 기본 네트워크 확인
ping turtlebot-pi.local
ping [RPi IP]

# 2단계: ROS_DOMAIN_ID 확인 (양쪽 값이 같아야 함)
echo $ROS_DOMAIN_ID   # 30이어야 함

# 3단계: RPi에서 DDS 테스트
ros2 run demo_nodes_py talker &
ros2 topic list       # /chatter가 보여야 함

# 4단계: PC에서 확인
ros2 topic list       # /chatter가 보여야 함
ros2 topic echo /chatter

# 5단계: RMW 교체 시도 (양쪽 동일하게)
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
```

### Q6: DDS 통신이 간헐적으로 끊김

**해결:**
```bash
# WiFi 전원 관리 비활성화 (RPi)
sudo iwconfig wlan0 power off

# 또는 NetworkManager 설정에서 WiFi power save 비활성화
sudo nano /etc/NetworkManager/conf.d/wifi-powersave.conf
# 내용: [connection] wifi.powersave=2
sudo systemctl restart NetworkManager

# 5GHz WiFi 우선 사용
```

### Q7: UDP 멀티캐스트 차단 (방화벽)

**해결:**
```bash
# 방화벽 임시 비활성화 후 테스트
sudo ufw disable

# 또는 DDS 포트 허용
sudo ufw allow 7400:7500/udp
sudo ufw allow 11311/udp
```

---

## 3. TurtleBot3 하드웨어 문제

### Q8: OpenCR이 인식되지 않음

**증상:** `/dev/ttyACM*` 없음, bringup 실패

**해결:**
```bash
# USB 연결 상태 확인
lsusb | grep STM

# udev 규칙 확인
sudo cp $(ros2 pkg prefix turtlebot3_bringup)/share/turtlebot3_bringup/script/99-turtlebot3-cdc.rules /etc/udev/rulesd/
sudo udevadm control --reload-rules
sudo udevadm trigger

# 권한 문제
sudo usermod -aG dialout $USER
# 로그아웃 후 재로그인 필요

# OpenCR 리셋 버튼 누른 후 재연결
```

### Q9: LIDAR (LDS)가 회전하지 않음

**증상:** LDS에서 소리가 안 나고, `/scan` 토픽 없음

**해결:**
```bash
# LDS 모델 확인
echo $LDS_MODEL

# LDS-02인데 LDS-01로 설정된 경우
export LDS_MODEL=LDS-02

# USB 연결 확인
lsusb | grep -i "silicon\|cp210"

# 전원 확인 (LDS는 OpenCR을 통해 전원 공급)
# 배터리가 부족하면 LDS가 작동하지 않을 수 있음
```

### Q10: OpenCR 펌웨어 업데이트 실패

**해결:**
```bash
# Bootloader 모드로 진입
# OpenCR 보드의 BOOT0 버튼을 누른 상태에서 USB 연결
# 그린 LED가 켜지면 bootloader 모드

export OPENCR_PORT=/dev/ttyACM0
export OPENCR_MODEL=burger

cd ~
wget https://github.com/ROBOTIS-GIT/OpenCR-Binaries/raw/master/turtlebot3/ROS2/latest/opencr_firmware.tar.gz
tar xvf opencr_firmware.tar.gz
cd opencr_firmware
./update.sh $OPENCR_PORT $OPENCR_MODEL
```

---

## 4. Gazebo 문제

### Q11: Gazebo 실행 시 검은 화면

**해결:**
```bash
# 소프트웨어 렌더링 사용
export LIBGL_ALWAYS_SOFTWARE=1

# 또는 SwiftShader 설치
sudo apt install -y libgl1-mesa-glx

# 기존 프로세스 정리
killall gzserver gzclient
```

### Q12: Gazebo 모델 다운로드 속도 느림

**해결:**
```bash
# 자동 모델 다운로드 비활성화
export GAZEBO_MODEL_DATABASE_URI=""

# .bashrc에 영구 적용
echo 'export GAZEBO_MODEL_DATABASE_URI=""' >> ~/.bashrc
```

---

## 5. SLAM & Navigation 문제

### Q13: SLAM 지도가 흐릿함

**해결:**
```bash
# 이동 속도 줄이기 (0.1 m/s 이하)
# Loop Closure를 위해 같은 경로 재방문
# minimum_travel_distance 파라미터 조정
minimum_travel_distance: 0.05
minimum_travel_heading: 0.05
```

### Q14: Nav2에서 "No map received" 오류

**해결:**
```bash
# 맵 파일 경로 확인
ls -la ~/my_map.yaml

# 절대 경로 사용
ros2 launch turtlebot3_navigation2 navigation2.launch.py \
  map:=/home/ubuntu/my_map.yaml
```

### Q15: AMCL 위치가 수렴하지 않음

**해결:**
```bash
# RViz2에서 "2D Pose Estimate"로 더 정확한 초기 위치 설정
# AMCL 파라미터 튜닝
max_particles: 3000
min_particles: 50

# 초기 위치를 yaml에 직접 설정
initial_pose_x: 0.0
initial_pose_y: 0.0
initial_pose_yaw: 0.0
```

### Q16: 로봇이 목적지로 직진하지 않고 빙글빙글 돎

**해결:**
```yaml
# DWB 파라미터 튜닝 (burger.yaml)
max_vel_theta: 1.0          # 회전 속도 감소
min_vel_theta: -1.0
acc_lim_theta: 2.0          # 각가속도 감소
inflation_radius: 0.3       # 안전 영역 축소
```

---

## 6. RPi 시스템 문제

### Q17: RPi 과열

**증상:** 성능 저하, 갑작스러운 종료

**해결:**
```bash
# 방열판 + 팬 필수
# CPU 온도 모니터링
watch -n 2 'cat /sys/class/thermal/thermal_zone0/temp'

# 쓰로틀링 확인
vcgencmd get_throttled

# 부하가 높은 작업은 PC에서 실행 (SLAM, Nav2)
```

### Q18: SD 카드 용량 부족

**해결:**
```bash
# 불필요한 패키지 제거
sudo apt autoremove -y
sudo apt clean

# ROS2 로그 제한
echo 'export RCUTILS_CONSOLE_OUTPUT_FORMAT="[{severity}] [{time}] [{name}]: {message}"' >> ~/.bashrc

# 오래된 로그 삭제
rm -rf ~/.ros/log/*
```

---

## 7. SSH 및 원격 접속 문제

### Q19: SSH 접속이 느림

**해결:**
```bash
# SSH 설정 최적화
sudo nano /etc/ssh/sshd_config

# 추가 또는 수정:
UseDNS no
GSSAPIAuthentication no

sudo systemctl restart sshd
```

### Q20: hostname.local 접속 안 됨

**해결:**
```bash
# mDNS/Avahi 설치 확인
sudo apt install -y avahi-daemon
sudo systemctl enable --now avahi-daemon

# IP로 직접 접속
ssh ubuntu@192.168.x.x
```

---

## 8. 로그 및 진단 명령어 모음

```bash
# === 시스템 진단 ===
# ROS2 환경 진단
ros2 doctor
ros2 doctor --report

# 시스템 로그
dmesg | grep -i "tty\|usb\|error"

# OpenCR 로그
journalctl -f | grep ttyACM

# === 네트워크 진단 ===
# 대역폭 테스트
iperf3 -c turtlebot-pi.local

# 포트 확인
netstat -uln | grep 7400

# === 성능 모니터링 ===
htop              # 프로세스별 CPU/RAM
iotop             # 디스크 I/O
nethogs wlan0     # 네트워크 트래픽
```
