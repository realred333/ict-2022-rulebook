# Breaker Block

대응 Rulebook: [breaker_block_rulebook.md](../rulebook/breaker_block_rulebook.md)

## 개념 정의

### 일반적인 정의

Breaker Block은 유동성을 제거한 가격이 반대 방향으로 구조를 관통한 뒤, 되돌림에서 반응 후보가 되는 candle zone이다.

단순한 support/resistance flip이 아니다. `기존 OB가 깨졌다`는 사실 하나만으로도 확정하지 않는다.

### ICT 정의

Episode 39는 bearish breaker를 다음 순서로 설명한다.

```text
buy-side liquidity pool 제거
-> 반전
-> short-term low 아래로 거래
-> bearish narrative 안에서 되돌림 관찰
```

같은 구간에서 zone은 다음 candle로 설명된다.

```text
short-term high를 제거하는 상승 이동 직전
-> 가장 낮은 down-close candle
-> bearish breaker candle
```

Episode 10도 bearish breaker를 `high -> low -> higher high` 패턴과 연결하고, short-term high를 제거하는 큰 상승 이동 직전의 down-close candle을 breaker candle로 설명한다.

코드 구현에서는 bearish 설명을 방향 대칭으로 확장한다.

```text
bearish breaker:
  BSL taken
  -> short-term low 아래 strict traversal
  -> 상승 이동 직전 lowest down-close candle zone을 위쪽 되돌림에서 관찰

bullish breaker:
  SSL taken
  -> short-term high 위 strict traversal
  -> 하락 이동 직전 highest up-close candle zone을 아래쪽 되돌림에서 관찰
```

bullish 규칙은 bearish 설명을 반전한 구현 대칭 규칙이다. 자막의 반복 설명이 bearish 사례에 집중되어 있으므로 `symmetry_inference = true`를 기록한다.

### OB와의 관계

Breaker candle은 반전 전 이동의 반대색 candle이므로 기존 [Order Block](order_block.md) 객체와 겹칠 수 있다. OB 모듈이 overrun 이력을 보존해 breaker 후보 입력으로 넘기는 것은 유용하다.

그러나 raw breaker의 hard dependency를 `source_ob_id`로 두지 않는다.

```text
wrong:
invalidated OB
-> always confirmed breaker

objective proxy:
liquidity taken
-> opposite short-term swing strict traversal
-> breaker candle zone
-> optional source_ob_id link
```

### 차트에서 의미

Breaker는 유동성을 제거한 첫 방향 이동이 유지되지 못하고, 반대 방향 전달로 전환됐다는 것을 보여주는 되돌림 후보이다.

```text
liquidity run
-> reversal traversal
-> breaker zone active
-> later retest
-> reaction 또는 overrun
```

### 시장 심리

OHLC 데이터는 실제 주문 흐름을 직접 확인하지 못한다. scanner는 다음 가격 행동을 프록시로 저장한다.

- 이전 고점 또는 저점의 유동성이 제거됐다.
- 가격이 반대편 short-term swing을 strict traversal했다.
- 반전 전 source candle zone으로 가격이 되돌아왔다.
- zone에서 예상 방향 반응 또는 overrun이 발생했다.

## ICT 전체 체계에서의 위치

### 선행 개념

- 필수: [Swing Points](swing_points.md)
- 필수: [BSL / SSL](bsl_ssl.md) `taken` event
- 필수: 같은 symbol과 timeframe의 닫힌 OHLC
- 권장 confluence: [Liquidity Sweep](liquidity_sweep.md)
- 권장 confluence: [Displacement](displacement.md)
- 선택 provenance: [Order Block](order_block.md) overrun history
- 선택 문맥: HTF draw on liquidity, session, economic calendar

### 후행 개념

- breaker retracement setup
- target liquidity selection
- HTF/LTF refinement
- retest 성과 분석

### 왜 중요한가

Breaker는 모든 FVG 또는 OB를 거래하지 않고, liquidity run 이후 반대 방향으로 전환된 구간을 별도 분류하게 한다.

Episode 5와 Episode 25는 단순 모델에서는 breaker가 필수가 아니라고 설명한다. 따라서 breaker는 최소 scanner의 필수 진입 신호가 아니라 독립적인 retracement setup profile이다.

## 실제 차트에서 발견하는 방법

### 찾는 순서: Bearish Breaker

1. active BSL source인 확정 swing high를 찾는다.
2. 해당 high 위로 가격이 strict traversal했는지 확인한다.
3. liquidity run 전후 구조에서 반전 기준 short-term low를 저장한다.
4. BSL taken 이후 가격이 해당 short-term low 아래로 strict traversal했는지 확인한다.
5. short-term high를 제거하는 상승 이동 직전의 down-close candle 중 가장 낮은 candle을 source로 저장한다.
6. source candle의 wick zone, body zone, midpoint, lower half를 저장한다.
7. 구조 관통 이후 가격이 zone으로 되돌아오면 retest interaction을 기록한다.
8. bearish narrative, displacement, FVG, MSS, target SSL을 metadata로 연결한다.

### 찾는 순서: Bullish Breaker

1. active SSL source인 확정 swing low를 찾는다.
2. 해당 low 아래로 가격이 strict traversal했는지 확인한다.
3. 반전 기준 short-term high를 저장한다.
4. SSL taken 이후 가격이 해당 short-term high 위로 strict traversal했는지 확인한다.
5. short-term low를 제거하는 하락 이동 직전의 up-close candle 중 가장 높은 candle을 source로 저장한다.
6. source candle zone으로 되돌아오면 retest interaction을 기록한다.
7. bullish 방향 대칭 규칙을 사용했다는 metadata를 저장한다.

### 유효 조건

- liquidity source swing은 사용 시점에 이미 확정되어 있어야 한다.
- liquidity run과 반대 방향 swing traversal은 같은 symbol과 timeframe에서 계산한다.
- 동일 가격 touch는 strict traversal이 아니다.
- breaker source candle id와 선택 window를 저장한다.
- zone 생성과 retest interaction을 분리한다.
- 방향성 narrative는 raw breaker 생성 조건이 아니라 setup eligibility 조건이다.

### 무효 조건

- 유동성 제거 없이 일반적인 support/resistance flip을 breaker로 부른다.
- 반대편 short-term swing traversal 없이 zone을 확정한다.
- 모든 invalidated OB를 breaker로 자동 승격한다.
- wick touch만으로 structure traversal이 있었다고 가정한다.
- 미래 봉을 읽고 과거 시점에 zone이 이미 확정됐다고 처리한다.

### 대표 예시: Bearish Breaker

```text
active BSL source = 105.00
break reference short-term low = 102.00

bar_a high = 105.25
-> BSL taken

later bar low = 101.75
-> short-term low 아래 strict traversal
-> bearish breaker confirmed

source candle:
  open=103.00, high=103.50, low=101.50, close=102.25
  down-close=true

wick zone = [101.50, 103.50]
lower half = [101.50, 102.50]
```

### Zone Refinement

| 표현 | 계산 | 용도 |
| --- | --- | --- |
| `wick_zone` | source candle high-low | 넓은 interaction 범위 |
| `body_zone` | source candle open-close | body retracement 분석 |
| `midpoint` | `(high + low) / 2` | half-zone 경계 |
| `lower_half` | `[low, midpoint]` | Episode 10 bearish 사례 refinement |

Episode 10은 bearish breaker candle의 lower half를 언급한다. 이를 모든 breaker의 유일한 진입 규칙으로 고정하지 않고 strategy refinement로 저장한다.

## 실전 활용

### 진입

```text
BSL taken
-> bearish reversal traversal
-> bearish breaker active
-> later breaker retest
-> target SSL이 남아 있음
-> short 후보
```

반대 방향은 대칭이다. retest 자체는 entry 확정이 아니다. narrative와 target liquidity를 함께 평가한다.

### 손절

- bearish setup: source candle wick zone high 위 buffer 검토
- bullish setup: source candle wick zone low 아래 buffer 검토
- half-zone refinement를 entry에 사용해도 stop reference는 별도 parameter로 둔다.

### 목표

- bearish breaker: 현재 가격 아래 active SSL
- bullish breaker: 현재 가격 위 active BSL
- target마다 source kind, timeframe, distance를 저장한다.

### 시간대

raw breaker 탐지는 시간대에 독립적이다. Episode 39는 afternoon session 사례를 제공하지만, session은 정의가 아니라 metadata이다.

## 흔한 오해

### 초보자가 잘못 해석하는 부분

- invalidated OB이면 항상 breaker라고 생각한다.
- 고점과 저점이 뒤집힌 모든 지점을 breaker라고 부른다.
- source candle zone 생성과 retest를 같은 사건으로 처리한다.
- narrative 없이 모양만 맞으면 충분하다고 생각한다.
- breaker가 모든 전략에 반드시 필요하다고 생각한다.

### ICT가 실제로 의미하는 것

- breaker는 유동성 제거와 반대 방향 전달을 함께 본다.
- Episode 39는 narrative가 없으면 breaker가 아니라 aimless speculation이라고 구분한다.
- source candle은 반전 전 liquidity를 제거한 이동과 연결된다.
- breaker는 사용할 수 있는 setup profile이지만 단순 모델의 필수 요소는 아니다.

## 다른 개념과의 연결

| 연결 개념 | Breaker Block의 역할 |
| --- | --- |
| Swing Points | liquidity source와 reversal traversal 기준 제공 |
| BSL / SSL | 선행 liquidity taken event와 후행 target 제공 |
| Liquidity Sweep | stop run 이후 reclaim metadata |
| MSS | 반대 방향 구조 변화 confluence |
| Displacement | 반전 이동의 속도와 방향 metadata |
| FVG | breaker retest 주변 imbalance confluence |
| Order Block | overrun 이력과 선택 provenance link |
| Daily Bias | narrative 방향 필터 |
| Killzone | intraday setup 시간 metadata |

## 코드 구현 시 문제점

### 사람이 보는 기준

사람은 어떤 liquidity pool을 기준으로 삼을지, 어느 short-term swing을 reversal reference로 선택할지, source candle 후보가 여러 개일 때 어떤 candle이 이동의 기원인지, HTF draw가 남아 있는지를 함께 본다.

### 코드로 구현 가능한 부분

- 확정 swing 기반 BSL/SSL taken event
- 반대편 short-term swing strict traversal
- down-close 및 up-close source candle 후보
- wick, body, midpoint, half-zone 계산
- retest, interaction count, overrun 추적
- optional `source_ob_id` 연결

### 코드로 구현 어려운 부분

- 가장 적절한 HTF narrative 선택
- 복수 liquidity pool 중 primary draw 선택
- source candle 검색 window의 보편적 고정값
- bullish 방향 대칭 규칙의 영상별 검증

이 항목은 `manual_review` 또는 versioned `research default`로 분리한다.

## 핵심 문장

1. Breaker는 단순 support/resistance flip이 아니다.
2. 유동성 제거가 먼저 필요하다.
3. 이후 반대편 short-term swing strict traversal을 확인한다.
4. bearish source는 상승 이동 직전 lowest down-close candle이다.
5. bullish source는 방향 대칭 구현 규칙으로 highest up-close candle을 사용한다.
6. raw breaker와 retest interaction을 분리한다.
7. `source_ob_id`는 유용한 링크지만 hard dependency가 아니다.
8. narrative는 raw zone 존재가 아니라 거래 적격성에 필요하다.
9. half-zone은 refinement이며 유일한 canonical zone이 아니다.
10. breaker는 최소 진입 모델의 필수 신호가 아니다.

## 이 개념의 본질

Breaker Block은 유동성 제거 뒤 반대 방향 구조 관통이 발생했을 때, 반전 전 source candle을 되돌림 관찰 zone으로 보존하는 가격 행동 객체이다.
