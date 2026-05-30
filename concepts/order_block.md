# Order Block

대응 Rulebook: [order_block_rulebook.md](../rulebook/order_block_rulebook.md)

## 개념 정의

### 일반적인 정의

Order Block(OB)은 가격 전달 방향이 바뀌는 지점을 추적하기 위한 가격 구간과 기준 level이다. ICT 2022 Mentorship에서 핵심은 단순히 “상승 직전 마지막 음봉” 또는 “하락 직전 마지막 양봉”을 찾는 것이 아니다.

Episode 3과 Episode 12는 OB를 `change in the state of delivery`로 설명한다.

### ICT 정의

Episode 3의 설명을 코드 관점으로 정리하면 다음과 같다.

```text
bullish OB candidate:
  연속 down-close candle series
  -> downside delivery 시작 open을 위로 관통
  -> delivery state가 bullish로 변화

bearish OB candidate:
  연속 up-close candle series
  -> upside delivery 시작 open을 아래로 관통
  -> delivery state가 bearish로 변화
```

방향별 candle series:

```text
down_close = close < open
up_close   = close > open
```

series의 첫 번째 봉 open을 `delivery_open`으로 저장한다.

```text
bullish state change:
  later high > bearish_delivery_series.first.open

bearish state change:
  later low < bullish_delivery_series.first.open
```

동일 가격 touch는 strict traversal이 아니다.

Episode 3은 복수 down-close candle을 하나의 continuous OB로 설명한다. Episode 12는 마지막 up-close candle 하나만 OB라고 부르는 축약 정의를 명시적으로 부정하고, consecutive series of up-close candles를 설명한다.

single candle도 길이가 `1`인 series로 허용한다. 그러나 `마지막 반대색 봉 1개`를 모든 상황의 canonical 정의로 강제하지 않는다.

### 차트에서 의미

OB는 특정 candle 색 자체가 아니라 가격 전달 방향 변화와 이후 되돌림 관찰 기준이다.

```text
directional candle series
-> delivery_open 반대 방향 strict traversal
-> raw OB confirmed
-> opening level, body zone, wick zone 저장
-> 이후 retracement interaction 관찰
```

Episode 3은 high-probability bearish OB 사례에서 다음 confluence를 함께 설명한다.

```text
gap
+ taken liquidity
+ MSS
-> high-probability bearish OB context
```

이 confluence는 raw OB 존재 조건과 분리한다.

### 시장 심리

ICT의 설명 모델은 다음과 같다.

- up-close series는 buy-side delivery, down-close series는 sell-side delivery를 보여준다.
- series의 시작 open이 반대 방향으로 관통되면 delivery state가 바뀐 것으로 본다.
- 이후 가격이 해당 opening level 또는 zone으로 되돌아오면 reaction 후보를 관찰한다.
- bullish move에서는 down-close candle zone이 support 역할을 할 수 있다.
- bearish move에서는 up-close candle zone이 resistance 역할을 할 수 있다.

OHLC 코드는 실제 기관 주문, 알고리즘 기억, 주문 수량을 확인할 수 없다. OB는 directional series와 opening-price traversal을 사용한 가격 행동 프록시이다.

## ICT 전체 체계에서의 위치

### 선행 개념

- 필수 입력: 같은 symbol과 timeframe의 닫힌 OHLC
- core reference: 연속 up-close 또는 down-close candle series
- 권장 confluence: [Fair Value Gap](fair_value_gap.md)
- 권장 confluence: [Displacement](displacement.md)
- 권장 confluence: [Market Structure Shift](market_structure_shift.md)
- 권장 confluence: [Liquidity Sweep](liquidity_sweep.md) 또는 BSL/SSL `taken` event
- 선택 문맥: [Dealing Range](dealing_range.md), premium/discount, HTF structure, session

raw OB confirmation은 FVG, MSS, sweep을 필수로 요구하지 않는다. 해당 사건은 setup eligibility와 ranking을 강화한다.

### 후행 개념

- [Breaker Block](breaker_block.md)
- retracement entry model
- multi-timeframe refinement
- target liquidity selection

### 왜 중요한가

OB를 단순 candle 색으로 탐지하면 후보가 과도하게 늘어난다. 반대로 마지막 반대색 봉 1개로만 고정하면 연속 delivery series와 opening level이라는 핵심을 잃는다.

```text
wrong:
last opposite-color candle
-> always OB

objective proxy:
directional close series
-> first opening level
-> opposite-direction strict traversal
-> raw OB
```

## 실제 차트에서 발견하는 방법

### 찾는 순서

1. symbol과 timeframe별 닫힌 OHLC를 시간순으로 정렬한다.
2. 연속된 down-close candle과 up-close candle series를 각각 추적한다.
3. series 시작 봉 open을 `delivery_open`으로 저장한다.
4. series 구성 봉의 high-low 전체 범위를 `wick_zone`으로 저장한다.
5. series 구성 봉의 open-close 전체 범위를 `body_zone`으로 저장한다.
6. down-close series의 `delivery_open` 위로 high가 strict traversal하면 bullish raw OB를 확정한다.
7. up-close series의 `delivery_open` 아래로 low가 strict traversal하면 bearish raw OB를 확정한다.
8. 이후 opening level, body zone, wick zone retracement interaction을 각각 기록한다.
9. FVG overlap, liquidity event, MSS, displacement, dealing range 위치를 metadata로 연결한다.
10. zone overrun과 setup invalidation을 기록하고 downstream breaker 후보로 전달한다.

### 유효 조건

- series 구성 봉은 같은 symbol과 timeframe에 속한다.
- series 구성 봉은 expected timestamp interval 기준으로 연속적이다.
- bullish candidate series는 `close < open` 봉으로 구성된다.
- bearish candidate series는 `close > open` 봉으로 구성된다.
- series 길이는 `1` 이상이다.
- `delivery_open`은 series 첫 번째 봉 open이다.
- bullish raw OB는 later `high > delivery_open`으로 확정한다.
- bearish raw OB는 later `low < delivery_open`으로 확정한다.
- event는 traversal 봉 close 이후 확정한다.
- opening level, body zone, wick zone을 분리해 저장한다.
- raw OB와 strategy eligibility를 분리한다.

### 무효 조건

- 모든 down-close candle을 자동으로 bullish OB라고 부른다.
- 모든 up-close candle을 자동으로 bearish OB라고 부른다.
- 마지막 반대색 봉 하나만 canonical OB라고 고정한다.
- candle series의 시작 open을 저장하지 않는다.
- 동일 가격 touch를 strict traversal로 처리한다.
- 미래 봉을 읽어 과거 시점에 OB를 미리 확정한다.
- FVG 또는 MSS가 있다는 이유만으로 OB를 자동 생성한다.
- zone overrun 뒤 과거 OB 이력을 삭제한다.
- wick zone, body zone, opening level을 하나의 모호한 가격으로 합친다.

### 대표 예시: Bullish OB

```text
down-close series:
  candle_a: open=103.00, close=102.00, low=101.50
  candle_b: open=102.00, close=101.75, low=101.00

delivery_open = candle_a.open = 103.00
wick_zone     = [101.00, 103.00]
body_zone     = [101.75, 103.00]

later bar:
  high = 103.25

103.25 > 103.00
-> bullish raw OB confirmed
```

### 대표 예시: Bearish OB

```text
up-close series:
  candle_a: open=104.00, close=105.00, high=105.25
  candle_b: open=105.00, close=105.50, high=106.00

delivery_open = candle_a.open = 104.00
wick_zone     = [104.00, 106.00]
body_zone     = [104.00, 105.50]

later bar:
  low = 103.75

103.75 < 104.00
-> bearish raw OB confirmed
```

### Single Candle과 Consecutive Series

| 형태 | 저장 방식 |
| --- | --- |
| 반대 방향 종가 봉 1개 | 길이 `1`인 series |
| 연속 반대 방향 종가 봉 여러 개 | 하나의 consecutive series |
| 중간에 doji 또는 반대 방향 봉 포함 | 기본 정책에서는 series 종료 |

Episode 3, 12, 40은 복수 candle series를 하나의 OB로 다루는 사례를 제공한다.

### Opening Level, Body Zone, Wick Zone

OB에는 서로 다른 가격 표현이 필요하다.

| 표현 | 계산 | 용도 |
| --- | --- | --- |
| `delivery_open` | series 첫 번째 봉 open | state delivery 변화 기준 |
| `body_zone` | series 구성 봉 body 전체 범위 | retracement refinement 후보 |
| `wick_zone` | series 구성 봉 high-low 전체 범위 | 넓은 interaction 관찰 범위 |

Episode 17은 긴 wick보다 body retracement를 선호하는 사례를 설명한다. Episode 3은 opening price를 시간축으로 연장해 limit reference로 사용하는 사례를 설명한다.

어느 refinement를 진입에 사용할지는 strategy parameter로 분리한다.

### Raw OB와 High-Probability Context

| 구분 | 조건 |
| --- | --- |
| raw OB | directional series와 delivery open 반대 방향 traversal |
| FVG overlap OB | raw OB와 FVG overlap |
| structure-aligned OB | raw OB와 같은 방향 MSS 또는 displacement |
| liquidity-aligned OB | raw OB와 선행 BSL/SSL taken 또는 sweep |
| high-probability context | gap, taken liquidity, MSS 등 configured confluence 충족 |

모든 raw OB를 거래하지 않는다.

## 실전 활용

### 진입

OB 단독 진입은 최소 scanner의 기본 전략이 아니다.

```text
SSL taken 또는 sweep
-> bullish displacement
-> bullish MSS confluence
-> bullish FVG
-> bullish OB retracement interaction
-> long 후보

BSL taken 또는 sweep
-> bearish displacement
-> bearish MSS confluence
-> bearish FVG
-> bearish OB retracement interaction
-> short 후보
```

공격적 profile은 opening level 또는 lower-timeframe imbalance interaction을 사용하고, 보수적 profile은 retracement와 구조 confluence를 추가로 요구한다.

### 손절

OB는 손절 후보를 제공하지만 자동 확정하지 않는다.

- bullish setup: wick zone low 또는 linked SSL sweep extreme 아래 buffer 검토
- bearish setup: wick zone high 또는 linked BSL sweep extreme 위 buffer 검토
- body refinement를 entry에 사용해도 stop reference는 별도 parameter로 둔다.

### 목표

- bullish OB setup: 현재 가격 위 active BSL
- bearish OB setup: 현재 가격 아래 active SSL
- target마다 timeframe, source kind, distance, IRL/ERL tag를 기록한다.

### 시간대

raw OB 탐지는 시간대에 독립적이다.

- Episode 20: London killzone 문맥
- Episode 40: London session interaction 사례
- Episode 41: afternoon session과 FVG, OB, MSS 결합 사례

세션과 경제지표는 OB 정의가 아니라 setup metadata이다.

## 흔한 오해

### 초보자가 잘못 해석하는 부분

- 모든 down-close candle을 bullish OB라고 부른다.
- 모든 up-close candle을 bearish OB라고 부른다.
- 마지막 반대색 candle 하나만 항상 OB라고 생각한다.
- 연속 candle series를 개별 OB 여러 개로 중복 저장한다.
- opening level과 zone 경계를 구분하지 않는다.
- OB가 있으면 displacement, MSS, FVG도 자동으로 있다고 가정한다.
- zone touch 하나만으로 entry를 확정한다.
- body refinement를 모든 상황의 유일한 규칙으로 고정한다.

### ICT가 실제로 의미하는 것

- OB의 핵심은 state of delivery 변화이다.
- 연속 같은 방향 종가 봉은 하나의 continuous delivery series가 될 수 있다.
- series 시작 open의 반대 방향 관통이 중요한 기준이다.
- 단일 봉 OB도 길이 `1`인 series로 표현할 수 있다.
- opening level, body zone, wick zone은 서로 다른 목적을 가진다.
- gap, liquidity taken, MSS는 high-probability context를 강화한다.

## 다른 개념과의 연결

### 관련 개념

| 연결 개념 | OB의 역할 |
| --- | --- |
| Opening Price | delivery state 변화 기준 level 제공 |
| FVG | OB overlap과 lower-timeframe refinement 후보 |
| Displacement | OB 뒤 repricing 방향 confluence |
| MSS | structure-aligned OB metadata |
| Liquidity Sweep | 선행 stop run context |
| BSL / SSL | 선행 liquidity event와 target 후보 |
| Dealing Range | premium/discount 위치 계산 |
| Breaker Block | 실패한 OB zone의 downstream 파생 후보 |
| Economic Calendar Filter | event window metadata |

### 의존 관계

```text
closed OHLC
-> directional close series
-> delivery_open
-> opposite-direction strict traversal
-> raw OB
  -> opening level interaction
  -> body zone interaction
  -> wick zone interaction
  -> overrun history

raw OB
+ optional FVG overlap
+ optional liquidity event
+ optional MSS
+ optional displacement
-> ranked OB retracement setup

overrun raw OB
-> Breaker Block candidate input
```

## 코드 구현 시 문제점

### 사람이 보는 기준

사람은 series의 시작과 끝, 어떤 timeframe의 OB를 우선할지, body 또는 wick 중 어디를 refinement로 사용할지, FVG와 구조가 얼마나 잘 정렬되는지 함께 본다. `좋은 OB`, `강한 OB`는 코드 규칙이 아니다.

### 코드로 구현 가능한 부분

- up-close 및 down-close contiguous series 생성
- series 첫 open 저장
- body zone과 wick zone 계산
- delivery open strict traversal 탐지
- single candle 및 multi-candle series 구분
- 이후 zone interaction 횟수와 최초 touch 저장
- direction별 overrun과 close-through 기록
- FVG overlap ratio 계산
- liquidity, displacement, MSS id 연결
- HTF/LTF OB 독립 저장

### 코드로 구현 어려운 부분

- 모든 chart context에 공통인 최적 series 종료 정책
- doji를 series에 포함할지 여부
- opening level, body zone, wick zone 중 최적 entry refinement
- 복수 중첩 OB 중 narrative상 핵심 zone 선택
- HTF zone 안 LTF refinement의 최적 결합
- OB overrun 뒤 breaker 전환의 세부 정책

어려운 부분은 `research default`, `manual_review_reason`, strategy-specific ranking으로 분리한다.

## 핵심 문장

1. OB의 핵심은 반대색 candle 하나가 아니라 state of delivery 변화이다.
2. Bullish OB candidate는 연속 down-close candle series에서 시작한다.
3. Bearish OB candidate는 연속 up-close candle series에서 시작한다.
4. Series 첫 번째 봉 open을 `delivery_open`으로 저장한다.
5. Bullish raw OB는 delivery open 위 strict traversal로 확인한다.
6. Bearish raw OB는 delivery open 아래 strict traversal로 확인한다.
7. Single candle은 길이 `1`인 series이며 복수 consecutive candle도 하나의 OB가 될 수 있다.
8. Opening level, body zone, wick zone을 분리한다.
9. FVG, liquidity taken, MSS, displacement는 raw OB 존재 조건이 아니라 ranking metadata이다.
10. OB overrun 이력은 삭제하지 않고 Breaker Block 후보에 전달한다.

## 이 개념의 본질

Order Block은 연속 directional close series의 시작 open이 반대 방향으로 strict traversal되어 delivery state가 바뀐 시점을 기록하고, opening level과 refinement zone의 이후 interaction을 추적하는 가격 행동 객체이다.

## Transcript 근거 분류

### 에피소드 전반에 반복되는 핵심 근거

- Episode 3: down-close candle series 전체를 하나의 continuous OB로 분류하고 opening price를 연장
- Episode 3: up-close 및 down-close series 시작 open의 반대 방향 관통을 state of delivery 변화로 설명
- Episode 3: gap, taken liquidity, MSS가 결합된 high-probability bearish OB context
- Episode 3: 모든 down-close candle과 up-close candle을 자동 OB로 보는 오해를 배제
- Episode 12: 마지막 반대색 봉 1개 축약 정의를 명시적으로 부정하고 consecutive series를 설명
- Episode 12: bearish move에서 up-close candle zone이 resistance 역할을 하는 사례
- Episode 13: bullish move에서 down-close candle zone이 support 역할을 하는 사례
- Episode 16: down-close candle bullish OB와 retracement interaction
- Episode 40: 복수 down-close candle을 하나의 consecutive OB로 분류
- Episode 41: FVG, MSS, bullish OB, OTE가 결합된 사례

### 특정 구간에 집중된 보조 근거

- Episode 17: long wick보다 candle body retracement를 선호하는 refinement 사례
- Episode 20: London killzone 안 OB interaction 사례
- Episode 26: 실시간 tape reading 중 bullish OB annotation
- Episode 27: down-close candle OB와 FVG overlap 사례
- Episode 35: daily bearish OB와 15분 OB 문맥

### 검색 분포

정리본 `transcripts/*.en.txt` 전체 검색 기준:

| 표현 | 검색 라인 수 | 등장 에피소드 파일 수 |
| --- | ---: | ---: |
| `order block` | 593 | 28 |
| `state of delivery` | 21 | 3 |
| `down close candle` | 45 | 7 |
| `down closed candles` | 66 | 11 |
| `up close candle` | 51 | 9 |
| `up closed candles` | 30 | 3 |

수치는 자막 중복을 포함하므로 절대 빈도가 아니라 반복 분포 확인용이다.
