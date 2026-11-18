# DA-DAKA M10 GPS 원시 MSL 소프트웨어 현장시험

- 예정일: 2026-08-05
- 시험 장소: 실외 개방 공간
- 대상 기체: DA-DAKA
- 비행제어기: Pixhawk 4 / PX4
- GPS: Holybro M10 Pixhawk Standard GPS (u-blox M10)
- 동반 컴퓨터: Raspberry Pi 5 / ROS 2 Jazzy / MAVROS

---

## 1. 이 문서를 먼저 읽을 Codex에게

이 문서는 현장에서 Raspberry Pi 5의 Codex가 현재 문제와 시험 순서를 빠르게
파악하도록 만든 인계 문서다.

이번 작업의 목적은 **GPS 원시 MSL 고도 자체가 소프트웨어 설정에 따라
개선되는지 확인하는 것**이다. PX4 Local Z가 안정적인지만 확인하는 시험이
아니다.

Codex는 다음 원칙을 지킨다.

1. 처음에는 모든 값을 읽기 전용으로 확인한다.
2. 사용자의 명시적인 지시 없이 PX4 파라미터를 변경하지 않는다.
3. 한 번에 파라미터 하나만 변경한다.
4. 변경 전 값, 변경 후 값, 재부팅 여부를 반드시 기록한다.
5. 파라미터가 현재 PX4 펌웨어에 없으면 추측해서 대체하지 않고 `지원 안 됨`으로 기록한다.
6. 각 시험은 Disarm 및 `ON_GROUND` 상태에서 수행한다.
7. 이번 원시고도 시험 중에는 Arm, 이륙, 모드 변경, setpoint 송신 및
   `/mission/start` 호출을 하지 않는다.
8. PX4 안전검사나 Mission Manager health 검사를 우회하지 않는다.
9. GPS 원시값과 PX4 추정값을 구분하여 기록한다.
10. 시험 종료 후 운용 파라미터가 원래 값으로 복구됐는지 재조회한다.

---

## 2. 현재 확인된 문제

M10 GPS 원시 MSL 고도는 정지 상태에서도 다음과 같은 이상을 보였다.

- 비행 전 지상 60초: 약 `+4.553 m`
- 비행 후 지상 60초: 약 `-5.111 m`
- 비행 전 지상 GPS MSL 전체 범위: 약 `4.618 m`
- 비행 후 지상 GPS MSL 전체 범위: 약 `6.979 m`
- Mission ABORT 후 지상 GPS MSL 전체 범위: 약 `5.781 m`

이는 한두 샘플이 순간적으로 튀는 스파이크가 아니다. 약 10 Hz 원본에서
작은 변화가 여러 샘플 연속으로 누적되어 수십 초 동안 수 m 상승하거나
하락하고, 이후 다시 방향을 바꾸는 형태다.

이 문서에서는 다음과 같이 부른다.

> 정지 상태에서 발생하는 비주기적·대진폭 GPS 수직 위치해 변동

문제 구간에서도 다음 상태가 확인됐다.

- 위성 수 약 `24~29`
- GPS `v_acc` 대체로 약 `0.36~0.56 m`
- `gps_glitch=false`
- 큰 변동 구간에서 `fix_type=4` 유지
- 기체는 Disarm 및 `ON_GROUND`
- LiDAR는 지면까지 약 `0.26 m`로 일정

따라서 위성 수, Fix, `v_acc`, glitch 상태만으로는 해당 수직오차를
검출하지 못했다.

---

## 3. 현재 유지해야 하는 PX4 수직 위치 설정

다음 설정은 GPS 원시고도를 고친 것은 아니지만, 잘못된 GPS 고도가 PX4
Local Z를 끌고 가지 못하도록 하는 현재의 운용 설정이다.

```text
EKF2_GPS_CTRL=5
EKF2_HGT_REF=0
EKF2_BARO_CTRL=1
EKF2_RNG_CTRL=1
```

의미:

- GPS 수평 위치와 속도는 사용한다.
- GPS 고도는 EKF 수직 위치 융합에서 제외한다.
- 기압계를 높이 기준으로 사용한다.
- 거리 센서는 PX4가 실제 거리 데이터를 받는 경우에만 조건부로 사용한다.

이번 현장시험에서는 `EKF2_GPS_CTRL=7`로 되돌리지 않는다.

---

## 4. 이번 시험에서 변경하지 않는 항목

### 동적 모델 비교 제외

이번 현장시험에서는 다음 파라미터를 변경하지 않는다.

```text
GPS_UBX_DYNMODEL
```

현재 값이 비행용 기본값인 `7 (Airborne <2g)`인지 읽기 전용으로만 확인한다.
`Stationary` 모델과의 비교시험은 수행하지 않는다.

### 그 밖의 변경 금지

- `EKF2_GPS_CTRL=7` 변경 금지
- EKF 및 GPS 사전검사 완화 금지
- `ignored_unhealthy_sensor_mask` 추가 확대 금지
- GPS 원시 MSL 값을 다른 값으로 덮어써서 PX4에 재전송하지 않기
- 여러 GPS 파라미터를 동시에 바꾸지 않기
- 단순 필터 결과만 보고 GPS 원시고도가 해결됐다고 판정하지 않기

---

## 5. 실내에서 준비할 것

실내에서는 GPS 정상 여부를 판정하지 않는다. 다음 준비만 하고 실외로
이동한다.

### 5.1 파일과 저장 공간

- QGC PX4 전체 파라미터 백업 파일
- rosbag 저장 공간 확인
- 시험 결과를 적을 메모 또는 CSV
- 이 문서

### 5.2 기록할 ROS 2 토픽

기본 기록 대상:

```text
/mavros/gpsstatus/gps1/raw
/mavros/global_position/raw/fix
/mavros/global_position/raw/satellites
/mavros/altitude
/mavros/local_position/pose
/mavros/local_position/velocity_local
/mavros/estimator_status
/mavros/extended_state
/mavros/state
/mavros/statustext/recv
/distance/filtered
```

위성별 신호 토픽은 설치된 MAVROS/PX4 버전에 따라 이름이 다를 수 있다.
추측하지 말고 다음처럼 실제 토픽을 확인한다.

```bash
ros2 topic list | grep -Ei 'gps|gnss|sat'
```

### 5.3 단위

`/mavros/gpsstatus/gps1/raw` 기준:

- `alt`: MSL 고도, 단위 mm → `1000`으로 나누면 m
- `v_acc`: 수직 불확실도 추정, 단위 mm → `1000`으로 나누면 m
- `h_acc`: 수평 불확실도 추정, 단위 mm → `1000`으로 나누면 m
- `epv`: 이 메시지에서는 VDOP 계열 값이며 MSL 고도 오차 자체가 아님
- `satellites_visible`: 위성 수
- `fix_type`: GPS Fix 종류

---

## 6. 실외 공통 시험 조건

모든 시험은 같은 장소와 같은 기체 상태에서 수행한다.

1. 기체를 평평한 곳에 놓는다.
2. Pixhawk와 Raspberry Pi를 켠다.
3. MAVROS 연결을 확인한다.
4. `armed=false`, `ON_GROUND`를 확인한다.
5. GPS Fix가 3D 이상이고 데이터가 연속 수신되는지 확인한다.
6. 기체를 움직이지 않는다.
7. 설정 변경 시험마다 필요한 PX4 재부팅을 수행한다.
8. 재부팅 후 GPS와 MAVROS 연결이 회복될 때까지 기다린다.
9. 일반 설정 변경은 GPS Fix 확인 후 30초간 기체를 움직이지 않고 기다린다.
10. `GPS_CFG_WIPE` 또는 SBAS 변경 후에는 GPS Fix 확인 후 60초간 기다린다.
11. 대기가 끝나면 60초를 기록한다.
12. 시험 사이에 기체 위치와 주변 배치를 바꾸지 않는다.

60초를 사용하는 이유는 기존 이상이 30~60초 안에도 나타났고 실외 고온
환경에서 시험자의 노출 시간을 줄여야 하기 때문이다. 10분 기록은 이번
비교시험에 사용하지 않는다.

### 6.1 더운 날씨에서의 시험 축소 원칙

- 현장 필수 시험은 `시험 0`, `시험 1`, `시험 2`, `시험 4`, `시험 5`,
  `시험 8`이다.
- `시험 3`, `시험 6`, `시험 7`은 시간과 온도 여건이 허용할 때만 수행한다.
- 시험자나 장비의 과열 우려가 있으면 즉시 그 단계에서 중단하고 결과에
  `고온으로 중단`이라고 기록한다.
- 시간을 줄이기 위해 복구·재부팅·안전 확인을 생략하지 않는다.

---

## 7. 각 시험에서 계산할 지표

모든 시험 결과는 동일한 방식으로 계산한다.

### 필수 지표

```text
첫 GPS MSL (m)
마지막 GPS MSL (m)
60초 순변화 (m)
60초 전체 범위 = 최댓값 - 최솟값 (m)
30초 rolling 최대 범위 (m)
30초 최대 상승량 (m)
30초 최대 하락량 (m)
10초 최대 절대 변화율 (m/s)
v_acc 최솟값/중앙값/최댓값 (m)
VDOP 계열 값 최솟값/중앙값/최댓값
위성 수 최솟값/중앙값/최댓값
fix_type 전환 횟수
GPS glitch 발생 여부
Local Z 전체 범위 (m)
LiDAR 전체 범위 (m)
```

### 임시 비교 기준

- 프로젝트의 기존 임시 목표: 정지 60초 GPS MSL 범위 `0.5 m 이하`
- 이 값은 M10 제조사 보증 기준이 아니라 DA-DAKA 비교용 임시 기준이다.
- 절대 PASS/FAIL뿐 아니라 이전 설정보다 반복적으로 개선됐는지도 본다.
- 한 번만 좋아졌으면 해결로 판정하지 않고 다른 시각에 재시험한다.

### 비교시험 공통 구조

`GPS_CFG_WIPE`를 제외한 설정 비교는 다음 `A-B-A` 순서를 지킨다.

```text
A1: 원래 설정 60초 기록
B : 시험 설정 적용·재부팅·대기 후 60초 기록
A2: 원래 설정 복구·재부팅·대기 후 60초 기록
```

B가 A1과 A2 모두보다 반복적으로 좋아야 개선 가능성이 있다고 판단한다.
각 시험이 끝나면 다음 시험으로 넘어가기 전에 반드시 원래 값으로 복구한다.

---

## 8. 현장 시험 순서

### 시험 0 — 현재 값 읽기 및 백업 확인

먼저 QGC에서 현재 PX4 버전과 다음 값을 읽어 기록한다.

```text
PX4 firmware version
GPS_1_PROTOCOL
SER_GPS1_BAUD
GPS_1_GNSS
GPS_UBX_MODE
GPS_UBX_DYNMODEL
GPS_UBX_RATE
GPS_CFG_WIPE
GPS_SAT_INFO
GPS_UBX_MIN_ELEV
GPS_UBX_MIN_CNO
EKF2_GPS_CTRL
EKF2_HGT_REF
EKF2_BARO_CTRL
EKF2_RNG_CTRL
```

기대 확인값:

```text
GPS_1_PROTOCOL=1       # u-blox
SER_GPS1_BAUD=0        # Auto
GPS_UBX_MODE=0         # Default
GPS_UBX_DYNMODEL=7     # 읽기만 하고 변경하지 않음
EKF2_GPS_CTRL=5
EKF2_HGT_REF=0
EKF2_BARO_CTRL=1
```

현재 펌웨어에 없는 파라미터는 `없음`으로 기록하고 해당 시험을 건너뛴다.

### 시험 1 — 당일 소프트웨어 기준값

목적: 같은 시각과 장소에서 이후 설정 시험과 비교할 짧은 기준값을 만든다.

- 파라미터 변경 없음
- Disarm, `ON_GROUND`
- 60초 기록
- 7절 지표 계산

이 시험은 과거 하드웨어 기준선 반복이 아니라, 오늘 수행할 A/B 설정 비교의
기준값이다.

### 시험 2 — u-blox 저장 설정 초기화

조건: QGC에 `GPS_CFG_WIPE`가 실제로 존재할 때만 수행한다.

절차:

1. PX4 파라미터 백업 파일이 있는지 확인한다.
2. 사용자 승인 후 `GPS_CFG_WIPE=1`로 설정한다.
3. Pixhawk를 재부팅한다.
4. 부팅과 GPS 인식을 확인한다.
5. `GPS_CFG_WIPE=0`으로 되돌린다.
6. 다시 재부팅한다.
7. GPS 연결과 관련 파라미터를 재조회한다.
8. GPS Fix 확인 후 60초 대기한다.
9. 60초 기록 및 지표 계산을 수행한다.
10. 초기화 후 상태를 이후 설정시험의 새로운 기준 A로 사용한다.

`GPS_CFG_WIPE`가 없거나 GPS 인식 문제가 발생하면 임의의 대체 초기화를 하지
않고 중단한다.

`GPS_CFG_WIPE`는 수신기 내부 저장 설정을 지우는 작업이므로 일반적인
가역 A-B-A 시험으로 취급하지 않는다. PX4 파라미터 백업만으로 초기화 전
수신기 내부 설정을 완전히 되돌릴 수 있다고 가정하지 않는다.

### 시험 3 — GPS 출력률 5 Hz 비교

조건: QGC에 `GPS_UBX_RATE`가 존재할 때만 수행한다.

절차:

1. 현재 값을 기록한다.
2. 원래 설정 A1을 60초 기록한다.
3. 사용자 승인 후 `GPS_UBX_RATE=5`로 변경한다.
4. Pixhawk를 재부팅하고 공통 대기 조건을 지킨다.
5. GPS 토픽 실제 수신률을 확인한다.
6. 시험 설정 B를 60초 기록한다.
7. 결과와 관계없이 원래 값으로 복구하고 재부팅한다.
8. 공통 대기 후 복구 설정 A2를 60초 기록한다.

판정:

- 5 Hz에서 60초 MSL 범위와 30초 최대 상승·하락이 줄었는지 확인한다.
- 위성 수와 VDOP가 나빠지지 않았는지 확인한다.
- 기존 출력률과 5 Hz 데이터를 모두 1초 단위 중앙값으로 변환한 뒤 같은
  1 Hz 데이터끼리 비교한다.
- 한 번의 결과로 영구 설정을 결정하지 않는다.

### 시험 4 — 위성별 정보 기록

조건: `GPS_SAT_INFO`가 존재할 때 수행한다.

1. 현재 값을 기록한다.
2. 사용자 승인 후 `GPS_SAT_INFO=1`로 변경한다.
3. 재부팅한다.
4. 위성별 식별자, 사용 여부, 신호 세기(C/N0 또는 SNR), 고도각을 기록한다.
5. QGC MAVLink Inspector, PX4 ULog 또는 실제 제공되는 ROS 2 토픽을 사용한다.
6. 위성별 정보가 노출되지 않으면 추측하지 말고 `수집 불가`로 기록한다.
7. 필요한 정보를 수집한 뒤 원래 값으로 복구하고 재부팅한다.

이 단계에서는 GPS 원시고도를 개선하는 설정을 추가로 바꾸지 않는다.

### 시험 5 — 낮은 고도각 위성 제외 비교

조건: `GPS_UBX_MIN_ELEV`가 존재할 때만 수행한다.

1. 현재 값을 기록한다.
2. 원래 설정 A1을 60초 기록한다.
3. 사용자 승인 후 시험값 `10 deg`를 적용한다.
4. 재부팅하고 공통 대기 조건을 지킨다.
5. 시험 설정 B를 60초 기록한다.
6. 위성 수, VDOP, Fix와 원시 MSL 범위를 확인한다.
7. 결과와 관계없이 원래 값으로 복구하고 재부팅한다.
8. 공통 대기 후 복구 설정 A2를 60초 기록한다.

중단·복구 조건:

- Fix 상실
- 위성 수 급감
- VDOP 뚜렷한 악화
- GPS MSL 범위 증가
- GPS 데이터 누락

이상이 있으면 B 기록을 중단하고 즉시 원래 값으로 복구한 뒤 재부팅한다.
이번 시험에서는 다른 고도각 값을 연속으로 무작정 시도하지 않는다.

### 시험 6 — 약한 위성 제외 비교

조건:

- `GPS_UBX_MIN_CNO`가 존재해야 한다.
- 시험 4에서 실제 위성별 신호 분포를 수집했어야 한다.

신호 분포를 보지 못했다면 이 시험은 건너뛴다. 임의로 `25`, `30` 같은 값을
넣지 않는다.

값을 정할 때는 약한 신호 일부를 제외하더라도 충분한 위성 수와 양호한 VDOP가
남도록 해야 한다. 한 값만 시험하며 다른 파라미터와 동시에 변경하지 않는다.

절차:

1. 사용자에게 근거가 포함된 시험값을 제안한다.
2. 원래 설정 A1을 60초 기록한다.
3. 승인 후 한 값만 적용한다.
4. 재부팅하고 공통 대기 조건을 지킨다.
5. 시험 설정 B를 60초 기록한다.
6. 결과와 관계없이 원래 값으로 복구하고 재부팅한다.
7. 공통 대기 후 복구 설정 A2를 60초 기록한다.

위성 수 또는 VDOP가 악화되면 B 기록을 중단하고 즉시 복구한다.

### 시험 7 — SBAS 사용 여부 비교

이 시험은 현재 활성 GNSS 위성군을 확인할 수 있을 때만 수행한다.

기존 큰 GPS MSL 변동은 `fix_type=4`가 유지되는 동안에도 발생했다. 따라서
`fix_type 3↔4` 전환 순간이 직접 원인이었던 것은 아니다. 하지만 SBAS를
사용한 해 자체와 관련됐는지 확인하기 위한 비교시험은 가능하다.

주의:

- `GPS_1_GNSS=0`이면 수신기 기본 위성군을 의미한다.
- 현재 활성 위성군을 확인하지 못한 상태에서 임의의 bitmask를 넣지 않는다.
- SBAS만 제외하고 나머지 주요 위성군은 가능한 한 동일하게 유지해야 한다.
- 위성 수가 크게 줄거나 VDOP가 악화되면 원래 설정으로 복구한다.

절차:

1. 현재 `GPS_1_GNSS`와 활성 위성군을 기록한다.
2. SBAS 포함 상태 A1을 60초 기록한다.
3. 사용자 승인 후 SBAS만 제외한 구성을 적용한다.
4. 재부팅하고 GPS Fix 확인 후 60초 대기한다.
5. SBAS 제외 상태 B를 60초 기록한다.
6. 원래 설정으로 복구하고 재부팅한다.
7. GPS Fix 확인 후 60초 대기하고 복구 상태 A2를 60초 기록한다.
8. A1·B·A2의 60초 MSL 범위, 30초 최대 상승·하락, 위성 수와 VDOP를
   비교한다.

활성 위성군을 확실히 확인하지 못하면 이 시험은 현장에서 강행하지 않는다.

### 시험 8 — 최종 복구 및 재확인

시험 종료 후 다음을 수행한다.

1. 영구 적용이 승인되지 않은 GPS 시험값을 모두 원래 값으로 복구한다.
2. Pixhawk를 재부팅한다.
3. 다음 운용 설정을 재조회한다.

```text
GPS_UBX_DYNMODEL=7
EKF2_GPS_CTRL=5
EKF2_HGT_REF=0
EKF2_BARO_CTRL=1
```

4. `GPS_CFG_WIPE=0`인지 확인한다.
5. Disarm, `ON_GROUND`, MAVROS 연결을 확인한다.
6. 시험별 로그가 정상 종료되어 metadata가 존재하는지 확인한다.
7. 실제 비행 여부는 이 문서의 원시고도 시험 결과만으로 결정하지 않는다.

---

## 9. 시험 결과 기록 양식

| 시험 | 변경 파라미터 | 원래 값 | 시험값 | 기록시간 | 60초 최대 범위 | 30초 최대 상승 | 30초 최대 하락 | v_acc 중앙값 | 위성 수 중앙값 | VDOP 중앙값 | 결과 |
|---|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---|
| 당일 기준 | 없음 | - | - | 60초 | | | | | | | |
| 설정 초기화 후 | GPS_CFG_WIPE | 0 | 1→0 | 60초 | | | | | | | |
| 출력률 비교 | GPS_UBX_RATE | | 5 | 각 60초 | | | | | | | |
| 고도각 비교 | GPS_UBX_MIN_ELEV | | 10 | 각 60초 | | | | | | | |
| 신호 세기 비교 | GPS_UBX_MIN_CNO | | | 각 60초 | | | | | | | |
| SBAS 비교 | GPS_1_GNSS | | | 각 60초 | | | | | | | |

결과에는 반드시 다음 중 하나를 적는다.

- 개선 재현됨
- 한 번만 개선되어 재시험 필요
- 차이 없음
- 악화되어 복구
- 파라미터 없음
- 데이터 수집 불가
- 안전 조건 미충족으로 중단

---

## 10. 현장에서 하지 않을 잘못된 해결법

### 단순 이동평균 또는 저역통과필터

이번 오류는 한두 샘플이 아니라 30~60초 동안 연속으로 이동한다. 필터는
변화를 늦게 따라갈 뿐 결국 잘못된 고도로 이동한다.

### 중앙값 또는 Hampel 필터만 사용

단발성 스파이크 제거에는 효과가 있지만 연속 램프형 변화에는 충분하지 않다.

### 지오이드 보정

MSL과 타원체고도의 일정한 오프셋은 바꿀 수 있지만, 시간에 따라 위아래로
4~5 m 움직이는 현상은 고칠 수 없다.

### EKF GPS 노이즈만 증가

EKF가 GPS를 덜 믿게 만들 수는 있지만 GPS 원시 MSL 자체는 변하지 않는다.
현재는 GPS 고도 융합이 이미 제외되어 있다.

### GPS 원시값 덮어쓰기

가공한 값을 정상 GPS 원시값처럼 `GPS_RAW_INT`에 다시 넣으면 문제를 숨기고
안전 판정을 오도할 수 있으므로 수행하지 않는다.

---

## 11. 최종 판단 기준

소프트웨어 설정 하나가 개선 효과가 있다고 판단하려면 다음 조건이 필요하다.

1. 같은 위치와 같은 시험 조건에서 MSL 60초 범위가 뚜렷하게 감소한다.
2. 30초 최대 상승·하락도 함께 감소한다.
3. 위성 수와 VDOP가 악화되지 않는다.
4. Fix 상실이나 GPS 데이터 누락이 없다.
5. 다른 시각의 반복시험에서도 개선이 재현된다.
6. 운용상 필요한 비행 설정과 충돌하지 않는다.

모든 수신기 설정 시험 후에도 정지 상태에서 수 m 왕복이 반복되면 다음과
같이 결론낸다.

> 현재 M10 GPS 원시 MSL 고도는 소프트웨어 설정만으로 비행 제어에 신뢰할
> 수 있는 수준까지 개선되지 않았다.

이 경우 원시 GPS MSL은 기록과 상태감시에만 사용하고, 수직 제어는 현재처럼
PX4 Local Z, 기압계 및 TF-Luna를 사용한다.

---

## 12. 참고사항 — Raspberry Pi에서 별도로 만들 수 있는 보정

> 이 절은 내일의 M10 수신기 설정 현장시험 본문이 아니다. 현장 설정시험이
> 끝난 뒤 별도 개발 작업으로 검토한다. 사용자의 별도 지시 없이는 구현하거나
> PX4에 전송하지 않는다.

라즈베리파이에서 GPS 원시 MSL을 직접 고치는 것은 불가능하지만, 다른 센서와
비교하여 GPS 수직 상태를 판정하고 별도의 파생 고도를 만들 수 있다.

### 입력

```text
GPS raw MSL
GPS v_acc / VDOP / 위성 수 / Fix
PX4 Local Z와 수직속도
기압계 기반 고도
TF-Luna 지면 거리
PX4 landed state
```

### GPS 수직 상태 감시

예시 출력:

```text
/gps_vertical_health/valid
/gps_vertical_health/reason
/gps_vertical_health/range_60s
/gps_vertical_health/rise_30s
/gps_vertical_health/fall_30s
/gps_vertical_health/slope_10s
```

정지 판정 예시:

```text
ON_GROUND
armed=false
|Local Vz|가 작음
TF-Luna 변화가 작음
```

정지 상태인데 GPS MSL만 크게 움직이면 `valid=false`로 판정하고 Mission
시작을 차단한다.

### 파생 고도

필요하면 다음 원칙으로 별도의 고도를 만들 수 있다.

```text
기본 수직 변화: PX4 Local Z / 기압계
저고도 AGL: TF-Luna
GPS MSL: 정상 판정 구간에서만 참고
GPS 불량 시: 마지막 정상 기준 유지 또는 GPS 사용 중단
```

이 값은 반드시 `derived_altitude` 또는 `trusted_altitude`처럼 별도 이름을
사용한다. `GPS raw MSL`이라고 부르거나 원래 GPS 메시지를 덮어쓰지 않는다.

단순 저역통과필터만으로는 현재의 30~60초 연속 왕복을 해결하지 못하므로,
센서 간 일치 검사와 GPS 유효성 상태머신이 필요하다.

---

## 13. 후반부 거리제어 시험 전 Arm 점검

GPS 원시고도 설정시험과 거리제어 비행시험은 같은 단계로 취급하지 않는다.
거리제어를 추가로 시험하려면 모든 GPS 시험값을 복구한 뒤 Pixhawk를 다시
부팅하고 별도의 비행 전 점검을 수행한다.

### 2026-08-03 Arm 거부 정리

첫 번째 `/mission/start`는 다음 health mask 때문에 Mission Manager
PRECHECK에서 차단되어 Arm 명령이 전송되지 않았다.

```text
0x14000
- 0x4000: XY position control health 비정상
- 0x10000: RC receiver health 비정상
```

두 번째 시도에서는 Mission Manager의 `ignored_unhealthy_sensor_mask=81920`로
위 두 bit를 예외 처리했지만 PX4 내부 상태에는 `0x14000`이 그대로 남았다.
PX4가 Arm 요청을 계속 거부하여 약 4.12초 후 다음 결과로 종료됐다.

```text
ABORTED: arm exceeded retry limit
```

동시점에는 `const_pos_mode=true`, `system_status=0 (UNKNOWN)`, XY position
control health 이상이 확인됐다. 가장 유력한 원인은 PX4 내부 위치·추정기
관련 Arm 검사지만, 당시 PX4의 구체적인 거부 문구가 rosbag에 저장되지 않아
단일 원인으로 확정하지 않는다. Mission Manager의 mask 예외는 PX4 자체
안전검사를 해제하지 않는다.

### 다음 거리제어 시험의 필수 조건

1. `ignored_unhealthy_sensor_mask=0`으로 복구하고 Mission Manager를 재시작한다.
2. QGC Fly 화면의 Flight Readiness를 눌러 Arming Check Report를 확인한다.
3. 차단 항목이 있으면 화면을 캡처하고 Arm을 시도하지 않는다.
4. `const_pos_mode=false`인지 확인한다.
5. `sensors_enabled & ~sensors_health`에서 최소한 `0x4000`이 없어야 한다.
6. QGC가 현재 비행 모드에서 `Ready to Fly`라고 표시할 때만 Mission 시작을
   검토한다.
7. 강제 Arm 및 PX4 안전검사 우회는 사용하지 않는다.
8. Arm을 다시 거부하면 MAVROS `CommandBool.Response`의 `success`와 `result`,
   QGC Arming Check Report 및 Vehicle Messages를 함께 기록한다.

위 조건을 충족하지 않으면 GPS 원시고도 시험 결과가 일부 좋아졌더라도
거리제어 비행은 수행하지 않는다.

---

## 14. 공식 참고자료

- Holybro M10 GPS:
  https://holybro.com/products/m10-gps
- PX4 GPS 파라미터:
  https://docs.px4.io/main/en/advanced_config/parameter_reference
- PX4 파라미터 확인 및 변경:
  https://docs.px4.io/main/en/advanced_config/parameters
- PX4 GNSS 설명:
  https://docs.px4.io/main/en/gps_compass/index
- PX4 Arm·Prearm 및 QGC Arming Check Report:
  https://docs.px4.io/main/en/advanced_config/prearm_arm_disarm
- u-blox M10 Integration Manual:
  https://content.u-blox.com/sites/default/files/documents/MAX-M10M_IntegrationManual_UBX-22038241.pdf
