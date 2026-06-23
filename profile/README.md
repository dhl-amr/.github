# DHL AMR (지상 자율주행 로봇)

메카넘 4WD 로봇이 라이다로 주변을 스캔해 **지도를 만들고(SLAM)**, 그 지도로 주행한다.
드론×AMR 재고 스캔 솔루션의 지상 대차 파트.

## 시스템 구성

```
[YDLIDAR G4] ─USB──┐
                   ├─▶ Raspberry Pi 4 ──▶ Foxglove (Mac에서 화면 확인)
[Encoder Motor]─I2C─┘   지도 생성 · 주행
   │
메카넘 4륜
```

| 장치 | 역할 |
|------|------|
| Raspberry Pi 4 | 두뇌 — 지도 생성·주행 (Ubuntu 22.04 + ROS2 Humble) |
| Encoder Motor Module | 모터 구동 + 엔코더 읽기 (I2C `0x34`) |
| YDLIDAR G4 | 2D 라이다 (`/scan`) |

전원: 배터리 12V → 모터 직결 + 벅컨버터(5V) → Pi.

## 처음이라면 여기부터

단계별 가이드는 **`docs` 저장소**에 있다. **01 → 04 순서**로 따라 하면 로봇이 지도를 그린다.

1. [라즈베리파이 셋업](../../docs/01-pi-setup.md)
2. [라이다 연결](../../docs/02-lidar.md)
3. [모터 연결 & 점검](../../docs/03-motor.md)
4. [운전 & 매핑](../../docs/04-drive-mapping.md)

코드는 → [`dhl-amr-rpi4`](../../dhl-amr-rpi4) 저장소

## 진행 상황

- [x] RPi4 + ROS2 Humble 셋업
- [x] 라이다 `/scan`
- [x] 모터 구동(I2C) + 엔코더 + WASD 운전
- [x] 실시간 SLAM 매핑
- [ ] **방향·거리·라이다 위치 보정** ← 현재 단계
- [ ] 자율 매핑·순찰 (`warehouse_explorer`, 향후)
