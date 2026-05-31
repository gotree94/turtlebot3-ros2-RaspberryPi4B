# Day 1 — Raspberry Pi OS 설치 & 초기 설정

> **목표:** Raspberry Pi 4B 8GB에 Ubuntu Server 22.04 LTS를 설치하고 SSH로 원격 접속까지 확인한다.

---

## 1.1 개요

TurtleBot3 Burger의 두뇌 역할을 할 Raspberry Pi 4B에 운영체제를 설치합니다. <br>
ROS2 Humble과의 호환성을 위해 **Ubuntu Server 22.04 LTS (Jammy Jellyfish)** 를 사용합니다. <br>

> **왜 Desktop이 아닌 Server 버전인가?**  
> TurtleBot3의 SBC는 리소스가 제한적입니다. GUI가 필요 없고,  <br> 모든 ROS2 명령어는 Remote PC에서 실행하므로 Server 버전으로 충분하며 더 가볍게 동작합니다.

---

## 1.2 준비물

- Raspberry Pi 4B 8GB
- MicroSD 카드 (32GB 이상, 64GB 권장)
- MicroSD 리더기
- USB-C 전원 어댑터 (5V 3A)
- (선택) HDMI 케이블 + 모니터 (초기 설정용)
- (선택) USB 키보드 (초기 설정용)

---

## 1.3 Raspberry Pi Imager 설치

**Windows:**

```bash
# https://www.raspberrypi.com/software/ 에서 다운로드 후 설치
```

**Ubuntu/Debian:**

```bash
sudo apt update
sudo apt install rpi-imager
```

**macOS:**

```bash
brew install --cask raspberry-pi-imager
```

---

## 1.4 Ubuntu Server 22.04 LTS 플래싱

### Step 1: Imager 실행

1. Raspberry Pi Imager 실행
2. **Choose Device** → Raspberry Pi 4
3. **Choose OS** → Other general-purpose OS → Ubuntu → **Ubuntu Server 22.04 LTS (64-bit)** (arm64)
4. **Choose Storage** → MicroSD 카드 선택

### Step 2: 고급 설정 (Ctrl+Shift+X 또는 톱니바퀴 아이콘)

⚙️ **설정 필수 항목:**

```
☑ Set hostname: turtlebot-pi      # 네트워크에서 식별할 이름
☑ Enable SSH
   ☑ Use password authentication
   Username: ubuntu
   Password: [강력한 비밀번호 설정]
☑ Configure wireless LAN
   SSID: [WiFi 이름]
   Password: [WiFi 비밀번호]
   Wireless LAN country: KR (또는 해당 국가)
☑ Set locale settings
   Timezone: Asia/Seoul
```

### Step 3: 쓰기 시작

**Save** 버튼 클릭 → **Yes** → 기존 데이터 삭제 경고 확인 → 쓰기 완료까지 대기 (약 5-10분)

---

## 1.5 초기 부팅 & SSH 접속

### Step 1: MicroSD 삽입 및 부팅

1. MicroSD를 RPi에 삽입
2. USB-C 전원 연결
3. 초록색 LED 점등 확인 (약 1-2분 후 안정화)

### Step 2: SSH 접속

같은 네트워크에 연결된 PC에서:

```bash
# hostname으로 접속
ssh ubuntu@turtlebot-pi.local

# 또는 IP 주소로 접속 (공유기 관리자 페이지에서 확인)
ssh ubuntu@192.168.x.x
```

> **처음 접속 시:** "Are you sure you want to continue connecting?" → `yes` 입력

### Step 3: 기본 시스템 업데이트

접속 후 즉시 실행:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt autoremove -y
```

### Step 4: 추가 필수 패키지 설치

```bash
# 빌드 도구
sudo apt install -y build-essential cmake git

# 네트워크 도구
sudo apt install -y net-tools curl wget

# 텍스트 에디터
sudo apt install -y vim nano

# 기타 유틸리티
sudo apt install -y htop tree
```

---

## 1.6 시스템 최적화

### 불필요한 서비스 중지 (RPi 리소스 확보)

```bash
# 자동 업데이트 중지 (임베디드 시스템에서는 수동 업데이트 권장)
sudo systemctl disable --now apt-daily.timer
sudo systemctl disable --now apt-daily-upgrade.timer

# NetworkManager wait 비활성화 (부팅 시간 단축)
sudo systemctl mask systemd-networkd-wait-online.service

# 절전 모드 비활성화
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

### WiFi 안정화 (5GHz 우선 사용)

```bash
# 5GHz WiFi 사용을 위한 설정
sudo nano /etc/NetworkManager/conf.d/5ghz.conf
```

다음 내용 입력:

```ini
[connection]
wifi.powersave=2
```

저장 후 재부팅:

```bash
sudo reboot
```

---

## 1.7 IP 주소 고정 (선택 사항)

매번 같은 IP로 접속하려면 DHCP 예약 또는 고정 IP 설정:

```bash
# 현재 IP 확인
ip addr show

# netplan 설정 파일 확인
ls /etc/netplan/
sudo nano /etc/netplan/00-installer-config.yaml
```

```
sudo nano /etc/netplan/50-cloud-init.yaml
```

```yaml
network:
  ethernets:
    eth0:
      dhcp4: true
  version: 2
  wifis:
    wlan0:
      dhcp4: no
      addresses:
        - 192.168.x.x/24   # 원하는 고정 IP
      gateway4: 192.168.x.1
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
      access-points:
        "YourWiFiSSID":
          password: "YourWiFiPassword"
```

```

# This file is generated from information provided by the datasource.  Changes
# to it will not persist across an instance reboot.  To disable cloud-init's
# network configuration capabilities, write a file
# /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg with the following:
# network: {config: disabled}
network:
    ethernets:
        eth0:
            dhcp4: true
            dhcp6: true
            optional: true
    version: 2
    wifis:
        wlan0:
            access-points:
                SK_3530_5G:
                    hidden: true
                    password: de1df9c5bdae585b1e2506bb50ebe8242c4c57301caa442892698595c5ae0c14
            dhcp4: true
            optional: true
            regulatory-domain: KR
```

```
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: true
  wifis:
    wlan0:
      dhcp4: false
      addresses:
        - 192.168.75.150/24  # <- 원하는 고정 IP로 변경하세요 (예: 150)
      routes:
        - to: default
          via: 192.168.75.1  # 게이트웨이 IP
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
      access-points:
        "YourWiFiSSID":      # <- 실제 WiFi 이름(SSID)으로 변경
          password: "YourWiFiPassword"  # <- 실제 WiFi 비밀번호로 변경
```

적용:

```bash
sudo netplan apply
```

---

## 1.8 확인 사항

```bash
# 호스트네임 확인
hostname

# IP 주소 확인
hostname -I

# 시스템 정보 확인
uname -a
cat /etc/os-release

# 메모리 확인
free -h

# 디스크 확인
df -h

# 네트워크 확인 (Remote PC와 ping 교환)
ping 192.168.x.x   # Remote PC IP 주소
```

---

## 📝 연습 문제

1. RPi의 CPU 온도를 확인하는 명령어를 찾아 실행해보세요 (`/sys/class/thermal/` 참고)
2. `htop` 명령어로 현재 프로세스 목록을 확인하고, CPU와 메모리 사용률을 기록하세요
3. Remote PC에서 RPi로 SSH 키 기반 로그인을 설정해보세요 (`ssh-keygen` + `ssh-copy-id`)
4. RPi의 WiFi 신호 강도를 확인하는 명령어를 찾아 실행해보세요

---

## ⚠️ 트러블슈팅

| 문제 | 해결 방법 |
|------|----------|
| SSH 접속 안 됨 | 같은 WiFi인지 확인, `ping turtlebot-pi.local`로 네트워크 확인 |
| 부팅 후 깜빡임만 있음 | MicroSD가 제대로 삽입되었는지 확인, 다른 이미지로 재시도 |
| apt update 실패 | `sudo timedatectl set-ntp true`로 시간 동기화 후 재시도 |
| WiFi 연결 불안정 | `sudo nano /etc/network/interfaces` 불필요한 설정 제거 |
