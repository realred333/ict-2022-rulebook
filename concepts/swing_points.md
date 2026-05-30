# Swing Points

대응 Rulebook: [swing_points_rulebook.md](../rulebook/swing_points_rulebook.md)

## 개념 정의

### 일반적인 정의

Swing point는 일정 구간 안에서 인접 가격보다 높거나 낮은 국지 극값이다. 일반적인 pivot 알고리즘은 좌우 비교 봉 수를 임의로 정할 수 있다.

### ICT 정의

ICT 2022 Mentorship에서 반복적으로 사용하는 최소 swing은 3캔들 구조이다.

- Swing High: 중심 봉의 `high`가 바로 왼쪽 봉과 바로 오른쪽 봉의 `high`보다 높다.
- Swing Low: 중심 봉의 `low`가 바로 왼쪽 봉과 바로 오른쪽 봉의 `low`보다 낮다.
- 중심 봉의 종가 방향은 조건이 아니다.
- 오른쪽 봉이 닫히기 전에는 swing을 확정할 수 없다.
- 동일 가격은 기본 swing 판정에서 제외하고 `relative equal highs/lows` 후보로 별도 전달한다.

Episode 5, 10, 16, 40에서 3캔들 구조가 반복된다. Episode 16과 40은 이 구조를 MT4 Williams fractal과 혼동하지 않도록 구분한다.

### 차트에서 의미

Swing point는 다음 계산의 원천 데이터이다.

- swing high 위 가격: BSL 후보
- swing low 아래 가격: SSL 후보
- 이전 swing 관통: liquidity sweep 또는 breakout 후보
- 이전 swing 종가 돌파: MSS 후보
- swing low와 swing high 쌍: dealing range 후보

### 시장 심리

ICT 체계에서는 swing high 위와 swing low 아래에 stop 주문이 존재할 가능성을 가정한다. OHLC만으로 실제 주문을 관측할 수는 없다. 따라서 코드는 `주문 존재`를 확정하지 않고 `liquidity reference candidate`를 생성한다.

## ICT 전체 체계에서의 위치

### 선행 개념

- 필수 선행 개념 없음
- 필수 입력: 닫힌 OHLC 봉과 timeframe

### 후행 개념

- [Liquidity](liquidity.md)
- [BSL / SSL](bsl_ssl.md)
- [Liquidity Sweep](liquidity_sweep.md)
- [Dealing Range](dealing_range.md)
- [Market Structure Shift](market_structure_shift.md)
- [SMT Divergence](smt_divergence.md)

### 왜 중요한가

Swing point는 ICT_LEARNING_MAP의 첫 단계이다. liquidity, MSS, dealing range, SMT는 모두 swing reference 없이 계산할 수 없다.

## 실제 차트에서 발견하는 방법

### 찾는 순서

1. 같은 timeframe의 OHLC를 시간순으로 정렬한다.
2. 연속된 닫힌 봉 3개를 선택한다.
3. 가운데 봉 `i`와 왼쪽 봉 `i-1`, 오른쪽 봉 `i+1`을 비교한다.
4. `high[i] > high[i-1]`이고 `high[i] > high[i+1]`이면 swing high를 확정한다.
5. `low[i] < low[i-1]`이고 `low[i] < low[i+1]`이면 swing low를 확정한다.
6. 확정 시각은 중심 봉 시각이 아니라 오른쪽 봉의 종가 확정 시각으로 기록한다.

### 유효 조건

- 3개 봉이 같은 symbol과 timeframe에 속한다.
- 세 봉이 모두 닫혔다.
- 비교에 사용하는 OHLC가 결측값이 아니다.
- high 또는 low 비교가 strict inequality를 만족한다.

### 무효 조건

- 오른쪽 봉이 아직 닫히지 않았다.
- 중심 봉과 인접 봉의 비교 가격이 같다.
- timeframe interval이 누락되어 연속된 3개 봉이 아니다.
- 미래 봉을 미리 읽어 중심 봉 시각에 신호를 발생시켰다.

### 대표 예시

```text
high: 100, 105, 102
              ^
          swing high

low:  100,  95,  98
              ^
          swing low
```

첫 번째 예시는 세 번째 봉이 닫힌 시점에 두 번째 봉을 swing high로 확정한다. 두 번째 예시는 세 번째 봉이 닫힌 시점에 두 번째 봉을 swing low로 확정한다.

### 보조 설명: 고급 swing 계층

Episode 11, 12, 17, 19, 22에는 short-term, intermediate-term, long-term swing 계층이 집중적으로 등장한다. 기본 탐지와 분리해 고급 구현으로 둔다.

- Intermediate-Term High: 확정 short-term high 중 양옆의 인접 short-term high보다 높은 high
- Intermediate-Term Low: 확정 short-term low 중 양옆의 인접 short-term low보다 낮은 low
- Long-Term High: 확정 intermediate-term high 중 양옆의 인접 intermediate-term high보다 높은 high
- Long-Term Low: 확정 intermediate-term low 중 양옆의 인접 intermediate-term low보다 낮은 low

Episode 12는 imbalance rebalance 문맥도 계층 판단에 사용한다. 이 부분은 순수 OHLC 계층 탐지와 분리해 `manual_review` 또는 추가 confluence metadata로 기록한다.

## 실전 활용

### 진입

Swing point 자체는 진입 신호가 아니다. 최소 진입 파이프라인은 다음과 같다.

```text
confirmed swing
-> liquidity reference
-> sweep 또는 breakout 분류
-> displacement
-> MSS
-> FVG 또는 OB retracement
```

### 손절

Swing point는 손절 후보 가격을 제공한다. 실제 손절 규칙은 진입 모델에서 정한다.

- long 후보: 선택한 구조 기준 swing low 아래에 stop 후보를 둔다.
- short 후보: 선택한 구조 기준 swing high 위에 stop 후보를 둔다.
- stop offset은 `max(1 * tick_size, configured_buffer_ticks * tick_size)`처럼 수치로 정의한다.

### 목표

- long 후보의 목표: active swing high 기반 BSL
- short 후보의 목표: active swing low 기반 SSL
- 이미 관통된 swing은 active liquidity 목표에서 제외한다.

### 시간대

기본 swing 탐지는 시간대와 무관하다. Episode 5와 38의 08:30, 13:30, lunch hour 설명은 swing 정의가 아니라 session filter의 보조 설명이다. session 조건은 [Killzone](killzone.md)에서 적용한다.

## 흔한 오해

### 초보자가 잘못 해석하는 부분

- 화면에서 눈에 띄는 고점과 저점을 수동으로만 선택한다.
- 오른쪽 확인 봉이 닫히기 전에 swing으로 표시한다.
- 캔들의 상승 종가 또는 하락 종가를 swing 조건에 넣는다.
- 모든 swing을 즉시 진입 신호로 사용한다.
- 좌우 2봉 pivot을 ICT 최소 정의로 고정한다.
- 나중에 가격이 swing을 관통하면 과거 swing 자체가 사라졌다고 처리한다.

### ICT가 실제로 의미하는 것

- 기본 swing은 재현 가능한 3캔들 구조이다.
- swing은 liquidity, 구조 변화, range 계산의 reference이다.
- 나중에 관통된 confirmed swing은 삭제하지 않는다. downstream liquidity 상태를 `taken`으로 변경한다.
- 더 넓은 문맥의 pivot 선택은 intermediate-term과 long-term 계층으로 확장한다.

## 다른 개념과의 연결

### 관련 개념

| 연결 개념 | swing point의 역할 |
| --- | --- |
| Liquidity | high 위, low 아래의 reference 생성 |
| BSL / SSL | swing high는 BSL, swing low는 SSL source 후보 |
| Liquidity Sweep | active swing level 관통과 reclaim 판정 |
| MSS | 반대 방향 short-term swing 종가 돌파 판정 |
| Dealing Range | swing low-high 또는 high-low leg 경계 |
| SMT Divergence | 상관 심볼의 swing 갱신 불일치 비교 |

### 의존 관계

```text
OHLC
-> confirmed short-term swing
  -> liquidity reference
  -> dealing range candidate
  -> MSS reference
  -> SMT comparison point

confirmed short-term swing
-> intermediate-term swing
  -> long-term swing
```

## 코드 구현 시 문제점

### 사람이 보는 기준

사람은 chart context를 보고 사용할 swing을 선택한다. 같은 차트에 short-term swing이 여러 개 있어도 HTF 방향, session, liquidity target, FVG 위치에 따라 하나를 우선할 수 있다.

### 코드로 구현 가능한 부분

- 3캔들 short-term swing 탐지
- 확정 지연 1봉 처리
- strict inequality 처리
- timeframe별 swing 레지스트리
- recursive intermediate-term, long-term swing 탐지
- 관통 시 downstream liquidity 상태 변경

### 코드로 구현 어려운 부분

- 복수 swing 중 narrative상 사용할 기준 swing 선택
- Episode 12의 imbalance rebalance 문맥을 포함한 계층 우선순위
- 영상에서 설명하는 HTF resistance와 특정 swing의 결합 우선순위

코드는 어려운 부분을 숨기지 않고 `manual_review_reason`으로 출력해야 한다.

## 핵심 문장

1. 기본 ICT swing은 닫힌 3캔들로 판별한다.
2. swing high는 중심 high가 좌우 high보다 높을 때 확정한다.
3. swing low는 중심 low가 좌우 low보다 낮을 때 확정한다.
4. 오른쪽 봉이 닫히기 전에는 swing을 확정하지 않는다.
5. 캔들의 종가 방향은 swing 판정 조건이 아니다.
6. 동일 가격은 기본 swing이 아니라 relative equal level 후보로 분리한다.
7. swing high는 BSL 후보를 만들고 swing low는 SSL 후보를 만든다.
8. swing 관통은 sweep 또는 breakout 분류의 입력이지 즉시 진입 신호가 아니다.
9. confirmed swing은 나중에 가격이 관통해도 과거 구조 기록으로 남긴다.
10. short-term swing을 먼저 구현하고 intermediate-term과 long-term 계층은 고급 구현으로 추가한다.

## 이 개념의 본질

Swing point는 ICT 운영체제에서 liquidity, 구조 변화, dealing range를 계산하기 위한 가장 작은 재현 가능 가격 reference이다.

## Transcript 근거 분류

### 에피소드 전반에 반복되는 핵심 근거

- Episode 2, 5, 10, 16, 19, 37, 38, 40: swing high/low를 liquidity reference로 사용
- Episode 5, 10, 16, 40: 좌우 인접 봉을 비교하는 3캔들 swing 구조
- Episode 12, 16, 24, 40: short-term swing break와 market structure shift 연결
- Episode 35, 36, 38: swing low-high leg와 dealing range 연결

### 특정 구간에 집중된 보조 근거

- Episode 11, 12, 17, 19, 22: intermediate-term과 long-term swing 계층
- Episode 5: 08:30 이전과 13:30 이후 swing 관찰 예시
- Episode 38: lunch hour swing low sweep 예시

