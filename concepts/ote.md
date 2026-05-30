# OTE

대응 Rulebook: [ote_rulebook.md](../rulebook/ote_rulebook.md)

## 개념 정의

### 일반적인 정의

OTE(Optimal Trade Entry)는 선택한 가격 leg의 되돌림 깊이를 Fibonacci 비율로 정규화해 관찰하는 retracement zone이다.

ICT 2022 Mentorship 자막에서 반복 확인되는 기본 구간은 `0.62~0.79`이다.

```text
OTE retracement zone = [0.62, 0.79]
```

`0.705`는 전체 정제 자막에서 직접 검색되지 않는다. 이는 `0.62`와 `0.79`의 산술 중간값이다.

```text
derived_midpoint = (0.62 + 0.79) / 2 = 0.705
```

scanner는 `0.705`를 저장할 수 있지만, 원문에 직접 제시된 별도 threshold가 아니라 `derived metadata`로 표시한다.

### ICT 정의

Episode 22는 OTE를 구성하는 level로 `62`와 `79`를 명시한다. Episode 38은 bullish leg에서 low부터 high까지 fib를 긋고 `62 to 79` 되돌림에서 매수하는 패턴을 설명한다. Episode 19도 `62`와 `79` retracement level을 OTE range라고 설명한다.

```text
bullish OTE:
  low -> high leg
  -> high에서 아래 방향으로 0.62~0.79 retracement
  -> discount 되돌림 후보

bearish OTE:
  high -> low leg
  -> low에서 위 방향으로 0.62~0.79 retracement
  -> premium 되돌림 후보
```

### Anchor Refinement

OTE는 비율만으로 끝나지 않는다. 측정 anchor를 어떻게 선택했는지 저장해야 한다.

Episode 22는 premium/discount를 볼 때 high-low range를 사용할 수 있지만, OTE에서는 swing 내부의 body를 선호한다고 설명한다.

```text
body-refined anchor:
  lower anchor = lowest open or close in lower swing
  upper anchor = highest open or close in upper swing
```

따라서 최소 구현은 확정 dealing range endpoint를 사용하고, 고급 구현은 `body_refined` anchor policy를 추가한다.

### 자막 표기 충돌

Episode 41 정제본과 대응 `-orig` 대조본은 `62 to 70`으로 전사되어 있다. 다른 여러 Episode의 반복 설명은 `62~79`를 명시한다.

이 저장소는 반복 근거가 있는 `0.62~0.79`를 기본 profile로 사용한다. Episode 41 표현은 삭제하거나 임의 교정하지 않고 `transcript_conflict`와 영상 수동 확인 대상으로 기록한다.

### 차트에서 의미

OTE는 `어디까지 되돌아왔는가`를 정규화한다. 방향성, liquidity target, MSS, FVG, OB를 대신하지 않는다.

```text
selected range
-> direction-aware retracement calculation
-> OTE zone
-> touch 또는 overlap metadata
-> strategy eligibility 평가
```

### 시장 심리

OHLC 코드가 Fibonacci 숫자 자체의 시장 원인을 검증할 수는 없다. scanner는 선택한 leg에 대해 반복 가능한 좌표를 만들고, 되돌림 후보와 다른 PD Array가 겹치는지를 측정한다.

## ICT 전체 체계에서의 위치

### 선행 개념

- 필수: [Dealing Range](dealing_range.md) 또는 재현 가능한 anchor range
- 필수: range direction
- 권장 위치 문맥: [Premium / Discount / Equilibrium](premium_discount_equilibrium.md)
- 선택 confluence: [Fair Value Gap](fair_value_gap.md)
- 선택 confluence: [Order Block](order_block.md)
- 선택 confluence: [Market Structure Shift](market_structure_shift.md)
- 선택 문맥: liquidity target, Daily Bias, session, economic calendar

### 후행 개념

- OTE touch interaction
- FVG 또는 OB overlap ranking
- retracement setup eligibility
- anchor policy별 백테스트

### 왜 중요한가

OTE는 price를 절대값이 아닌 range 내부 위치로 비교하게 한다.

Episode 41은 FVG, OB, OTE가 결합된 패턴을 강조한다. 반면 Episode 1과 Episode 7은 정밀한 OTE entry 또는 Fibonacci 없이도 단순 모델을 설명한다. 따라서 raw OTE zone은 객관적으로 계산하되, 모든 setup의 필수 진입 조건으로 강제하지 않는다.

## 실제 차트에서 발견하는 방법

### 찾는 순서: Bullish OTE

1. low에서 high로 완료된 bullish range를 선택한다.
2. range id, timeframe, 두 endpoint의 pivot id, 확정 시각을 저장한다.
3. anchor policy를 선택한다.
4. `span = anchor_high - anchor_low`를 계산한다.
5. high에서 아래 방향으로 `0.62`와 `0.79` retracement 가격을 계산한다.
6. 두 가격 사이를 bullish OTE zone으로 저장한다.
7. zone이 discount 쪽에 위치하는지 검증한다.
8. 이후 zone touch와 FVG 또는 OB overlap을 기록한다.

```text
bullish_price(r) = anchor_high - r * span

bullish_zone_low  = bullish_price(0.79)
bullish_zone_high = bullish_price(0.62)
```

### 찾는 순서: Bearish OTE

1. high에서 low로 완료된 bearish range를 선택한다.
2. anchor policy와 endpoint provenance를 저장한다.
3. low에서 위 방향으로 `0.62`와 `0.79` retracement 가격을 계산한다.
4. 두 가격 사이를 bearish OTE zone으로 저장한다.
5. zone이 premium 쪽에 위치하는지 검증한다.
6. 이후 zone touch와 confluence overlap을 기록한다.

```text
bearish_price(r) = anchor_low + r * span

bearish_zone_low  = bearish_price(0.62)
bearish_zone_high = bearish_price(0.79)
```

### 유효 조건

- `anchor_high > anchor_low`여야 한다.
- range endpoint는 사용 시점에 이미 확정되어 있어야 한다.
- `range_id`, direction, anchor policy를 저장한다.
- bullish와 bearish 계산 방향을 분리한다.
- raw zone 생성과 이후 touch interaction을 분리한다.
- FVG 또는 OB overlap은 metadata이며 raw zone의 hard dependency가 아니다.

### 무효 조건

- 화면에 맞추기 위해 range를 사후 변경한다.
- 미확정 pivot 또는 미래 봉을 anchor에 사용한다.
- bullish와 bearish retracement 계산 방향을 뒤집는다.
- `0.705`를 자막이 직접 제시한 별도 threshold라고 기록한다.
- Episode 41의 `62 to 70` 전사 충돌을 근거 없이 자동 교정한다.
- OTE touch 하나만으로 entry를 확정한다.

### 대표 예시: Bullish OTE

```text
anchor_low  = 100.00
anchor_high = 110.00
span        = 10.00

0.62 retracement = 110.00 - 0.62 * 10.00 = 103.80
0.79 retracement = 110.00 - 0.79 * 10.00 = 102.10

bullish OTE zone = [102.10, 103.80]
derived midpoint = 102.95
```

### 대표 예시: Bearish OTE

```text
anchor_high = 110.00
anchor_low  = 100.00
span        = 10.00

0.62 retracement = 100.00 + 0.62 * 10.00 = 106.20
0.79 retracement = 100.00 + 0.79 * 10.00 = 107.90

bearish OTE zone = [106.20, 107.90]
derived midpoint = 107.05
```

## 실전 활용

### 진입

OTE touch 단독으로 alert를 만들지 않는다.

```text
bullish directional context
-> selected bullish range
-> discount OTE zone
-> bullish FVG 또는 OB overlap
-> target BSL 유지
-> long 후보

bearish directional context
-> selected bearish range
-> premium OTE zone
-> bearish FVG 또는 OB overlap
-> target SSL 유지
-> short 후보
```

### 손절

OTE는 손절 위치를 자동 확정하지 않는다.

- bullish setup: anchor low, linked SSL sweep extreme, linked OB low 검토
- bearish setup: anchor high, linked BSL sweep extreme, linked OB high 검토
- stop reference와 buffer는 strategy parameter로 둔다.

### 목표

- bullish OTE setup: 현재 가격 위 active BSL
- bearish OTE setup: 현재 가격 아래 active SSL
- OTE extension projection은 별도 모듈로 분리한다.

### 시간대

raw OTE 계산은 시간대에 독립적이다. intraday setup에서는 session tag를 추가하고, HTF/LTF range를 구분한다.

## 흔한 오해

### 초보자가 잘못 해석하는 부분

- 모든 Fibonacci retracement를 OTE라고 부른다.
- 임의 range를 사후 선택해 결과를 맞춘다.
- `0.705`가 자막에서 직접 명시된 유일한 entry price라고 생각한다.
- range endpoint와 body-refined anchor를 섞는다.
- OTE가 없으면 ICT setup이 항상 무효라고 생각한다.
- touch만 있으면 방향성과 target 검토가 필요 없다고 생각한다.

### ICT가 실제로 의미하는 것

- 반복 확인되는 OTE level은 `0.62`와 `0.79`이다.
- bullish OTE는 low-high leg의 discount retracement이다.
- bearish OTE는 high-low leg의 premium retracement이다.
- Episode 22는 OTE anchor에서 open-close body refinement를 설명한다.
- OTE는 FVG와 OB 등 다른 조건과 결합할 수 있지만 단순 모델의 필수 신호는 아니다.

## 다른 개념과의 연결

| 연결 개념 | OTE의 역할 |
| --- | --- |
| Swing Points | 재현 가능한 range endpoint 후보 |
| Dealing Range | anchor range와 provenance 제공 |
| Premium / Discount / Equilibrium | bullish discount, bearish premium 위치 검증 |
| FVG | OTE zone overlap confluence |
| Order Block | OTE zone overlap confluence |
| MSS | range 후보와 directional context metadata |
| Liquidity Sweep | 되돌림 전 사건과 stop reference metadata |
| BSL / SSL | 방향별 target liquidity 후보 |
| Daily Bias | setup 방향 필터 |

## 코드 구현 시 문제점

### 사람이 보는 기준

사람은 어느 range가 현재 narrative에 적합한지, wick endpoint와 body refinement 중 어느 anchor를 쓸지, 여러 timeframe range 중 무엇을 우선할지 판단한다.

### 코드로 구현 가능한 부분

- versioned anchor range 저장
- 방향별 `0.62`, `0.79` 가격 계산
- derived `0.705` 비율과 가격 계산
- premium/discount consistency 검증
- zone touch와 interaction count
- FVG, OB overlap 계산

### 코드로 구현 어려운 부분

- 복수 range 중 primary narrative range 선택
- 모든 시장과 timeframe에 공통인 anchor refinement 정책
- Episode 41 `62 to 70` 전사의 실제 음성 판독
- OTE overlap의 전략별 가중치

이 항목은 `manual_review`, `transcript_conflict`, versioned `research default`로 분리한다.

## 핵심 문장

1. OTE는 선택한 range의 retracement zone이다.
2. 반복 자막 근거가 있는 기본 구간은 `0.62~0.79`이다.
3. `0.705`는 두 경계의 파생 midpoint이지 직접 인용 threshold가 아니다.
4. bullish OTE는 low-high leg의 discount 되돌림이다.
5. bearish OTE는 high-low leg의 premium 되돌림이다.
6. range id와 anchor policy를 저장해야 결과를 재현할 수 있다.
7. Episode 22 body refinement는 최소 규칙과 분리한다.
8. FVG와 OB overlap은 confluence metadata다.
9. OTE touch만으로 entry를 확정하지 않는다.
10. Episode 41 전사 충돌은 영상 수동 확인 대상으로 남긴다.

## 이 개념의 본질

OTE는 versioned range anchor에 대해 방향별 `0.62~0.79` retracement 가격을 계산하고, 되돌림 후보와 다른 PD Array의 overlap을 비교하는 위치 객체이다.
