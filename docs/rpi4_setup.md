# 라즈베리파이 4 셋업 가이드

메카넘 AMR을 처음부터 구축하는 순서. 위에서 아래로 그대로 따라 하면 된다.

**접속 정보** — `ssh piapp@dhl-amr.local` (비번 = 이미지 구울 때 설정한 값), `ROS_DOMAIN_ID=42`

---

## 1. OS 이미지 굽기

Raspberry Pi Imager에서:

- **장치**: Raspberry Pi 4
- **OS**: Ubuntu Server **22.04 LTS (64-bit)** ← 32-bit 쓰면 안 됨
- **설정(톱니바퀴)**: hostname `dhl-amr` / 사용자 `piapp` + 비번 / SSH 켜기 / Wi-Fi 입력
- **쓰기** → SD카드 꽂고 부팅

## 2. 접속 & 업데이트

```bash
ssh piapp@dhl-amr.local
sudo apt update && sudo apt upgrade -y
```

## 3. ROS2 Humble 설치

```bash
sudo apt install -y curl
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key \
  -o /usr/share/keyrings/ros-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] \
  http://packages.ros.org/ros2/ubuntu jammy main" | sudo tee /etc/apt/sources.list.d/ros2.list
sudo apt update
sudo apt install -y ros-humble-desktop ros-dev-tools     # 오래 걸림
```

## 4. 환경변수 등록

```bash
cat >> ~/.bashrc <<'EOF'
source /opt/ros/humble/setup.bash
[ -f ~/ros2_ws/install/setup.bash ] && source ~/ros2_ws/install/setup.bash
export ROS_DOMAIN_ID=42
export MACHINE_TYPE=JetAuto
export HOST=jetauto
export MASTER=jetauto
export need_compile=True
export LIDAR_TYPE=G4
EOF
source ~/.bashrc
ros2 pkg list | wc -l     # 200 넘으면 정상
```

## 5. 소스 & 라이다 드라이버 빌드

소스 받기: https://drive.google.com/drive/folders/1hjoC0W_N4U9O1SrOTyMKOyGLLDtpt1YU

```bash
# Mac 에서 Pi 로 전송
scp -r ~/dev/work/paymentinapp/DHL/AMR/src/* piapp@dhl-amr.local:~/ros2_ws/src/
scp ~/dev/work/paymentinapp/{amr_build_setup.sh,lidar_g4_driver_build.sh} piapp@dhl-amr.local:~/

# Pi 에서 빌드
bash ~/amr_build_setup.sh           # SLAM/라이다 launch 등
bash ~/lidar_g4_driver_build.sh     # YDLIDAR G4 드라이버
source ~/.bashrc
```

## 6. 라이다 연결

```bash
sudo apt install -y ros-humble-laser-filters

# /dev/lidar 고정
sudo tee /etc/udev/rules.d/99-lidar.rules >/dev/null <<'EOF'
SUBSYSTEM=="tty", ATTRS{idVendor}=="10c4", ATTRS{idProduct}=="ea60", MODE="0666", GROUP="dialout", SYMLINK+="lidar"
EOF
sudo udevadm control --reload-rules && sudo udevadm trigger
ls -l /dev/lidar           # ttyUSB0 가리키면 OK
```

연결: `USB_DATA` → Pi USB, `USB_PWR`(micro-USB) → 5V. 확인:

```bash
ros2 launch peripherals lidar.launch.py
# 새 터미널
ros2 topic echo /scan --once --qos-reliability best_effort   # 숫자 배열 나오면 OK
```

## 7. 모터 모듈(I2C) 연결 & 테스트

**배선**: SDA→Pi 3번핀(GPIO2), SCL→Pi 5번핀(GPIO3), GND 공통. 전원: 배터리 12V → 모듈 VIN.

```bash
# I2C 켜기
echo "dtparam=i2c_arm=on" | sudo tee -a /boot/firmware/config.txt
sudo reboot
# 재접속 후
i2cdetect -y 1             # 0x34 보이면 OK
```

스크립트 전송 + 가상환경:

```bash
# Mac
ssh piapp@dhl-amr.local 'mkdir -p ~/amr_lidar_viz'
scp ~/dev/work/paymentinapp/{amr_env_setup.sh,amr_motor_odom_node.py,wasd_teleop.py,run_mapping.sh,motor_i2c_test.py,module_health.py,motor_init_test.py,motor_load_test.py} \
    piapp@dhl-amr.local:~/amr_lidar_viz/

# Pi
bash ~/amr_lidar_viz/amr_env_setup.sh      # ~/amr_env 생성 + smbus2
```

모터 테스트 (**바퀴 공중에 띄우고**):

```bash
source ~/amr_env/activate_amr.sh
cd ~/amr_lidar_viz
python3 module_health.py       # I2C/전원 안정성
python3 motor_init_test.py     # 4모터 회전 확인
```

## 8. Foxglove (지도 보기)

```bash
sudo apt install -y ros-humble-foxglove-bridge
```

Mac에 Foxglove Studio 설치(foxglove.dev/download) → Open connection → **Foxglove WebSocket** → `ws://dhl-amr.local:8765`

## 9. 매핑 실행

```bash
bash ~/amr_lidar_viz/run_mapping.sh
```

운전: `w/s` 전후 · `a/d` 평행 · `q/e` 회전 · `space` 정지. **Ctrl-C** 하면 지도가 `~/maps/`에 저장된다.

---

### 막힐 때

- OS는 **64-bit** 여야 함 (32-bit는 ROS2 설치 안 됨)
- venv는 `amr_env_setup.sh`가 만드는 것만 사용 (직접 만들면 rclpy 안 보임)
- Foxglove 연결은 꼭 **"Foxglove WebSocket"** 선택
- `i2cdetect`에 `0x34` 안 보이면 → 배선(SDA/SCL/GND)·전원 확인
