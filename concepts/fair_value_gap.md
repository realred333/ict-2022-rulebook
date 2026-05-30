# Fair Value Gap

대응 Rulebook: [fair_value_gap_rulebook.md](../rulebook/fair_value_gap_rulebook.md)

## 개념 정의

### 일반적인 정의

Fair Value Gap(FVG)은 연속된 3개 캔들에서 첫 번째 봉과 세 번째 봉의 wick이 겹치지 않아, 가운데 봉의 가격 전달 과정에 남는 비중첩 구간이다.

```text
bullish FVG:
  candle_1.high < candle_3.low
  zone = [candle_1.high, candle_3.low]

bearish FVG:
  candle_1.low > candle_3.high
  zone = [candle_3.high, candle_1.low]
```

동일 가격 경계는 gap이 아니다.

### ICT 정의

Episode 6은 FVG를 `institutional order flow pattern`이자 3캔들 formation으로 설명한다.

- bearish FVG: 첫 번째 봉의 low와 세 번째 봉의 high 사이 비중첩 구간
- bullish FVG: 첫 번째 봉의 high와 세 번째 봉의 low 사이 비중첩 구간
- 가운데 봉: gap이 형성되는 방향성 전달 봉

Episode 6은 bearish FVG에서 가격이 아래로 전달되는 동안 해당 구간에 매수자에게 효율적으로 가격이 제공되지 않았다고 설명한다. 이후 가격이 그 구간으로 되돌아오면 rebalance 후보가 된다.

```text
directional delivery
-> candle_1과 candle_3 wick 비중첩
-> raw FVG zone 생성
-> 이후 zone retracement 또는 target interaction 관찰
```

FVG zone의 기하학적 탐지와 거래 setup 적격성은 다르다.

```text
raw FVG existence
!= qualified trade setup
```

Episode 6은 liquidity run 뒤 displacement range 내부 FVG를 찾도록 설명한다. Episode 41은 FVG가 존재해도 의미 있는 displacement가 없으면 참여하지 않는 사례를 제공한다.

### 차트에서 의미

FVG는 빠른 가격 전달 과정에서 양방향 거래가 충분히 겹치지 않은 구간을 표시한다.

- 되돌림 후보: displacement 이후 가격이 FVG로 돌아오는지 관찰한다.
- 목표 후보: 반대편에 남아 있는 old FVG 또는 imbalance를 target metadata로 사용할 수 있다.
- 구조 확인 보조: liquidity run, MSS, displacement와 같은 방향으로 생성된 FVG인지 확인한다.

FVG 자체는 주문 주체를 직접 증명하지 않는다. OHLC에서 관찰 가능한 것은 wick 비중첩 구간과 이후 상호작용이다.

### 시장 심리

ICT의 설명 모델은 다음과 같다.

- 가격이 한 방향으로 빠르게 전달되면 가운데 봉 주변에 한쪽 방향 거래만 제공된 구간이 남는다.
- 이후 가격은 해당 inefficient area로 되돌아와 rebalance할 수 있다.
- 모든 FVG가 즉시 되돌림되거나 반응하는 것은 아니다.
- liquidity run, MSS, displacement 없이 발견한 작은 gap을 모두 진입 신호로 사용하지 않는다.

코드는 효율성, 기관 주문, 실제 체결 의도를 확인할 수 없다. FVG는 `3-candle wick non-overlap`을 사용한 객관적 가격 zone 프록시이다.

## ICT 전체 체계에서의 위치

### 선행 개념

- 필수 입력: 같은 symbol과 timeframe의 연속된 닫힌 OHLC 3개
- 권장 confluence: [Displacement](displacement.md)
- 권장 confluence: [Market Structure Shift](market_structure_shift.md)
- 권장 confluence: [Liquidity Sweep](liquidity_sweep.md) 또는 BSL/SSL `taken` event
- 선택 문맥: [Dealing Range](dealing_range.md), premium/discount, HTF zone, session

raw FVG 탐지는 sweep, MSS, displacement를 필수로 요구하지 않는다. 해당 사건은 setup ranking과 전략 filter에 연결한다.

### 후행 개념

- [Order Block](order_block.md)
- [Breaker Block](breaker_block.md)
- retracement entry model
- target liquidity 및 imbalance selection

### 왜 중요한가

FVG는 차트에서 반복적으로 탐지할 수 있는 명확한 zone이다. 자막 전반에서 진입 retracement와 목표 후보로 반복된다.

```text
raw geometry:
closed OHLC 3개
-> FVG zone

strengthened setup:
liquidity taken 또는 sweep
-> opposite-direction displacement
-> MSS confluence
-> same-leg FVG
-> retracement 후보
```

raw geometry와 strengthened setup을 분리하면 모든 작은 gap을 거래하는 오류를 줄일 수 있다.

## 실제 차트에서 발견하는 방법

### 찾는 순서

1. symbol과 timeframe별 OHLC를 시간순으로 정렬한다.
2. 연속된 닫힌 봉 `candle_1`, `candle_2`, `candle_3`을 읽는다.
3. `candle_1.high < candle_3.low`이면 bullish FVG를 생성한다.
4. `candle_1.low > candle_3.high`이면 bearish FVG를 생성한다.
5. zone low, zone high, gap size tick, ATR 비율을 저장한다.
6. 이후 봉이 zone과 겹치는지 추적한다.
7. 최초 touch, 부분 rebalance, 완전 rebalance를 분리한다.
8. displacement leg, MSS, liquidity event, OB overlap, dealing range 위치를 별도 metadata로 연결한다.

### 유효 조건

- 세 봉이 모두 닫혀 있다.
- 세 봉은 같은 symbol과 timeframe에 속한다.
- 세 봉 timestamp가 기대 interval 기준으로 연속적이다.
- bullish FVG는 `candle_1.high < candle_3.low`이다.
- bearish FVG는 `candle_1.low > candle_3.high`이다.
- gap size가 `0`보다 크다.
- zone 생성 시각은 세 번째 봉 close 시각이다.
- zone interaction은 생성 이후 봉에서만 계산한다.
- raw zone과 strategy eligibility 상태를 분리한다.

### 무효 조건

- 세 번째 봉이 닫히기 전에 FVG를 확정한다.
- 첫 번째 봉과 세 번째 봉 wick이 겹치는데 FVG로 표시한다.
- 동일 가격 touch를 양수 gap으로 처리한다.
- 결측 봉 또는 비정상 timestamp gap을 정상 3봉 formation으로 처리한다.
- displacement가 없다는 이유로 raw FVG 기록을 삭제한다.
- FVG가 있다는 이유만으로 displacement, MSS, 진입을 자동 확정한다.
- 완전히 rebalance된 zone의 과거 이력을 삭제한다.
- wick interaction과 close-through를 같은 상태로 합친다.

### 대표 예시: Bullish FVG

```text
candle_1:
  high = 101.00

candle_3:
  low = 102.00

101.00 < 102.00
-> bullish FVG
-> zone_low  = 101.00
-> zone_high = 102.00
-> gap_size  = 1.00
```

이후 가격이 위에서 아래로 되돌아온다.

```text
later low = 101.60
-> touched
-> partially_rebalanced
-> fill_ratio = (102.00 - 101.60) / 1.00 = 0.40

later low = 100.90
-> fully_rebalanced
-> fill_ratio = 1.00
```

### 대표 예시: Bearish FVG

```text
candle_1:
  low = 105.00

candle_3:
  high = 104.00

105.00 > 104.00
-> bearish FVG
-> zone_low  = 104.00
-> zone_high = 105.00
-> gap_size  = 1.00
```

이후 가격이 아래에서 위로 되돌아온다.

```text
later high = 104.25
-> touched
-> partially_rebalanced
-> fill_ratio = (104.25 - 104.00) / 1.00 = 0.25
```

### Zone Interaction과 Setup Eligibility

zone 이력과 전략 적격성은 분리한다.

| 구분 | 의미 |
| --- | --- |
| `active` | 생성 후 아직 touch되지 않은 raw zone |
| `touched` | 이후 봉 range가 zone과 처음 겹침 |
| `partially_rebalanced` | zone 내부 일부 가격이 다시 거래됨 |
| `fully_rebalanced` | 반대 경계까지 거래되어 fill ratio가 `1.0` |
| `overrun` | zone을 넘어 거래됨 |
| `setup_invalidated` | 특정 전략의 유지 조건 위반 |

`fully_rebalanced` 또는 `overrun`이 발생해도 raw zone event를 삭제하지 않는다.

### FVG와 Imbalance

자막에서는 `fair value gap`, `imbalance`, `inefficient area`, `rebalance`가 연결되어 사용된다. 그러나 모든 imbalance 표현을 동일한 3캔들 FVG로 자동 변환하지 않는다.

- raw FVG: 3캔들 wick 비중첩 수식으로 탐지
- broader imbalance: FVG 외 가격 비효율 설명을 포함할 수 있으므로 별도 연구 대상

최소 scanner는 raw FVG만 자동 생성한다.

### 복수 FVG

Episode 3은 복수 FVG 중 특정 구간을 선택하는 사례를 설명한다. 모든 복수 zone 선택을 하나의 규칙으로 고정할 수 없다.

scanner는 다음 metadata를 출력한다.

```text
relative_position
distance_ticks
timeframe
creation_time
linked_displacement_leg_id
first_touch
fill_ratio
```

primary zone 선택은 strategy-specific ranking으로 분리한다.

## 실전 활용

### 진입

FVG 단독 진입은 최소 scanner의 기본 전략이 아니다.

```text
BSL taken 또는 sweep
-> bearish displacement
-> bearish MSS confluence
-> 같은 leg 내부 bearish FVG retracement
-> short 후보

SSL taken 또는 sweep
-> bullish displacement
-> bullish MSS confluence
-> 같은 leg 내부 bullish FVG retracement
-> long 후보
```

Episode 6은 displacement range 내부 FVG가 없으면 해당 model의 trade가 없다고 설명한다. 이는 raw FVG 탐지 규칙이 아니라 강화된 거래 모델의 filter이다.

### 손절

FVG는 손절을 직접 확정하지 않는다.

- long 후보: linked SSL sweep extreme 또는 구조 low 아래 buffer 검토
- short 후보: linked BSL sweep extreme 또는 구조 high 위 buffer 검토
- FVG 반대 경계 close-through를 setup invalidation 정책으로 사용할 수 있다.

close-through 정책은 전략 parameter이며 raw FVG zone 삭제 규칙이 아니다.

### 목표

- bullish setup target: 현재 가격 위 active BSL 또는 bearish FVG/imbalance 후보
- bearish setup target: 현재 가격 아래 active SSL 또는 bullish FVG/imbalance 후보
- old FVG를 target으로 사용할 때 timeframe, distance, fill ratio를 저장한다.

Episode 6은 반대 방향 FVG를 profit target으로 사용하는 예시도 제공한다.

### 시간대

raw FVG 탐지는 시간대에 독립적이다.

- session tag: London, New York, lunch hour, afternoon
- calendar tag: high/medium impact event window
- timezone: `America/New_York`

세션과 경제지표는 FVG 정의가 아니라 setup filter metadata이다.

## 흔한 오해

### 초보자가 잘못 해석하는 부분

- 모든 작은 chart gap을 FVG라고 부른다.
- wick overlap을 무시하고 body만 비교한다.
- 세 번째 봉 close 전에 FVG를 확정한다.
- 모든 FVG를 즉시 거래한다.
- FVG가 있으면 displacement와 MSS도 자동으로 있다고 생각한다.
- rebalance된 zone을 데이터에서 삭제한다.
- broader imbalance 표현을 모두 raw 3캔들 FVG로 강제 변환한다.
- 복수 FVG 중 선택 기준을 근거 없이 하나로 고정한다.

### ICT가 실제로 의미하는 것

- FVG는 3캔들 formation의 wick 비중첩 구간이다.
- 가운데 봉의 방향성 전달 과정에서 gap이 형성된다.
- 이후 가격이 해당 inefficient area로 되돌아와 rebalance할 수 있다.
- 이상적인 model에서는 liquidity run, displacement, MSS 문맥 안의 FVG를 찾는다.
- FVG 존재만으로 trade setup이 완성되지 않는다.

## 다른 개념과의 연결

### 관련 개념

| 연결 개념 | FVG의 역할 |
| --- | --- |
| Displacement | FVG가 포함될 수 있는 방향성 leg 제공 |
| MSS | 같은 leg의 구조 변화 confluence 제공 |
| Liquidity Sweep | FVG setup을 강화하는 선행 사건 |
| BSL / SSL | liquidity run 방향과 target 후보 제공 |
| Order Block | zone overlap 또는 entry refinement metadata |
| Dealing Range | premium/discount 위치 계산 기준 |
| Opening Price | 시간 문맥과 가격 기준 metadata |
| Economic Calendar Filter | event window metadata |

### 의존 관계

```text
closed OHLC 3개
-> raw FVG zone
  -> interaction history
  -> fill ratio

liquidity taken 또는 sweep
-> displacement leg
-> MSS confluence
-> same-direction raw FVG link
-> ranked retracement setup
```

## 코드 구현 시 문제점

### 사람이 보는 기준

사람은 FVG의 크기, displacement leg 안 위치, 선행 liquidity run, MSS, HTF 방향, 세션, 복수 zone 중 상대적 중요도를 함께 본다. `깨끗한 FVG`, `좋은 FVG`는 그대로 코드 규칙이 아니다.

### 코드로 구현 가능한 부분

- bullish/bearish 3캔들 wick 비중첩 탐지
- zone low, high, midpoint, gap size 계산
- gap size tick과 ATR 비율 계산
- 최초 touch와 상호작용 횟수 기록
- 방향별 fill ratio 계산
- fully rebalanced 및 overrun 기록
- displacement leg, MSS, sweep, OB id 연결
- dealing range normalized position 계산
- HTF/LTF zone 독립 저장

### 코드로 구현 어려운 부분

- 모든 symbol과 timeframe에 공통인 최소 gap size
- 복수 FVG 중 narrative상 핵심 zone 선택
- wick-only touch와 close interaction의 전략별 우위
- broader imbalance와 raw FVG의 완전한 자동 대응
- 모든 setup에 공통인 expiry 시간

어려운 부분은 `research default`, `manual_review_reason`, strategy-specific ranking으로 분리한다.

## 핵심 문장

1. FVG는 연속된 3개 닫힌 봉에서 첫 번째 wick과 세 번째 wick이 겹치지 않는 zone이다.
2. Bullish FVG는 `candle_1.high < candle_3.low`이다.
3. Bearish FVG는 `candle_1.low > candle_3.high`이다.
4. 동일 가격 경계와 wick overlap은 raw FVG가 아니다.
5. FVG는 세 번째 봉 close 이후에만 확정한다.
6. Raw FVG 존재와 qualified trade setup은 다르다.
7. 이상적인 model은 liquidity run, displacement, MSS 문맥 안의 FVG를 찾는다.
8. Zone interaction, fill ratio, strategy invalidation을 분리한다.
9. Rebalance된 zone도 과거 event 이력을 유지한다.
10. FVG, displacement, MSS, OB는 연결되지만 서로 다른 event이다.

## 이 개념의 본질

Fair Value Gap은 같은 symbol과 timeframe의 연속된 3개 닫힌 봉에서 첫 번째 봉과 세 번째 봉 wick의 비중첩 구간을 저장하고, 이후 rebalance 이력과 구조 confluence를 분리해 추적하는 객관적 가격 zone이다.

## Transcript 근거 분류

### 에피소드 전반에 반복되는 핵심 근거

- Episode 3: 3캔들 비중첩 구간 생성 뒤 retracement 관찰
- Episode 5: sudden displacement 이후 FVG 탐색
- Episode 6: bearish 및 bullish FVG의 3캔들 formation과 방향별 zone 경계
- Episode 6: liquidity run 이후 FVG를 찾고, 모든 작은 gap을 무차별 탐색하지 않는 필터
- Episode 6: displacement range 내부 FVG가 없으면 해당 model trade를 배제
- Episode 7: sell-side taken, bullish MSS, FVG retracement 연결
- Episode 12: MSS 이후 imbalance retracement
- Episode 24: sweep, displacement, displacement leg 내부 FVG 흐름
- Episode 40: FVG rebalance와 반복 interaction 사례
- Episode 41: FVG가 있어도 meaningful displacement가 없으면 참여 배제

### 특정 구간에 집중된 보조 근거

- Episode 3: 복수 FVG 중 상하 zone 선택 사례
- Episode 6: 반대 방향 FVG를 profit target으로 사용하는 사례
- Episode 10: daily 및 hourly FVG 문맥
- Episode 38: FVG가 차트 전반에 많으므로 문맥 필터가 필요하다는 설명
- Episode 41: 15분 FVG overrun 뒤 더 큰 zone을 관찰하는 사례

### 검색 분포

정리본 `transcripts/*.en.txt` 전체 검색 기준:

| 표현 | 검색 라인 수 | 등장 에피소드 파일 수 |
| --- | ---: | ---: |
| `fair value gap` | 827 | 39 |
| `imbalance` | 633 | 32 |
| `rebalance` | 168 | 11 |
| `rebalanced` | 45 | 8 |
| `rebalancing` | 36 | 5 |

수치는 자막 중복을 포함하므로 절대 빈도가 아니라 반복 분포 확인용이다.
