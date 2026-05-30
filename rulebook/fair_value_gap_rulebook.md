# Fair Value Gap Rulebook

공통 계약: [README.md](README.md)

## 목적

같은 symbol과 timeframe의 연속된 닫힌 OHLC 3개에서 첫 번째 봉과 세 번째 봉 wick의 비중첩 구간을 탐지하고, 이후 zone interaction과 setup eligibility를 분리해 추적한다.

raw FVG는 OHLC만으로 객관적으로 탐지할 수 있다. liquidity run, displacement, MSS, OB overlap, session은 raw zone의 존재 조건이 아니라 전략 ranking metadata이다.

## 객관적 정의

### Candle Index

```text
candle_1 = bars[i - 2]
candle_2 = bars[i - 1]
candle_3 = bars[i]
```

세 번째 봉 `i`가 닫힌 뒤에만 FVG를 생성한다.

### Bullish FVG

```text
is_bullish_fvg =
  candle_1.high < candle_3.low

bullish_fvg.zone_low  = candle_1.high
bullish_fvg.zone_high = candle_3.low
```

### Bearish FVG

```text
is_bearish_fvg =
  candle_1.low > candle_3.high

bearish_fvg.zone_low  = candle_3.high
bearish_fvg.zone_high = candle_1.low
```

### 공통 계산

```text
gap_size = zone_high - zone_low
midpoint = (zone_high + zone_low) / 2
gap_ticks = gap_size / tick_size if tick_size > 0 else null
gap_atr = gap_size / ATR(14) if ATR(14) > 0 else null
```

동일 가격 경계는 gap이 아니다.

```text
zone_high == zone_low -> no raw FVG
```

### 출력 객체

```text
fvg_zone = {
  id,
  symbol,
  timeframe,
  direction,                 # bullish | bearish
  created_at,
  candle_1_id,
  candle_2_id,
  candle_3_id,
  zone_low,
  zone_high,
  midpoint,
  gap_size,
  gap_ticks,
  gap_atr,
  lifecycle_status,          # candidate | confirmed | invalidated | expired | manual_review
  interaction_status,        # active | touched | partially_rebalanced | fully_rebalanced | overrun
  setup_eligibility,         # unranked | eligible | filtered_out | invalidated
  first_touched_at,
  last_interaction_at,
  interaction_count,
  fill_ratio,
  linked_liquidity_event_ids[],
  linked_sweep_ids[],
  linked_displacement_ids[],
  linked_mss_ids[],
  linked_order_block_ids[],
  dealing_range_id,
  normalized_position,
  session_tags[],
  calendar_event_ids[],
  reason_codes[],
  manual_review_reason[],
  parameters_version
}
```

### Interaction Overlap

생성 이후 봉만 검사한다.

```text
zone_overlap =
  bar.high >= zone_low
  and bar.low <= zone_high
```

최초 overlap:

```text
interaction_status = touched
first_touched_at = bar.closed_at
```

### Fill Ratio

bullish FVG는 위에서 아래로 되돌림되는 fill depth를 계산한다.

```text
bullish_fill_depth =
  clamp(zone_high - minimum_later_low, 0, gap_size)

bullish_fill_ratio =
  bullish_fill_depth / gap_size
```

bearish FVG는 아래에서 위로 되돌림되는 fill depth를 계산한다.

```text
bearish_fill_depth =
  clamp(maximum_later_high - zone_low, 0, gap_size)

bearish_fill_ratio =
  bearish_fill_depth / gap_size
```

```text
0 < fill_ratio < 1 -> partially_rebalanced
fill_ratio == 1    -> fully_rebalanced
```

### Overrun

```text
bullish_overrun =
  later_bar.low < zone_low

bearish_overrun =
  later_bar.high > zone_high
```

overrun은 zone 역사 삭제 조건이 아니다.

### Setup Eligibility

zone 자체 상태와 전략 적격성을 분리한다.

예시 strategy profile:

```text
eligible_retracement_fvg =
  raw FVG confirmed
  and linked same-direction displacement exists
  and optional linked MSS exists
  and optional linked liquidity event exists
  and not strategy_invalidated
```

close-through를 setup invalidation으로 사용할 수 있다.

```text
bullish_setup_invalidated =
  configured_close_through_filter
  and later_bar.close < zone_low

bearish_setup_invalidated =
  configured_close_through_filter
  and later_bar.close > zone_high
```

이 정책은 strategy-specific parameter이다. raw FVG zone을 삭제하지 않는다.

## 필수 조건

- 입력 OHLC에 `timestamp, symbol, timeframe, open, high, low, close`가 있다.
- OHLC는 symbol과 timeframe별로 오름차순 정렬된다.
- 세 봉은 같은 symbol과 timeframe에 속한다.
- 세 봉은 expected interval 기준으로 연속적이다.
- 세 번째 봉이 닫힌 뒤에만 raw FVG를 확정한다.
- bullish FVG는 `candle_1.high < candle_3.low`이다.
- bearish FVG는 `candle_1.low > candle_3.high`이다.
- `gap_size > 0`이다.
- zone low, zone high, candle id 3개를 보존한다.
- interaction은 생성 이후 봉에서만 계산한다.
- raw zone lifecycle, interaction status, setup eligibility를 분리한다.
- linked event는 foreign key와 시간 순서를 보존한다.

## 무효 조건

- 세 번째 봉 close 전에 FVG를 확정한다.
- 미래 봉을 사용해 과거 시점 FVG를 미리 생성한다.
- wick overlap이 있는데 FVG로 저장한다.
- 동일 가격 경계를 양수 gap으로 저장한다.
- 결측 또는 timestamp gap이 있는 세 봉을 정상 formation으로 처리한다.
- 다른 symbol 또는 timeframe 봉을 한 formation에 섞는다.
- raw FVG가 있다는 이유만으로 displacement, MSS, OB, 진입을 확정한다.
- displacement가 없다는 이유로 raw FVG event를 삭제한다.
- full rebalance 또는 overrun 뒤 raw zone 이력을 삭제한다.
- strategy invalidation과 raw zone 존재를 같은 필드로 관리한다.

## 예외 조건

- tick size 결측: `gap_ticks = null`, `manual_review_reason = missing_tick_size`를 기록한다.
- ATR 부족: `gap_atr = null`로 저장한다.
- 데이터 gap: formation 생성을 skip하고 warm-up 정책을 적용한다.
- same-bar interaction: 세 번째 봉은 formation 일부이므로 interaction 계산에서 제외한다.
- 다음 봉 gap open이 zone을 건너뜀: `gap_overrun = true`를 기록한다.
- 복수 FVG 중첩: zone별 event를 유지하고 overlap group id를 추가한다.
- HTF/LTF 중첩: timeframe별 event를 독립 저장하고 parent-child link를 추가한다.
- fully rebalanced zone: 과거 zone을 유지하고 active entry candidate에서는 제외할 수 있다.
- overrun zone: zone 이력을 유지하고 target 또는 historical interaction 분석에 사용할 수 있다.
- broader imbalance 표현: raw 3캔들 FVG로 자동 생성하지 않는다.
- displacement 없는 FVG: raw zone으로 허용하고 setup ranking에서 분리한다.
- MSS 없는 FVG: raw zone으로 허용하고 setup ranking에서 분리한다.

## 상태 변화(State Machine)

zone lifecycle, interaction, setup eligibility를 분리한다.

```text
lifecycle:
candidate -> confirmed | invalidated | expired | manual_review

interaction:
active -> touched -> partially_rebalanced -> fully_rebalanced
active | touched | partially_rebalanced -> overrun

setup eligibility:
unranked -> eligible | filtered_out | invalidated
```

### candidate

두 번째 봉 close 시점에는 formation 후보를 선택적으로 만들 수 있다. 세 번째 봉이 닫히기 전에는 confirmed FVG가 아니다.

```text
candidate_storage = optional
```

최소 scanner는 candidate 저장을 생략한다.

### confirmed

세 번째 봉 close 시점에 wick 비중첩 조건을 만족하면 confirmed이다.

```text
candle_1.high < candle_3.low -> confirmed bullish FVG
candle_1.low > candle_3.high -> confirmed bearish FVG
```

### invalidated

raw FVG lifecycle invalidated는 다음 데이터 오류에 사용한다.

- formation 봉의 OHLC 정정으로 비중첩 조건이 사라졌다.
- symbol 또는 timeframe 혼합이 발견되었다.
- timestamp gap이 발견되었다.
- source data version이 바뀌었다.

가격이 zone을 통과한 경우에는 raw lifecycle을 invalidated로 바꾸지 않고 interaction status와 strategy eligibility를 갱신한다.

### expired

zone 자체는 고정 만료 시간을 요구하지 않는다. 전략이 entry candidate expiry를 사용할 수 있다.

```text
setup_expiry_bars = configurable per timeframe
```

configured 기간 안에 retracement가 없으면:

```text
setup_eligibility = filtered_out
reason = setup_expired
```

raw zone은 유지한다.

### manual_review

다음 중 하나이면 자동 결과와 함께 검토 사유를 남긴다.

- 복수 중첩 FVG 중 primary setup zone을 선택해야 한다.
- broader imbalance를 별도 zone으로 저장할지 결정해야 한다.
- gap overrun을 일반 interaction과 분리할지 결정해야 한다.
- HTF와 LTF FVG 방향이 충돌한다.
- 전략별 wick touch와 close interaction 우선순위를 결정해야 한다.

## 코드 구현 규칙

### 입력 데이터

```text
required:
  closed_ohlc[]

optional:
  tick_size
  atr_14[]
  liquidity_events[]
  sweep_events[]
  displacement_events[]
  mss_events[]
  order_block_events[]
  dealing_ranges[]
  session_metadata
  calendar_metadata
  setup_parameters
```

### 필요 변수

```text
fvg_id
symbol
timeframe
direction
created_at
candle_1_id
candle_2_id
candle_3_id
zone_low
zone_high
midpoint
gap_size
gap_ticks
gap_atr
lifecycle_status
interaction_status
setup_eligibility
first_touched_at
last_interaction_at
interaction_count
fill_depth
fill_ratio
gap_overrun
linked_liquidity_event_ids[]
linked_sweep_ids[]
linked_displacement_ids[]
linked_mss_ids[]
linked_order_block_ids[]
dealing_range_id
normalized_position
session_tags[]
calendar_event_ids[]
reason_codes[]
manual_review_reason[]
parameters_version
```

### 조건식

```text
is_bullish_fvg =
  candle_1.high < candle_3.low

is_bearish_fvg =
  candle_1.low > candle_3.high

zone_overlap =
  later_bar.high >= zone_low
  and later_bar.low <= zone_high

bullish_fill_ratio =
  clamp(zone_high - minimum_later_low, 0, gap_size) / gap_size

bearish_fill_ratio =
  clamp(maximum_later_high - zone_low, 0, gap_size) / gap_size
```

### 의사코드

```text
for each symbol, timeframe:
  sort closed bars by timestamp

  for i from 2 to last_closed_bar:
    c1 = bars[i - 2]
    c2 = bars[i - 1]
    c3 = bars[i]

    if not consecutive(c1, c2, c3):
      continue

    if c1.high < c3.low:
      emit confirmed bullish FVG(
        zone_low = c1.high,
        zone_high = c3.low,
        created_at = c3.closed_at
      )

    if c1.low > c3.high:
      emit confirmed bearish FVG(
        zone_low = c3.high,
        zone_high = c1.low,
        created_at = c3.closed_at
      )

for each confirmed FVG:
  for each later closed bar:
    if bar overlaps zone:
      update first touch, interaction count, fill ratio

    if fill_ratio == 1:
      set interaction_status = fully_rebalanced

    if bar trades beyond opposite boundary:
      set interaction_status = overrun

  link optional displacement, MSS, liquidity, OB, range metadata
  evaluate strategy eligibility separately
```

## 최소 구현 버전

닫힌 OHLC만 사용한다.

- 같은 symbol과 timeframe의 연속된 3개 봉만 검사한다.
- bullish와 bearish FVG를 방향별 수식으로 탐지한다.
- 세 번째 봉 close 이후 raw zone을 확정한다.
- zone low, high, midpoint, gap size를 저장한다.
- 이후 봉의 overlap, 최초 touch, fill ratio, full rebalance, overrun을 기록한다.
- raw zone 이력과 setup eligibility를 분리한다.
- FVG를 entry signal로 변환하지 않는다.

최소 출력:

```text
id, symbol, timeframe, direction, created_at,
candle_1_id, candle_2_id, candle_3_id,
zone_low, zone_high, midpoint, gap_size,
lifecycle_status, interaction_status,
first_touched_at, interaction_count, fill_ratio,
reason_codes, parameters_version
```

## 고급 구현 버전

### HTF

- timeframe별 FVG를 독립 계산한다.
- LTF zone에 겹치는 HTF FVG id를 parent link로 추가한다.
- HTF FVG는 세 번째 HTF 봉 close 이후에만 LTF에서 사용할 수 있다.
- HTF/LTF 방향 충돌을 metadata로 남긴다.

### 세션

- raw FVG 정의 자체는 session으로 제한하지 않는다.
- London, New York, lunch hour, afternoon tag를 추가한다.
- `America/New_York` timezone과 DST-aware calendar를 사용한다.
- session별 formation 및 retracement 성과를 별도 cohort로 분석한다.

### 유동성

- preceding BSL/SSL taken 또는 confirmed sweep id를 연결한다.
- liquidity event부터 FVG 생성까지 elapsed bars를 저장한다.
- target으로 사용한 old FVG와 entry retracement FVG를 role metadata로 구분한다.

### Displacement와 MSS

- displacement leg 내부 same-direction FVG를 link한다.
- MSS 이전, MSS 포함 leg, MSS 이후 FVG를 시간순으로 구분한다.
- raw zone을 삭제하지 않고 setup ranking을 조정한다.
- Episode 6 강화 profile은 liquidity run, displacement, MSS, same-leg FVG를 요구할 수 있다.

### Order Block

- FVG와 겹치는 OB zone id를 연결한다.
- overlap 가격 범위와 overlap ratio를 계산한다.
- FVG와 OB를 동일 zone으로 합치지 않는다.

### SMT

- correlated symbol별 comparable FVG formation과 liquidity event를 정렬한다.
- FVG 모듈 자체는 SMT divergence를 확정하지 않는다.

### 경제지표

- configured event window 안 FVG에 calendar event id를 붙인다.
- news spike를 삭제하지 않고 별도 cohort로 분석한다.

## False Signal 제거 방법

- 세 번째 봉 close 이후에만 FVG를 확정한다.
- wick overlap과 동일 가격 경계를 제외한다.
- timestamp gap이 있는 formation을 제외한다.
- raw FVG와 qualified setup을 구분한다.
- displacement, MSS, liquidity link를 별도 metadata로 저장한다.
- full rebalance, overrun, setup invalidation을 구분한다.
- 복수 FVG를 하나로 합치지 않는다.
- HTF/LTF zone을 중복 수익으로 계산하지 않는다.
- broader imbalance 표현을 raw FVG로 자동 변환하지 않는다.
- strategy expiry를 raw zone 삭제로 구현하지 않는다.

## Pine Script 구현 가이드

Pine Script에서는 닫힌 세 번째 봉에서 zone을 생성한다.

```text
is_bullish_fvg = high[2] < low
is_bearish_fvg = low[2] > high

if barstate.isconfirmed and is_bullish_fvg:
  create bullish zone(low = high[2], high = low)

if barstate.isconfirmed and is_bearish_fvg:
  create bearish zone(low = high, high = low[2])
```

- box에는 zone id, 방향, 생성 시각을 연결한다.
- active, touched, partially rebalanced, fully rebalanced, overrun을 색상으로 구분한다.
- 시각화 객체 수 제한 때문에 오래된 box를 정리해도 event count는 유지한다.
- alert payload에 `fvg_id`, direction, zone boundaries, interaction status를 넣는다.
- displacement, MSS, liquidity confluence는 label 또는 alert metadata로 표시한다.

## Python Scanner 구현 가이드

event table과 interaction table을 분리한다.

```text
fvg_zones
fvg_interactions
fvg_setup_eligibility
fvg_liquidity_links
fvg_displacement_links
fvg_mss_links
fvg_order_block_links
fvg_context_metadata
```

- `groupby(symbol, timeframe)` 안에서 `high.shift(2) < low`와 `low.shift(2) > high`를 계산한다.
- expected timestamp interval을 검증한다.
- 세 번째 행 timestamp를 `created_at`으로 저장한다.
- interaction은 `bar.timestamp > fvg.created_at` 조건을 지킨다.
- 방향별 cumulative minimum low 또는 maximum high로 fill ratio를 계산한다.
- raw zone과 setup eligibility를 다른 table 또는 field로 유지한다.
- displacement leg join은 생성 봉 id와 leg 범위를 사용한다.
- MSS, liquidity, OB, HTF, session, calendar metadata는 후속 join으로 연결한다.

## 백테스트 시 주의사항

- 세 번째 봉 close 이전에 FVG를 알고 있었다고 가정하지 않는다.
- 세 번째 봉 자체를 retracement interaction으로 계산하지 않는다.
- wick touch, close interaction, full rebalance, overrun을 분리한다.
- raw FVG 전체와 strengthened setup FVG 성과를 분리한다.
- minimum gap tick 또는 ATR filter는 자막 고정값이 아니라 research parameter이다.
- close-through setup invalidation 정책을 parameters version에 기록한다.
- expiry 정책을 바꾸면 setup 결과도 달라진다.
- 복수 FVG 선택 ranking을 versioning한다.
- HTF FVG는 HTF 세 번째 봉 close 이후에만 LTF에서 사용할 수 있다.
- news spike 포함 여부를 별도 cohort로 비교한다.
- vendor별 timestamp, session, DST, missing bar 정책을 기록한다.

## 최종 Rulebook 정의

Fair Value Gap은 같은 symbol과 timeframe의 연속된 닫힌 OHLC 3개에서 첫 번째 봉과 세 번째 봉 wick이 strict non-overlap일 때 세 번째 봉 close 이후 생성되는 객관적 가격 zone이다. bullish FVG는 `candle_1.high < candle_3.low`, bearish FVG는 `candle_1.low > candle_3.high`로 판정한다. zone의 interaction history, fill ratio, full rebalance, overrun을 추적하되 raw zone 이력과 setup eligibility를 분리한다. liquidity run, displacement, MSS, OB overlap은 raw zone 존재 조건이 아니라 versioned 전략 ranking metadata이다.
