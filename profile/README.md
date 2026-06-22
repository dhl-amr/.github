# AMR (지상 자율주행 로봇)

메카넘 4WD 로봇이 라이다로 주변을 스캔해 **지도를 만들고(SLAM)**, 그 지도로 자율주행한다.
드론×AMR 재고 스캔 솔루션의 지상 대차 파트.

```
[LiDAR] ─USB─┐
             ├─▶ Raspberry Pi 4  (Ubuntu 22.04 + ROS2 Humble)
[모터모듈]─I2C─┘        │  지도 생성 · 경로 주행
   │                   └─▶ Foxglove (Mac에서 화면으로 확인)
메카넘 4륜
```

## 구성

- **두뇌**: Raspberry Pi 4
- **모터**: Encoder Motor Module (I2C `0x34`) → 메카넘 4륜 + 엔코더
- **센서**: YDLIDAR G4 (2D 라이다)
- **전원**: 배터리 12V → 모터 직결 + 벅컨버터(5V) → Pi

---

## 빠른 시작

**1. 처음이면 셋업부터** → [`docs/rpi4_setup.md`](docs/rpi4_setup.md) (라즈베리파이 설치~빌드 전체)

**2. 매핑 실행**
```bash
bash ~/amr_lidar_viz/run_mapping.sh
```
운전: `w/s` 전후 · `a/d` 평행 · `q/e` 회전 · `space` 정지 · **Ctrl-C** = 지도 저장(`~/maps/`)

**3. 화면으로 보기** (Mac Foxglove Studio)
- Open connection → **Foxglove WebSocket** → `ws://dhl-amr.local:8765`
- 3D 패널에서 frame = `map`, 토픽 `/map` `/scan` 추가

---

## 파일

| 파일 | 설명 |
|---|---|
| `run_mapping.sh` | **매핑 실행** (라이다+모터+SLAM+Foxglove 한 번에) |
| `amr_motor_odom_node.py` | 모터 구동 + 엔코더 위치추정 노드 |
| `wasd_teleop.py` | WASD 키보드 운전 |
| `amr_env_setup.sh` | 가상환경 + 의존성 설치 |
| `module_health.py` / `motor_init_test.py` / `motor_i2c_test.py` / `motor_load_test.py` | 모터·전원 점검 도구 |

---

## 모터 점검 (조립 직후, 바퀴 공중에)

```bash
source ~/amr_env/activate_amr.sh
cd ~/amr_lidar_viz
python3 module_health.py       # I2C·전원 안정성
python3 motor_init_test.py     # 4모터 회전 확인
python3 motor_i2c_test.py      # 개별 구동·방향 확인
```

## 첫 주행 보정 (지도가 깨지면)

`amr_motor_odom_node.py` 위쪽 값을 고친다.

1. **방향** — `w` 줬을 때 거꾸로 도는 바퀴 → `DIR`을 `-1`
2. **거리** — 1m 갔는데 `/odom`이 다르면 → `PULSES_PER_REV` 조정
3. **라이다 위치** — `run_mapping.sh`의 static TF `--z`(높이)·`--yaw`(정면 방향)

---

## 메모

- 모터는 SPEED 모드(I2C reg 51)로 구동. 처음에 모터 종류(reg 20 = 3) 등록이 필요한데 노드가 자동으로 한다.
- 막히면 `docs/rpi4_setup.md` 맨 아래 "막힐 때" 참고.
