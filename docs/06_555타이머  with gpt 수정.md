NE555 기반 12V-to-5V Buck Converter 설계 및 LTspice 검증

1. 프로젝트 개요

스마트 USB 충전기 프로젝트의 전력 변환부를 이해하기 위해, NE555를 PWM 발생기로 사용한 Buck Converter를 LTspice에서 설계하였다.

단순히 12V를 5V로 낮추는 회로를 그리는 데서 끝내지 않고 다음 항목을 직접 확인하는 것을 목표로 했다.

MOSFET의 스위칭에 따른 인덕터 전류 변화

프리휠 다이오드의 역할

출력 커패시터와 ESR이 출력 리플에 미치는 영향

부하 변화에 따른 출력전압 변화

고정 듀티 제어와 피드백 제어의 차이

실제 핸드폰 입력을 어떤 부하로 모델링해야 하는지

LTspice에서 발생하는 수치 수렴 문제와 해결 과정

현재 상태: 전력단 배선과 핸드폰 등가부하를 수정하고, PHONE 단자 전압을 감지하는 피드백 구조를 적용하였다. 최종 5V 유지 파형과 SPICE Error Log 검증은 진행 중이며, 검증이 끝나면 결과 표와 파형 이미지를 갱신할 예정이다.

2. 목표 사양

항목

목표값

입력전압

12V DC

목표 출력전압

5V DC

기본 부하전류

약 1A

부하 시험 범위

0.5A, 1A, 2A

스위칭 주파수

약 100kHz

PWM 발생부

NE555

시뮬레이션 도구

LTspice

이상적인 Buck Converter의 듀티비는 다음과 같이 예상할 수 있다.

D = VOUT / VIN
D = 5V / 12V
D = 0.417

따라서 이상적인 조건에서는 약 41.7%의 듀티비가 필요하다.

하지만 이 관계는 정상상태, 연속전도모드, 이상적인 소자라는 조건에서 성립하는 근사식이다. 실제 시뮬레이션에서는 MOSFET 손실, 다이오드 순방향 전압, 인덕터 DCR, 커패시터 ESR, 부하 조건에 의해 결과가 달라질 수 있다.

3. 전체 시스템 구조

최종적으로 구성한 시스템은 다음과 같다.

12V 입력
  |
  v
NE555 타이밍 회로
  |
  v
PWM 및 피드백 제어
  |
  v
High-Side NMOS Gate Driver
  |
  v
MOSFET + Schottky Diode + Inductor + Output Capacitor
  |
  v
OUT -> Cable Resistance -> PHONE Load

전력단의 기본 구조는 다음과 같다.

VIN -> M1 -> SW -> L1 -> OUT -> RCABLE -> PHONE
              |          |                  |
              D1         C1                 CPHONE
              |          |                  |
             GND        GND                GND
                                             |
                                          RPHONE
                                             |
                                            GND

4. 주요 소자와 설정값

소자

설정값

역할

V1

12V

입력전원

U1

NE555

타이밍 파형 및 PWM 기준 생성

M1

NMOS_BUCK

High-Side 스위칭 소자

D1

MBRS340

MOSFET OFF 시 인덕터 전류 경로 제공

L1

220uH, DCR 60mohm

에너지 저장 및 전류 평활

C1

220uF, ESR 40mohm

출력전압 평활

RCABLE

150mohm

USB 케이블 전압강하 모델

CPHONE

22uF, ESR 50mohm

핸드폰 입력 커패시턴스 모델

RPHONE

5ohm

5V, 1A 핸드폰 등가부하

부하저항 계산

5V에서 1A를 소비하는 부하는 다음과 같이 계산할 수 있다.

R = V / I
R = 5V / 1A
R = 5ohm

부하전류별 등가저항은 다음과 같다.

목표 부하전류

RPHONE

0.5A

10ohm

1A

5ohm

2A

2.5ohm

5. 이론적인 인덕터 전류 리플 검토

스위칭 주파수를 약 100kHz, 듀티비를 0.417, 인덕터를 220uH라고 가정하면 인덕터 전류 리플은 다음과 같이 근사할 수 있다.

Delta_IL = (VIN - VOUT) * D / (L * fs)

Delta_IL = (12V - 5V) * 0.417 / (220uH * 100kHz)
Delta_IL = 약 0.133A

따라서 1A 부하에서 인덕터 전류는 이상적인 조건에서 약 0.133A의 첨두간 리플을 가질 것으로 예상된다.

출력 커패시터의 용량 성분에 의한 전압 리플은 다음과 같이 근사할 수 있다.

Delta_VC = Delta_IL / (8 * fs * C)

Delta_VC = 0.133A / (8 * 100kHz * 220uF)
Delta_VC = 약 0.00076V

커패시터 ESR에 의한 리플은 다음과 같이 예상할 수 있다.

Delta_VESR = Delta_IL * ESR
Delta_VESR = 0.133A * 0.04ohm
Delta_VESR = 약 0.0053V

따라서 실제 출력 리플은 이상적인 커패시턴스 계산값보다 ESR의 영향을 더 크게 받을 수 있다. 이 계산은 소자 선정 단계의 이론적 예상값이며, 최종값은 LTspice 파형으로 확인해야 한다.

6. 시행착오 1: R1에서 12V가 측정된 문제

문제 상황

처음에는 R1을 핸드폰 부하라고 가정했지만, R1 위쪽에서 약 12V가 측정되었다. Buck Converter를 설계했음에도 출력이 입력전압과 거의 같았기 때문에 듀티비와 소자값을 먼저 의심했다.

확인한 원인

회로를 다시 확인한 결과 MOSFET이 스위치 역할을 해야 하는 구간에 배선이 직접 연결되어 있었다.

잘못된 연결

VIN ---------------- SW -> L1 -> OUT
        MOSFET 우회

LTspice에서는 MOSFET 그림 위에 선을 그렸다고 해서 선이 자동으로 끊어지지 않는다. 드레인과 소스 사이를 배선이 직접 연결하면 MOSFET의 ON/OFF 상태와 관계없이 VIN과 SW가 같은 노드가 된다.

수정

정상 연결

VIN -> M1 Drain
M1 Source -> SW -> L1 -> OUT

VIN과 SW가 서로 다른 전기적 노드인지 확인하고, MOSFET을 통과하지 않는 우회 배선을 제거하였다.

배운 점

회로도에서 소자의 모양보다 실제 노드 연결이 더 중요하다.

MOSFET을 배치했다고 해서 자동으로 스위칭 경로가 형성되는 것은 아니다.

이상한 출력전압이 측정되면 소자값을 변경하기 전에 전원에서 출력까지의 직류 경로를 먼저 추적해야 한다.

7. 시행착오 2: 1kohm은 핸드폰 부하가 아니었다

초기 설정

초기 R1 값은 1kohm이었다.

I = V / R
I = 5V / 1000ohm
I = 5mA

5mA는 스마트폰 충전 전류와 비교하면 매우 작은 값이다. 따라서 1kohm 부하는 사실상 무부하에 가까운 조건이었다.

왜 문제가 되는가

부하가 매우 작으면 출력 커패시터에 공급된 에너지가 충분히 소비되지 않는다. 특히 고정 듀티의 개방루프 Buck Converter에서는 출력전압이 목표값보다 높게 올라갈 수 있다.

또한 VOUT = D * VIN 관계는 모든 조건에서 자동으로 5V를 보장하는 식이 아니다. 부하가 매우 작아 불연속전도모드로 들어가거나 피드백이 없다면 출력전압은 크게 달라질 수 있다.

수정

1A 충전을 기준으로 RPHONE을 5ohm으로 설정하였다. 이후 10ohm, 5ohm, 2.5ohm으로 변경하면서 0.5A, 1A, 2A 조건을 각각 시험할 수 있도록 구성하였다.

배운 점

부하저항은 단순히 회로를 닫기 위한 부품이 아니라 시스템의 동작점을 결정한다.

전원회로 시뮬레이션에서는 목표 출력전류에 맞는 부하 모델이 필요하다.

무부하, 경부하, 정격부하의 동작이 서로 다를 수 있다.

8. 시행착오 3: 핸드폰을 단순 저항으로만 볼 수 있는가

실제 핸드폰은 USB 5V 입력에 단순 저항이 직접 연결된 구조가 아니다. 내부 충전 IC가 배터리 상태와 온도, 충전 단계에 따라 입력전류를 조절한다.

초기에는 다음과 같은 PWL 전류 싱크를 사용했다.

IPHONE OUT 0 PWL(0 0.1 5m 0.1 5.01m 0.5 10m 0.5 10.01m 1 15m 1 15.01m 2)

이 방식은 부하전류를 0.1A, 0.5A, 1A, 2A로 변화시킬 수 있어 과도응답 시험에 유리하다.

그러나 SPICE 명령문으로만 작성했기 때문에 회로도에서는 핸드폰 부하가 보이지 않는 문제가 있었다. 회로를 처음 보는 사람이 C1 뒤쪽이 무부하라고 오해할 수 있었다.

최종 가시화 구조

OUT -> RCABLE -> PHONE
                  |-- CPHONE -> GND
                  |-- RPHONE -> GND

RCABLE: 케이블 저항과 커넥터 접촉저항을 단순화

CPHONE: 핸드폰 입력단의 커패시턴스를 단순화

RPHONE: 특정 충전전류를 나타내는 등가부하

배운 점

시뮬레이션이 전기적으로 동작하는 것과 회로도가 이해하기 쉬운 것은 별개의 문제다.

포트폴리오 회로도는 결과뿐 아니라 모델의 의미가 시각적으로 드러나야 한다.

실제 시스템을 단순화할 때는 무엇을 생략했는지 명확히 기록해야 한다.

9. 시행착오 4: High-Side NMOS의 게이트 구동

High-Side N-Channel MOSFET을 완전히 켜려면 게이트 전압을 GND 기준이 아니라 소스 기준으로 확인해야 한다.

VGS = VGATE - VSOURCE

MOSFET이 ON되면서 SW 전압이 상승하면 소스 전압도 상승한다. 따라서 게이트를 단순히 12V로 인가하면 SW가 12V에 가까워졌을 때 VGS가 부족해질 수 있다.

시뮬레이션에서는 다음과 같은 이상적인 High-Side Gate Driver를 사용하였다.

BDRV GDRV SW V=V(REG)

이 소스는 GDRV 전압을 SW 기준으로 생성한다. 따라서 확인해야 할 파형은 V(GATE) 단독이 아니라 다음과 같다.

V(GATE,SW)

한계

현재 Gate Driver는 동작 원리를 확인하기 위한 이상적인 행동 모델이다. 실제 회로에서는 Bootstrap Gate Driver 또는 High-Side Driver IC가 필요하다.

10. 시행착오 5: 약 9ms에서 시뮬레이션이 멈춘 문제

관찰한 파형

초기 출력이 약 6V까지 상승

일정 시간 동안 높은 전압을 유지

부하가 인가된 후 출력전압이 감소

약 9ms 부근에서 파형 데이터가 끝남

초기 피드백 방식

처음에는 다음과 같이 출력전압이 5.02V보다 낮으면 PWM을 통과시키고, 높으면 완전히 차단하는 방식을 사용했다.

BREG REG 0 V=if(V(OUT)<5.02,V(PWM),0)

이 방식은 구현이 간단하지만 5.02V 경계에서 출력이 조금만 변해도 제어가 즉시 ON/OFF를 반복한다.

VOUT < 5.02V -> PWM ON
VOUT > 5.02V -> PWM OFF

이상적인 비교기, 이상적인 Gate Driver, 이상적인 스위칭 모델이 동시에 사용되면 LTspice가 매우 작은 시간 간격을 요구할 수 있다. 그 결과 Timestep too small 또는 수렴 문제로 시뮬레이션이 특정 시간에서 멈출 수 있다.

시뮬레이션 설정 문제

초기 Transient 설정은 다음과 같았다.

.tran 0 25m 0 50n startup
.options plotwinsize=0

25ms를 최대 50ns 간격으로 계산하면 최소 약 500,000개의 시간점이 필요하다. 또한 plotwinsize=0은 파형 압축을 끄기 때문에 계산 및 저장 부담을 증가시킨다.

수정

.tran 0 25m 0 200n startup
.options method=gear reltol=0.003

그리고 5.02V에서 갑자기 ON/OFF하는 비교 방식을 제거하고, NE555 타이밍 커패시터 파형과 연속적인 제어전압을 비교하는 구조로 변경하였다.

배운 점

시뮬레이션 정지는 실제 회로의 물리적 정지와 다를 수 있다.

이상적인 스위치와 불연속 행동 모델은 수치 수렴 문제를 만들 수 있다.

파형이 특정 시간에서 끝나면 회로만 볼 것이 아니라 SPICE Error Log와 최대 시간 간격도 확인해야 한다.

11. 피드백 제어 구조 개선

개방루프 고정 듀티 방식은 입력전압, 부하, 손실이 변하면 출력전압을 5V로 유지할 수 없다. 따라서 PHONE 단자 전압을 다시 PWM 제어부로 되돌리는 폐루프 피드백을 구성하였다.

제어 목표

VPHONE < 5V -> CTRL 증가 -> PWM Duty 증가
VPHONE > 5V -> CTRL 감소 -> PWM Duty 감소

현재 LTspice 행동 모델은 다음과 같다.

BCTRL CTRL 0 V=5.67+2.5*tanh((5-V(PHONE))/1)

PWM 비교부도 불연속적인 if() 전환 대신 완만한 전환을 사용하였다.

BREG REG 0 V=6*(1+tanh((V(CTRL)-V(TIMING))/50m))

피드백 위치를 OUT이 아닌 PHONE으로 선택한 이유

OUT과 PHONE 사이에는 RCABLE이 존재한다.

VCABLE_DROP = IPHONE * RCABLE

예를 들어 1A가 흐르고 RCABLE이 0.15ohm이면 다음과 같은 전압강하가 발생한다.

VCABLE_DROP = 1A * 0.15ohm
VCABLE_DROP = 0.15V

OUT을 정확히 5V로 제어하면 PHONE에서는 약 4.85V가 될 수 있다. 따라서 이번 모델에서는 실제 충전 단자인 PHONE 전압을 감지하도록 구성하였다.

현재 피드백 모델의 의미

현재 BCTRL과 BREG는 피드백 원리를 검증하기 위한 LTspice 행동 모델이다. 실제 하드웨어에 그대로 사용할 수 있는 부품 회로는 아니다.

실제 제작 단계에서는 다음 구조가 필요하다.

PHONE Voltage Divider
        |
        v
Reference + Error Amplifier
        |
        v
PWM Comparator
        |
        v
High-Side Gate Driver

12. 프리휠 다이오드 D1의 역할

MOSFET이 ON일 때 인덕터 전류는 다음 경로로 증가한다.

VIN -> M1 -> L1 -> Load -> GND

MOSFET이 OFF되어도 인덕터 전류는 순간적으로 0이 될 수 없다. 인덕터는 기존 전류 방향을 유지하기 위해 자신의 단자전압 극성을 바꾸며, D1이 다음과 같은 전류 경로를 제공한다.

GND -> D1 -> SW -> L1 -> Load -> GND

D1이 없으면 SW 노드에 큰 음의 전압 스파이크가 발생하여 MOSFET에 과도한 전압 스트레스를 줄 수 있다.

MBRS340은 Schottky Diode이므로 일반 PN 다이오드보다 순방향 전압이 낮고 역회복 특성이 빠르다. Buck Converter의 프리휠 다이오드로 사용하기 적절한 이유다.

13. 반드시 확인해야 할 LTspice 파형

파형

확인 목적

V(PHONE)

실제 핸드폰 단자 전압이 5V 부근인지 확인

V(OUT)

케이블 저항 이전의 컨버터 출력 확인

V(SW)

MOSFET 스위칭 노드가 정상적으로 전환되는지 확인

V(GATE,SW)

MOSFET의 실제 VGS 확인

V(TIMING)

NE555 타이밍 커패시터 파형 확인

V(CTRL)

출력 오차에 따른 제어전압 변화 확인

I(L1)

인덕터 평균전류와 전류 리플 확인

I(RPHONE)

핸드폰 등가부하 전류 확인

정상 동작 판단 기준

Startup 후 VPHONE이 5V 부근으로 수렴하는가

부하 변경 후 VPHONE이 다시 목표전압으로 복귀하는가

VGS가 MOSFET을 충분히 켤 수 있는 수준인가

인덕터 전류가 비정상적으로 발산하거나 음수로 크게 내려가지 않는가

SW 노드에 과도한 음의 스파이크가 발생하지 않는가

SPICE Error Log에 수렴 오류가 없는가

14. 검증 계획

14.1 1A 기본 부하

RPHONE = 5ohm
예상 부하전류 = 약 1A

확인 항목:

VPHONE 평균값

출력 리플

Startup Overshoot

정상상태 인덕터 평균전류

MOSFET VGS

14.2 0.5A 부하

RPHONE = 10ohm

경부하에서 출력전압이 상승하지 않는지 확인한다.

14.3 2A 부하

RPHONE = 2.5ohm

중부하에서 출력전압 강하, 인덕터 전류, MOSFET 및 다이오드 전류를 확인한다.

14.4 결과 기록 표

최종 시뮬레이션 후 아래 표를 실제 측정값으로 채울 예정이다.

RPHONE

목표전류

VOUT 평균

VPHONE 평균

VPHONE 리플

최대 인덕터 전류

결과

10ohm

0.5A

측정 예정

측정 예정

측정 예정

측정 예정

검증 예정

5ohm

1A

측정 예정

측정 예정

측정 예정

측정 예정

검증 예정

2.5ohm

2A

측정 예정

측정 예정

측정 예정

측정 예정

검증 예정

15. 이미지 정리 예시

GitHub에는 다음과 같은 순서로 이미지를 배치하면 시행착오가 잘 드러난다.

images/
|-- 01_initial_output_12V.png
|-- 02_vin_sw_short_path.png
|-- 03_corrected_mosfet_path.png
|-- 04_output_stopped_at_9ms.png
|-- 05_feedback_block.png
|-- 06_visible_phone_load.png
|-- 07_final_vphone_waveform.png
|-- 08_inductor_current_and_vgs.png

README에서 이미지는 다음과 같이 삽입할 수 있다.

![초기 12V 출력 문제](images/01_initial_output_12V.png)

![9ms 수렴 문제](images/04_output_stopped_at_9ms.png)

![최종 PHONE 단자 전압](images/07_final_vphone_waveform.png)

16. 프로젝트 파일 구조 예시

NE555-Buck-Converter/
|-- README.md
|-- circuits/
|   |-- 01_initial_open_loop.asc
|   |-- 02_mosfet_wiring_fix.asc
|   |-- 03_feedback_v2.asc
|   `-- 04_visible_phone_load_v3.asc
|-- images/
|   |-- 01_initial_output_12V.png
|   |-- 04_output_stopped_at_9ms.png
|   |-- 06_visible_phone_load.png
|   `-- 07_final_vphone_waveform.png
`-- docs/
    `-- calculation_notes.md

파일 이름에 설계 단계가 드러나도록 번호를 붙이면 결과만 제시하는 것보다 문제 해결 과정이 잘 보인다.

17. 설계의 한계

현재 모델은 Buck Converter의 동작과 피드백 개념을 학습하기 위한 시뮬레이션 모델이며 다음과 같은 한계가 있다.

MOSFET 모델이 실제 제조사 소자의 상세 특성을 모두 반영하지 않는다.

High-Side Gate Driver가 이상적인 행동 모델이다.

BCTRL과 BREG는 실제 Error Amplifier와 PWM Comparator를 단순화한 모델이다.

핸드폰 부하는 충전 프로토콜과 배터리 상태를 포함하지 않은 등가모델이다.

USB D+/D- 또는 USB Type-C CC 통신과 PD 협상은 포함하지 않았다.

PCB 기생 인덕턴스와 배선 저항에 의한 Switching Spike는 상세히 반영하지 않았다.

MOSFET, 다이오드, 인덕터의 온도 상승과 열 특성은 아직 검토하지 않았다.

이 한계를 명확히 기록함으로써 시뮬레이션 결과를 실제 제품 성능으로 과도하게 해석하지 않도록 하였다.

18. 향후 개선 계획

SPICE Error Log를 확인하여 25ms까지 정상적으로 계산되는지 검증

RPHONE을 10ohm, 5ohm, 2.5ohm으로 변경하며 부하별 결과 기록

Startup Overshoot와 정상상태 출력 리플 측정

MOSFET 제조사 모델을 적용하여 도통손실과 스위칭손실 비교

다이오드의 순방향 손실 계산

인덕터 DCR 손실과 포화전류 조건 검토

행동 모델 피드백을 실제 Error Amplifier와 Comparator 회로로 교체

Bootstrap High-Side Gate Driver 모델 추가

스위칭 손실과 열 발생을 고려한 효율 분석

이후 KiCad에서 PCB 전력 루프와 접지 구조 검토

19. 프로젝트를 통해 배운 점

이번 과정에서 가장 크게 배운 점은 전력회로에서 출력전압 하나만 보는 것으로는 문제를 정확히 판단할 수 없다는 것이다.

12V가 그대로 출력된 원인은 듀티비가 아니라 MOSFET을 우회한 배선이었다.

1kohm 부하는 핸드폰 부하가 아니라 거의 무부하 조건이었다.

고정 듀티만으로는 입력과 부하 변화에 대해 5V를 유지할 수 없었다.

High-Side NMOS는 GND 기준 게이트 전압이 아니라 VGS를 확인해야 했다.

피드백은 단순히 임계값에서 ON/OFF하는 것보다 안정성과 수치 수렴을 고려해야 했다.

실제 핸드폰 부하는 저항, 전류 싱크, 케이블 저항, 입력 커패시터 등 목적에 맞는 등가모델이 필요했다.

LTspice에서 특정 시간에 파형이 끝나는 현상은 실제 회로 동작이 아니라 수치 계산 실패일 수 있었다.

회로가 동작하는 것뿐만 아니라 다른 사람이 회로도를 보고 모델의 의미를 이해할 수 있도록 표현하는 것도 중요했다.

단순한 Buck Converter 예제를 따라 그리는 것에서 출발했지만, 배선 검증, 부하 모델링, 피드백 제어, 수치 수렴, 측정 기준까지 단계적으로 확장할 수 있었다.

20. 포트폴리오 요약

NE555 기반 약 100kHz PWM 회로와 12V-to-5V Buck Converter 전력단 설계

MOSFET 우회 배선으로 인한 12V 출력 문제를 노드 추적으로 분석 및 수정

1kohm 단순 부하의 한계를 확인하고 0.5A~2A 충전 조건에 맞는 부하 모델 설계

High-Side NMOS의 VGS 기준 구동 조건과 이상적 Gate Driver 모델 적용

불연속 피드백으로 발생한 LTspice 수렴 문제를 분석하고 연속 PWM 비교 구조로 개선

케이블 저항과 핸드폰 입력 커패시터를 포함한 가시적인 PHONE 등가부하 구성

최종 출력뿐 아니라 VSW, VGS, IL, VCTRL, VPHONE을 함께 검증하는 측정 계획 수립

<img width="1895" height="1025" alt="최종본 with gpt 수정본" src="https://github.com/user-attachments/assets/a98b582e-4ba6-4dbd-95c4-cf7b90836f13" />
