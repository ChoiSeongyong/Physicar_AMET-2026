# PhysiCar Autonomous Driving — AMET 2026

2026 자율주행 해커톤 경진대회(AMET 2026)의 PhysiCar 주행 환경을 대상으로 개발한 자율주행 제어 프로젝트입니다. 카메라 영상으로 주행 영역과 방향을 추정하고, LiDAR로 콘을 감지해 속도와 조향을 보정합니다. 시뮬레이터에서는 제공되는 경로·차량 상태 API를 사용하는 별도 제어기도 포함합니다.

2026 AMET은 한국자율주행산업협회가 안내한 자율주행 해커톤으로, 2026년 8월 25일부터 27일까지 코엑스에서 열린 자율주행모빌리티산업전 기간에 진행됐습니다.

- 대회 안내: [한국자율주행산업협회](https://www.kaami.or.kr/)

## 핵심 구현

- **카메라 기반 주행 영역 추정**: HSV 기반 마스크와 연결 성분 분석으로 차량 아래의 주행 가능 영역을 찾고, 여러 영상 행의 중심을 다항식으로 근사해 횡방향 오차와 굽힘 정도를 계산합니다.
- **차선 단서와 복구 처리**: 노란 중앙선, 점선 연결, 화면 밖으로 잘린 도로, 끊어진 도로 구간을 보조 단서로 사용하고, 주행 영역을 잃은 경우 화면 안의 다른 도로 후보를 찾아 복귀 방향을 정합니다.
- **LiDAR 기반 콘 회피**: 전방 스캔에서 콘 후보의 거리·방향·폭을 계산하고, 접근 거리와 차량 속도에 따라 회피 조향과 감속 상한을 결정합니다.
- **카메라–LiDAR 선택적 결합**: `PC_CONE_MODEL`이 지정된 경우 카메라 검출 결과와 LiDAR 콘 후보를 방위각으로 연결합니다. 모델을 지정하지 않으면 LiDAR만 사용합니다.
- **상태 기반 제어**: 주행 영역의 오차, 곡률, 가시 거리와 콘 회피 결과를 조합해 조향각과 속도를 계산합니다. 급격한 조향 방향 반전을 억제하고 센서 프레임 누락 시 직전 명령을 제한적으로 유지합니다.
- **v6-3 대각선 진입 보조**: 기본 제어기가 이미 선택한 방향과 영상의 완만한 대각선 방향이 여러 프레임 동안 일치할 때만 최대 2.8°의 제한된 조향을 추가합니다. 급커브, 콘 회피, 복구, 차선 미검출 상황에서는 즉시 기본 제어로 돌아갑니다.

기본 센서 주행은 딥러닝 모델이 아니라 OpenCV 기반 영상 처리와 기하학적·규칙 기반 제어를 사용합니다. 저장소의 ONNX 모델과 학습 도구는 기존 제어 명령을 제한적으로 보정하는 실험용 경로이며, v6-3 기본 실행의 필수 요소가 아닙니다.

## 실행 환경

- Python 3
- Bash
- PhysiCar 센서 또는 시뮬레이터 API 서버
- 필수 패키지: `numpy`, `opencv-python`, `requests`
- 선택 패키지: `ultralytics`(카메라 콘 검출), `onnxruntime`(ONNX adviser 실험)

```bash
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install --upgrade pip
python3 -m pip install numpy opencv-python requests
```

선택 기능이 필요한 경우에만 추가합니다.

```bash
python3 -m pip install ultralytics onnxruntime
```

PhysiCar 서버와 모델 가중치는 이 저장소에 포함되어 있지 않습니다. 기본 API 주소는 `http://localhost`이며 `PHYSICAR_URL`로 변경할 수 있습니다.

## 실행 방법

### 시뮬레이터 경로 기반 제어

`race/run.py`는 `/sim/api/route`, `/sim/api/state`, `/sim/api/objects`를 제공하는 시뮬레이터에서 사용합니다.

```bash
PC_TARGET=sim python3 race/run.py
```

이 경로는 Pure Pursuit 형태의 look-ahead 조향, 경로 곡률 기반 감속, 정적 장애물 위치를 이용한 회피를 수행합니다. 실차 센서 기반 제어와 입력 조건이 다릅니다.

### 센서 기반 기본 제어

```bash
PC_MAX_SECONDS=60 python3 autodrive.py
```

카메라·LiDAR·오도메트리 API를 이용하며, `PC_MAX_SECONDS=0`이면 코드 자체의 시간 제한을 두지 않습니다.

### v6-3 실행

```bash
PC_TARGET=sim PC_MAX_SECONDS=30 source ./run_real_upgrade_v6_3.sh
```

실제 차량에 명령을 보내는 실행은 명시적인 확인값이 필요합니다.

```bash
PC_REAL_CONFIRM=YES source ./run_real_upgrade_v6_3.sh
```

기본 설정은 최대 속도 0.80 m/s, 최소 속도 0.40 m/s이며 환경변수로 변경할 수 있습니다.

## 주요 파일

| 경로 | 역할 |
| --- | --- |
| `autodrive.py` | 센서 기반 기본 자율주행 루프 |
| `autodrive_v6_3.py` | 기본 제어에 제한된 대각선 진입 보조를 추가한 최신 실험 경로 |
| `drive/lane.py` | 주행 영역, 차선 단서, 도로 연속성과 복구 방향 추정 |
| `drive/cones.py` | LiDAR 콘 감지와 회피 조향·감속 계산 |
| `drive/control.py` | 차선 추정값을 속도·조향 명령으로 변환 |
| `drive/fusion.py` | 선택적 카메라 검출과 LiDAR 후보 결합 |
| `drive/light.py` | 지정 ROI의 HSV 색상으로 출발 신호 판별 |
| `drive/robot.py` | 카메라·LiDAR·IMU·오도메트리 조회와 차량 명령 API |
| `drive/health.py` | 센서 호출과 제어 루프 처리 속도 점검 |
| `drive/v6_3_diagonal_assist.py` | 완만한 대각선 구간의 제한적 조향 보조 |
| `race/run.py` | 시뮬레이터 경로 추종 및 실행 모드 선택 |
| `race/config.py` | 경로 추종·속도·회피 파라미터 |
| `t_run.sh` | 로그, PID, 재시작과 종료 시 차량 정지를 관리하는 실행 래퍼 |
| `run_real_upgrade_v6_3.sh` | v6-3 파라미터와 실차 실행 확인 절차 |
| `tools/` | telemetry 분석, 모델 학습과 보조 모듈 테스트 도구 |
| `models/` | 실험용 ONNX adviser와 메타데이터 |

## 주요 환경변수

| 변수 | 설명 |
| --- | --- |
| `PHYSICAR_URL` | PhysiCar API 주소 |
| `PC_TARGET` | `sim`, `real`, `auto` 실행 대상 선택 |
| `PC_SPEED_MAX`, `PC_SPEED_MIN` | 센서 기반 속도 범위 |
| `PC_MAX_SECONDS` | 최대 실행 시간 |
| `PC_WAIT_GREEN` | 출발 신호 대기 여부 |
| `PC_AVOID_CONES` | 콘 회피 사용 여부 |
| `PC_CONE_MODEL` | 선택적 카메라 콘 검출 모델 경로 |
| `PC_TELEMETRY_CSV` | 주행 telemetry 저장 경로 |
| `PC_DIAGONAL_ASSIST` | v6-3 대각선 진입 보조 사용 여부 |

## 실행 전 확인

- `run.sh`는 대회 환경의 `/home/physicar/physicar_ws` 경로를 전제로 합니다. 다른 위치에서는 위의 Python 명령이나 저장소 위치를 자동으로 찾는 실행 스크립트를 사용합니다.
- 센서 지연과 제어 주기는 회피 거리에 직접 영향을 줍니다. `python3 drive/health.py`로 현재 환경의 처리 속도를 점검할 수 있습니다.
- 실제 차량에서는 저속·짧은 시간으로 먼저 동작을 확인합니다.
