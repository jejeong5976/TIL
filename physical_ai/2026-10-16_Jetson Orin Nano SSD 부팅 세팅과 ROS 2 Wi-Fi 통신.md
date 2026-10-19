# 🗓️ TIL: Jetson Orin Nano SSD 부팅 세팅과 ROS 2 Wi-Fi 통신

### 핵심 키워드

#JetsonOrin  #SDKManager #NVMe #SMART #ROS2Lyrical

---

## 오늘 배운 것

### 중고 NVMe SSD 고르기와 확인

- **Orin Nano 개발자 키트의 M.2 슬롯** (보드 뒷면 Key M 2개)

| 슬롯 | 규격 | 비고 |
| --- | --- | --- |
| 2280 | PCIe Gen3 x4 | 부팅용 SSD는 여기에 장착 |
| 2230 | PCIe Gen3 x2 | |

- **NVMe만 인식됨** ⭐
	- 모양이 같은 M.2 **SATA** SSD는 인식되지 않음
	- 홈 1개(M-key)면 대부분 NVMe, 홈 2개(B+M key)면 대부분 SATA
- **사용한 SSD**: Samsung PM991a 256GB (`MZVLQ256HBJD-00B`)
	- 노트북 OEM 제품, DRAM-less(HMB). 리눅스 nvme 드라이버에서 정상 동작

- **SMART 확인** (`nvme-cli`는 JetPack에 기본 포함)

```bash
sudo nvme smart-log /dev/nvme0n1
sudo nvme error-log /dev/nvme0n1 -e 8 | grep error_count
```

| 항목 | 정상 기준 | 오늘 값 |
| --- | --- | --- |
| `critical_warning` | 0 | 0 |
| `media_errors` | 0 | 0 |
| `available_spare` | 100%에 가깝게 (threshold 10%) | 100% |
| `percentage_used` | 낮을수록 좋음 | 0% |
| `Data Units Written` | 256GB급 수명 약 150TBW | 90.68 GB |
| `power_on_hours` | 참고용 | 0 (1시간 넘게 켰는데도 0 → 이 제품은 신뢰도 낮음) |

- SMART 누적값은 **포맷해도 초기화되지 않는** 컨트롤러 기록이라 중고 판단 근거로 쓸 수 있음 ⭐
- 속도 테스트는 반드시 **파일에** 쓰기 (`of=`를 디스크 장치로 주면 SSD 전체가 지워짐)

```bash
sudo dd if=/dev/nvme0n1 of=/dev/null bs=4M count=1024 iflag=direct      # 읽기
dd if=/dev/zero of=~/ddtest.bin bs=4M count=512 oflag=direct conv=fsync # 쓰기
```

- 결과: PCIe `8.0 GT/s` x4, 읽기 2.4 GB/s / 쓰기 1.0 GB/s, 온도 40°C → 테스트 직후 57°C

---

### 복구 모드(Recovery Mode)와 SDK Manager 플래시

- **복구 모드**: Jetson이 저장 장치로 부팅하지 않고 USB로 PC의 명령을 기다리는 상태
	- SSD가 비어 있어 부팅할 OS가 없을 때, PC가 SSD에 직접 기록하려고 사용
	- Jetson은 **전원이 들어오는 순간에만** `FC REC` 핀을 확인함 → 순서가 중요 ⭐

```text
전원 분리 → SSD 장착 → FC REC ↔ GND 쇼트 → USB-C ↔ PC → 전원 연결
```

| `lsusb` 출력 | 의미 |
| --- | --- |
| `0955:7523 NVIDIA Corp. APX` | 복구 모드 (Orin Nano 8GB) |
| `0955:7020 ... L4T running on Tegra` | 정상 부팅 |



---

### ROS 2 Lyrical 설치

- 노트북·Jetson·라즈베리파이는 **같은 ROS 2 배포판**이어야 안정적으로 통신됨 → 노트북에 맞춰 Lyrical
- ROS 공식 저장소(`packages.ros.org`)에는 Ubuntu 24.04 **arm64용 Lyrical이 없음** (Jazzy, Kilted, Rolling만 있음)
	- 노트북의 Lyrical도 사실 **NVIDIA Isaac ROS 저장소**에서 설치된 것이었음 ⭐

```bash
curl -fsSL https://isaac.download.nvidia.com/isaac-ros/repos.key | sudo gpg --dearmor --yes -o /usr/share/keyrings/nvidia-isaac-ros.gpg
echo "deb [signed-by=/usr/share/keyrings/nvidia-isaac-ros.gpg] https://isaac.download.nvidia.com/isaac-ros/ubuntu/main noble main" | sudo tee /etc/apt/sources.list.d/nvidia-isaac-ros-buildfarm.list
sudo apt update && sudo apt install -y ros-lyrical-ros-base
```

- 미들웨어 `rmw_fastrtps_cpp`, `ROS_DOMAIN_ID=num` (노트북과 같은 값)

---

```bash
# Jetson ~/.bashrc
export ROS_STATIC_PEERS="10.2.12.120"   # 노트북 IP
# 노트북 ~/.bashrc
export ROS_STATIC_PEERS="10.2.12.123"   # Jetson IP
```

- 한쪽만 지정해도 통신됨 (지정한 쪽이 먼저 연결을 걸어 줌)
- `~/.bashrc`를 고쳐도 **이미 열린 터미널에는 적용 안 됨** → `source ~/.bashrc` 또는 새 터미널 ⭐
	- 예전 터미널에서 실행한 `pub`에 `ROS_STATIC_PEERS`가 없어서 또 안 됐음

---

## 알게 된 것

- SSD 플래시: Storage Device를 `NVMe`로 바꾸기, 복구 모드 진입 순서(쇼트 → USB → 전원)
- SSD는 SMART의 `critical_warning`, `media_errors`, `percentage_used`, `Data Units Written`으로 판단. 누적값은 포맷해도 안 지워짐
- 학원 Wi-Fi는 멀티캐스트가 막혀서 ROS 2 자동 탐색이 안 됨 → `ROS_STATIC_PEERS`에 IP 지정

---

## 다음에 할 것

- Nav2·realsense2_camera·v4l2_camera 설치
- JetPack 7.2용 PyTorch 휠 확인 후 Ultralytics 설치 → YOLO TensorRT 엔진 빌드