# DRV8876 솔레노이드 구동 회로 수정 — 담당자 인계 문서

- 작성일: 2026-08-22
- 근거: TI DRV8876 데이터시트 SLVSDS7B (2019-11 개정)
- 상태: **원인 후보 확정, 회로 수정 대기**

---

## 1. 한 줄 요약

**솔레노이드 밸브가 DRV8876을 통해서는 안 열리고, 12 V 직결로는 열린다.**
DRV8876의 전류 조절(current regulation) 설정이 원인이다. 해결책 3가지 중 하나를 적용하면 된다.

---

## 2. 확정된 사실

| 시험 | 결과 | 판정 |
|---|---|---|
| 펌프 단독 + 호스 | 물 나옴 | 펌프 정상 |
| 솔레노이드 **12 V 직결** | **열림, 물 나옴** | **밸브 정상** |
| DRV8876 경유 (PX4 명령) | 딸깍 소리만, 물 안 나옴 | **회로 문제** |
| 배터리 전류 측정 | 지속 전류 계단 없음 | 통전이 유지되지 않음 |

소프트웨어 경로(ROS → MAVROS → MAVLink 203 → PX4 camera_trigger → AUX5)는 정상이다.
PX4 파라미터도 문서값과 일치한다. **문제는 DRV8876과 그 주변 회로에 국한된다.**

---

## 3. 부품 값

```
밸브    스톰테크 HSV-FF-12V 1/4, 직동식 NC
        코일 저항 33 Ω (실측)
        12 V에서 자연 전류 = 12 ÷ 33 = 0.364 A
        소비전력 약 4.4 W
        사용압력 0.2 ~ 10 bar

펌프    SEAFLO SFDP1-015-040-23C
        12 V, 5.6 L/min, 최대 40 PSI = 2.76 bar

구동    DRV8876, SLEEP=5V, PMODE=GND, PH/IN2=GND
        전류 제한 약 0.36 A (기존 문서의 계산 추정치, 실측 아님)
        FAULT / CS 피드백 미배선
```

---

## 4. 가장 유력한 원인 — IMODE 사이클별 초핑 + 100% 듀티 충돌

**이것이 이번에 새로 찾은 핵심이다.**

DRV8876의 전류 초핑 방식은 `IMODE` 핀으로 정해진다.

### 데이터시트 Table 6 — IMODE Functions

| IMODE | R_IMODE | 초핑 방식 | 과전류 응답 | nFAULT |
|---|---|---|---|---|
| Level 1 | GND | **Fixed Off-Time** | 자동 재시도 | 과전류만 |
| Level 2 | 20 kΩ to GND | Cycle-By-Cycle | 자동 재시도 | 초핑+과전류 |
| Level 3 | 62 kΩ to GND | Cycle-By-Cycle | 출력 래치 정지 | 초핑+과전류 |
| Level 4 | Hi-Z (개방) | **Fixed Off-Time** | 출력 래치 정지 | 과전류만 |

### 왜 문제가 되는가

데이터시트 원문:

> **Cycle-by-cycle mode will not support 100% duty cycle current regulation** as a new control input edge is required to reset the outputs after the brake, low-side slow decay state has been entered.

즉 **Cycle-By-Cycle 모드(Level 2, 3)** 에서는 전류가 I_TRIP에 도달하면 브레이크 상태로 들어가고,
**EN/IN1에 새로운 엣지가 들어와야만** 출력이 복구된다.

그런데 우리 구동 방식은 PX4 camera_trigger가 AUX5를 **정적으로 HIGH 유지**하는 방식이다.
펄스 동안 엣지가 **한 번도 없다.**

```
전류 상승 → I_TRIP 도달 → 브레이크 진입 → 엣지 대기 → 엣지 없음 → 펄스 끝까지 출력 정지
```

**증상이 정확히 일치한다.**

- 초기 전류 버스트 → **딸깍 소리**
- 이후 출력 정지 → **지속 전류 없음** (배터리 전류 측정 결과와 일치)
- 밸브가 유지되지 않음 → **물 안 나옴**

`I_TRIP`이 0.36 A이고 코일 자연 전류가 0.364 A이므로, **매번 반드시 트립된다.**

### 확인 방법

**IMODE 핀(HTSSOP 7번)이 GND에 직결인지, 저항을 통하는지, 개방인지 확인한다.**

- GND 직결 또는 개방(Hi-Z) → Fixed Off-Time. 이 가설은 탈락
- 20 kΩ 또는 62 kΩ → **Cycle-By-Cycle. 이 가설 확정**

---

## 5. 해결 방법 — 3가지, 권장 순서

### A. IPROPI를 GND에 직결 — 가장 간단, 권장

데이터시트 원문:

> The internal current regulation can be **disabled by tying IPROPI to GND** and setting the VREF pin voltage greater than GND (if current feedback is not required).

전류 조절 기능을 **완전히 끈다.** 저항 계산이 필요 없다.

```
IPROPI 핀 (HTSSOP 6번 / VQFN 4번)  →  GND 직결
VREF 는 GND보다 높게 유지 (현재 분압 그대로 두면 됨)
```

**안전한가?**
- 코일은 어차피 12 V ÷ 33 Ω = **0.364 A만 흘린다.** 전류 조절은 상한일 뿐 밀어넣는 값이 아니다
- 과전류 보호(OCP)는 **별개 회로**로 5.5 A에서 여전히 동작한다
- DRV8876 정격은 3.5 A이므로 0.364 A는 10분의 1 수준

솔레노이드처럼 **저항성 고정 부하**에는 전류 조절 자체가 불필요하다.

### B. IMODE를 Fixed Off-Time으로 변경

`IMODE`를 **GND 직결(Level 1)** 또는 **개방(Level 4)** 으로 바꾼다.

Fixed Off-Time은 t_OFF(25 µs) 후 **자동으로 출력을 복구**하므로 100% 듀티를 지원한다.
전류 조절은 유지되지만 평균 전류가 I_TRIP 부근으로 제한되어, 밸브가 열릴 만큼의 힘이
나오지 않을 수 있다. **C와 함께 적용하는 것이 안전하다.**

### C. I_TRIP 상향 — 저항 교체

데이터시트 식 (3):

```
I_TRIP (A) × A_IPROPI (µA/A) = V_VREF (V) / R_IPROPI (Ω)
```

`A_IPROPI = 1000 µA/A` (데이터시트 전기적 특성)이므로 정리하면:

```
I_TRIP (A) = V_VREF (V) × 1000 ÷ R_IPROPI (Ω)
```

데이터시트 예시 검증: V_VREF = 2.5 V, R_IPROPI = 1500 Ω → 2.5 × 1000 ÷ 1500 = **1.67 A** (데이터시트와 일치)

#### 적용 절차

1. **R_IPROPI 실측** — IPROPI 핀과 GND 사이 저항
2. **V_VREF 실측** — VREF 핀과 GND 사이 전압 (전원 인가 상태, 밸브 발사 불필요)
3. 현재 I_TRIP 계산 → 문서의 0.36 A와 맞는지 검증
4. 목표 I_TRIP = **0.8 A** (밸브 정격 0.364 A의 약 2.2배 여유)
5. 필요한 V_VREF 계산:

```
V_VREF_필요 = 0.8 × R_IPROPI ÷ 1000
```

6. VREF 분압 저항을 그에 맞게 교체

**주의: V_VREF 권장 범위는 0 ~ 3.6 V다** (절대 최대 5.75 V). 계산값이 3.6 V를 넘으면
R_IPROPI 쪽을 낮추는 방향으로 조정해야 한다.

---

## 6. 핀 배치 (데이터시트 5절)

### HTSSOP-16 (PWP) — 세로 직사각형, 양옆 8핀씩

```
 1  EN/IN1   ┌──────┐  16  PMODE
 2  PH/IN2   │      │  15  GND
 3  nSLEEP   │      │  14  CPL
 4  nFAULT   │ 열판  │  13  CPH
 5  VREF     │      │  12  VCP
 6  IPROPI   │      │  11  VM
 7  IMODE    │      │  10  OUT2
 8  OUT1     └──────┘   9  PGND
```

### VQFN-16 (RGT) — 정사각형, 각 변 4핀

```
 1  nSLEEP      13  GND
 2  nFAULT      14  PMODE
 3  VREF        15  EN/IN1
 4  IPROPI      16  PH/IN2
 5  IMODE       12  CPL
 6  OUT1        11  CPH
 7  PGND        10  VCP
 8  OUT2         9  VM
```

1번 핀은 칩 위 **점 또는 홈**으로 표시. 거기서 반시계 방향으로 센다.
브레이크아웃 보드면 실크스크린 라벨을 우선 확인할 것.

### 이번 작업에 관련된 핀

```
VREF    HTSSOP 5번 / VQFN 3번    전류 제한 기준 전압
IPROPI  HTSSOP 6번 / VQFN 4번    출력 전류 비례 아날로그 출력
IMODE   HTSSOP 7번 / VQFN 5번    초핑 방식 결정 (4단계 입력)
EN/IN1  HTSSOP 1번 / VQFN 15번   AUX5가 연결된 제어 입력
```

---

## 7. 작업 순서 (권장)

```
1. IMODE 핀 결선 확인      → Cycle-By-Cycle 인지 판정 (4절)
2. R_IPROPI, V_VREF 실측   → 현재 I_TRIP 계산 (5-C절)
3. A안 적용: IPROPI → GND  → 전류 조절 비활성화
4. 밸브 발사 시험          → 물 나오는지
5. 성공 시 정량 측정으로 진행
```

**A안 하나면 대부분 해결된다.** 저항 교체 없이 점퍼 하나로 끝나므로 되돌리기도 쉽다.

---

## 8. 검증 방법

### 성공 판정

- 밸브 발사 명령 시 **물이 3초간 나오고 멈춘다**
- 코일 양단(OUT1–OUT2) 전압이 펄스 동안 **12 V 근처 유지**
- AUX5–GND 전압이 펄스 동안 **3.3 V 유지**

### 소프트웨어로는 판정 불가

`DRV8876 FAULT / CS` 피드백이 배선되지 않아 밸브 상태를 소프트웨어가 알 수 없다.
노드 응답 `PX4 accepted one 3000ms trigger`는 **MAVLink ACK일 뿐 물리 개폐 증거가 아니다.**

PM07 배터리 전류로도 판정 불가 — 노이즈 ±0.25 A인데 신호가 0.36 A라 묻힌다.

**폐루프 진단이 필요하면 `nFAULT` 또는 절연 전류 센서를 별도 입력으로 추가해야 한다.**
특히 IMODE Level 2/3에서는 nFAULT가 **초핑 발생 시에도 Low로 떨어지므로**, 이 선 하나만
파이 GPIO에 연결해도 "전류 조절이 트립되고 있다"를 소프트웨어로 감지할 수 있다.

---

## 9. 대안 — MOSFET 교체

DRV8876은 **H-브리지 모터 드라이버**다. 양방향 구동, 전류 조절, 초핑 모드가 붙어 있는데
솔레노이드는 단방향 ON/OFF만 필요하다. **기능 과잉이고, 그 과잉 기능이 이번 문제를 만들었다.**

```
로직레벨 N채널 MOSFET  +  플라이백 다이오드
```

- 전류 조절 없음 → 이번 문제가 구조적으로 발생하지 않음
- 부품값 저렴
- 회로 단순, 디버깅 쉬움
- 게이트를 AUX5에서 직접 구동 (3.3 V 로직레벨 MOSFET 선정 필요)
- **플라이백 다이오드 필수** — 솔레노이드는 유도성 부하라 차단 시 역기전력이 발생한다

어차피 회로에 손을 댈 거라면 이쪽이 근본적이다.

---

## 10. 참고

- 데이터시트: TI DRV8876, SLVSDS7B (2019-11) — https://www.ti.com/lit/ds/symlink/drv8876.pdf
  - 5절 Pin Configuration
  - 6.3 Recommended Operating Conditions (V_VREF 0~3.6 V)
  - 6.5 Electrical Characteristics (A_IPROPI = 1000 µA/A, I_OCP = 5.5 A)
  - 7.3.3.2 Current Regulation (Table 6 IMODE, 식 3)
- 기존 절차서: `~/da-daka_Ai/docs/solenoid_bench_test.md`
- 배선 정리: `~/Desktop/DRV8876_솔레노이드밸브_3초시험_코드정리_20260820.txt`
- 진단 경위: `~/Desktop/DA-DAKA_2026-08-22_분사계통_밸브미개방_진단기록.md`

### 회로 수정 후 반드시 할 것

- 벤치 노드를 `bench_test_approved:=false` 잠금 상태로 복귀
- 프로펠러 제거 상태에서만 시험
- 첫 시험은 1회만, 발열·냄새 확인
