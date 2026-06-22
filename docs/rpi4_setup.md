# AMR 셋업 가이드 (Raspberry Pi 4 + Encoder Motor Module)

지상 AMR(메카넘 4WD)을 RPi4에서 처음부터 구축하는 설치 런북.
**모터 제어 = Encoder Motor Module V1.3 (I2C 0x34)** 경로 기준.

> ⚠️ 과거 ROS Robot Controller V1.2(`/dev/rrc`, USB시리얼, IMU) 경로는 통신 포트
> 하드웨어 사망으로 폐기. `/dev/rrc`·`amr_udev_setup.sh`·`1a86:55d4` 관련 단계는 **불필요**.

## 접속 정보 (공통)

| 항목 | 값 |
|---|---|
| Hostname | `dhl-amr.local` |
| User | `piapp` |
| SSH | `ssh piapp@dhl-amr.local` |
| 비밀번호 | OS 이미지 구울 때 설정한 값 |
| ROS_DOMAIN_ID | `42` |

> 작업폴더(Mac): `/Users/hipark/dev/work/paymentinapp` — 아래 스크립트들이 있는 곳.

---

## 1. 이미지 설치 (Raspberry Pi Imager)

1. **장치 선택**: Raspberry Pi 4
2. **운영체제**: Other general-purpose OS → Ubuntu → **Ubuntu Server 22.04.x LTS (64-bit)**
   - ⚠️ **반드시 64-bit(arm64)**. 32-bit(armhf)는 `ros-humble-*` 패키지 미지원.
3. **저장소**: 대상 microSD 선택
4. **사용자 지정**(설정 톱니바퀴):
   - Hostname: `dhl-amr`
   - SSH 사용: 체크 (비밀번호 인증)
   - 사용자: `piapp` / 비밀번호 설정
   - Wi-Fi SSID/비밀번호 + 국가 설정
   - 로케일/시간대 설정
5. **쓰기** → 완료 후 SD 카드를 Pi에 삽입, 부팅.

---

## 2. 원격 접속 & 업데이트

```bash
ssh piapp@dhl-amr.local
# (재이미징 후 host key 에러 시 Mac에서: ssh-keygen -R dhl-amr.local)

sudo apt update && sudo apt upgrade -y
```

---

## 3. ROS2 Humble 설치

저장소 추가:
```bash
sudo apt install -y curl
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key \
  -o /usr/share/keyrings/ros-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] \
  http://packages.ros.org/ros2/ubuntu jammy main" \
  | sudo tee /etc/apt/sources.list.d/ros2.list
sudo apt update
```

설치(오래 걸림):
```bash
sudo apt install -y ros-humble-desktop ros-dev-tools
```

---

## 4. 환경변수 고정 (`~/.bashrc`)

```bash
grep -qxF "source /opt/ros/humble/setup.bash" ~/.bashrc || echo "source /opt/ros/humble/setup.bash" >> ~/.bashrc
grep -qxF "export ROS_DOMAIN_ID=42" ~/.bashrc || echo "export ROS_DOMAIN_ID=42" >> ~/.bashrc
source ~/.bashrc

# 검증
printenv ROS_DISTRO        # -> humble
printenv ROS_DOMAIN_ID     # -> 42
ros2 pkg list | wc -l      # 200+ 이면 desktop 정상
```

통신 테스트:
```bash
# 터미널 A
ros2 run demo_nodes_cpp talker
# 터미널 B (새 SSH)
ros2 run demo_nodes_cpp listener
```

---

## 5. Hiwonder 소스 빌드 (라이다 launch / SLAM 설정용)

> 모터 구동은 우리 자체 노드(`amr_motor_odom_node.py`)가 하지만,
> **라이다 launch(`peripherals`)·메카넘 운동학·slam 설정**은 Hiwonder 소스를 빌드해서 사용한다.
> (`ros_robot_controller`·`controller` 패키지는 빌드는 되지만 **런타임 미사용**.)

소스 다운로드: https://drive.google.com/drive/folders/1hjoC0W_N4U9O1SrOTyMKOyGLLDtpt1YU

전송 + 빌드:
```bash
# Pi
mkdir -p ~/ros2_ws/src

# Mac (로컬 터미널)
scp -r /Users/hipark/dev/work/paymentinapp/DHL/AMR/src/* piapp@dhl-amr.local:~/ros2_ws/src/
scp /Users/hipark/dev/work/paymentinapp/amr_build_setup.sh piapp@dhl-amr.local:~/

# Pi
bash ~/amr_build_setup.sh
source ~/.bashrc
ros2 pkg list | grep -E 'slam|navigation|peripherals'
```

> `amr_build_setup.sh`가 .bashrc에 프로젝트 env(MACHINE_TYPE=JetAuto, HOST/MASTER=jetauto,
> need_compile=True, LIDAR_TYPE=G4 등) 등록 + slam_toolbox/nav2 등 apt 의존성 설치 + colcon 빌드
> (xf_mic_asr_offline·yolov5 제외).

---

## 6. 라이다(YDLIDAR G4) 드라이버 + 연결

드라이버 빌드:
```bash
# Mac
scp /Users/hipark/dev/work/paymentinapp/lidar_g4_driver_build.sh piapp@dhl-amr.local:~/
# Pi
bash ~/lidar_g4_driver_build.sh        # YDLidar-SDK + ydlidar_ros2_driver(★humble 브랜치)
source ~/.bashrc && ros2 pkg list | grep ydlidar
```

라이다 패키지 + udev:
```bash
sudo apt install -y ros-humble-laser-filters

# /dev/lidar 고정 매핑 (어댑터 CP2102 = 10c4:ea60)
sudo tee /etc/udev/rules.d/99-lidar.rules >/dev/null <<'EOF'
SUBSYSTEM=="tty", ATTRS{idVendor}=="10c4", ATTRS{idProduct}=="ea60", MODE="0666", GROUP="dialout", SYMLINK+="lidar"
EOF
sudo udevadm control --reload-rules && sudo udevadm trigger
ls -l /dev/lidar       # -> ttyUSB0 가리키면 성공
```

연결: `USB_DATA` → Pi USB, `USB_PWR`(micro-USB) → 5V(벅). 스캔 확인:
```bash
ros2 launch peripherals lidar.launch.py        # 터미널 1
# 새 터미널
ros2 topic list | grep scan
ros2 topic echo /scan --once --qos-reliability best_effort   # ranges 숫자 배열 나오면 성공
```

---

## 7. Encoder Motor Module (I2C) 연결 & 브링업

### 7.1 배선

| Pi 핀(물리) | 신호 | 모듈 |
|---|---|---|
| pin 3 (GPIO2) | SDA | SDA |
| pin 5 (GPIO3) | SCL | SCL |
| GND | GND | GND (**공통 필수**) |

전원: 배터리 12V → **모듈 VIN 직결**(모터). Pi는 **벅(12V→5V)** 으로 별도 공급.

### 7.2 I2C 활성화
```bash
# /boot/firmware/config.txt 에 추가 후 재부팅
dtparam=i2c_arm=on
sudo reboot
# 재접속 후 확인
i2cdetect -y 1        # 모듈 연결 시 0x34 표시
```

### 7.3 가상환경 + 의존성
```bash
# Mac → Pi 로 스크립트/노드 전송 (한 폴더에)
ssh piapp@dhl-amr.local 'mkdir -p ~/amr_lidar_viz'
scp /Users/hipark/dev/work/paymentinapp/{amr_env_setup.sh,amr_motor_odom_node.py,wasd_teleop.py,run_mapping.sh,motor_i2c_test.py,module_health.py,motor_init_test.py,motor_load_test.py} \
    piapp@dhl-amr.local:~/amr_lidar_viz/

# Pi
bash ~/amr_lidar_viz/amr_env_setup.sh     # ~/amr_env (--system-site-packages) + smbus2 + 활성화 헬퍼
```
> ⚠️ venv는 반드시 `--system-site-packages` (rclpy 보이게). 매 작업 시:
> `source ~/amr_env/activate_amr.sh` (ROS→워크스페이스→venv 순서 고정).

### 7.4 모터 브링업 테스트 (바퀴 공중에!)
```bash
source ~/amr_env/activate_amr.sh
cd ~/amr_lidar_viz
python3 module_health.py       # I2C/전원 안정성 (성공률 100%, VIN 안정 확인)
python3 motor_init_test.py     # ★ MOTOR_TYPE=3 설정 후 SPEED 모드로 4모터 구동 확인
python3 motor_i2c_test.py      # 인터랙티브 개별/방향 확인
```

> **핵심(실증)**: 이 보드는 ① `MOTOR_TYPE`(reg20=3, JGB37 12V 110RPM)·엔코더극성(reg21)을
> 먼저 등록해야 출력 켜짐, ② 개루프 PWM(reg31) 미작동 → **폐루프 SPEED(reg51)로만 구동**.

---

## 8. Foxglove 시각화

```bash
# Pi
sudo apt install -y ros-humble-foxglove-bridge
ros2 launch foxglove_bridge foxglove_bridge_launch.xml      # 포트 8765
```
- Mac에 **Foxglove Studio** 설치: https://foxglove.dev/download
- Open connection → **"Foxglove WebSocket"**(★Rosbridge 아님) → `ws://dhl-amr.local:8765`
- 3D 패널 Display frame = `map`, 토픽 `/map`·`/scan`·**TF** 추가.

---

## 9. SLAM 매핑 실행 (올인원)

```bash
bash ~/amr_lidar_viz/run_mapping.sh
```
- 라이다 + 오돔 노드 + 정적TF + slam_toolbox + Foxglove bridge 자동 기동.
- `✅ /scan` `✅ /odom` 뜨면 같은 창에서 **WASD 운전**
  (`w/s` 전후, `a/d` 평행, `q/e` 회전, `space`/`k` 정지, `+/-` 속도).
- **Ctrl-C** → 맵 자동 저장 `~/maps/warehouse_<날짜시각>.pgm/.yaml`.

> 첫 주행 전 보정 3종(모터 방향 `DIR` / 오돔 스케일 `PULSES_PER_REV` / 라이다 장착 TF)은
> `README_AMR.md` 6장 참조.

---

## 부록: 자주 겪는 함정

- **64-bit OS 필수**(armhf면 ros-humble 못 찾음).
- **ydlidar는 humble 브랜치**로 빌드(master는 Humble 컴파일 실패).
- **Foxglove는 "Foxglove WebSocket"** 선택(Rosbridge 고르면 연결 실패).
- **venv는 `--system-site-packages`**(아니면 rclpy import 실패).
- **준비 확인은 `ros2 topic list`** 로(‘topic echo --once’는 콜드스타트로 거짓 ‘없음’).
- **유령 노드**: 같은 `ROS_DOMAIN_ID=42` 쓰는 다른 머신 있으면 토픽 혼선 → 격리 필요.
