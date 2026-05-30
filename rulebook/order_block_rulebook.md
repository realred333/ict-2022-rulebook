# Order Block Rulebook

공통 계약: [README.md](README.md)

## 목적

같은 symbol과 timeframe의 닫힌 OHLC에서 연속 directional close series를 만들고, series 시작 open의 반대 방향 strict traversal을 state of delivery 변화 프록시로 기록한다. 이후 opening level, body zone, wick zone interaction을 분리해 추적한다.

Episode 3과 Episode 12는 OB가 단순히 마지막 반대색 candle 하나가 아니라고 설명한다. 최소 구현은 single candle과 consecutive series를 모두 같은 객체로 처리한다.

## 객관적 정의

### Directional Candle

```text
is_down_close = close < open
is_up_close   = close > open
is_doji       = close == open
```

### Delivery Series

```text
bearish_delivery_series =
  one or more contiguous down-close candles

bullish_delivery_series =
  one or more contiguous up-close candles
```

기본 series 정책:

```text
doji_series_policy = terminate
opposite_close_policy = terminate
```

doji 처리 정책은 자막 고정값이 아니라 versioned `research default`이다.

### Series 계산

```text
delivery_open = series.first.open

wick_zone_low  = min(series.low)
wick_zone_high = max(series.high)

body_zone_low  = min(min(bar.open, bar.close) for bar in series)
body_zone_high = max(max(bar.open, bar.close) for bar in series)

series_length = count(series.bars)
```

### Bullish Raw OB

down-close series가 downside delivery를 제공한 뒤 시작 open을 위로 관통하면 bullish raw OB이다.

```text
is_bullish_raw_ob =
  source_series.direction == bearish_delivery
  and later_bar.high > source_series.delivery_open
```

### Bearish Raw OB

up-close series가 upside delivery를 제공한 뒤 시작 open을 아래로 관통하면 bearish raw OB이다.

```text
is_bearish_raw_ob =
  source_series.direction == bullish_delivery
  and later_bar.low < source_series.delivery_open
```

동일 가격 touch는 confirmation이 아니다.

### 출력 객체

```text
order_block = {
  id,
  symbol,
  timeframe,
  direction,                    # bullish | bearish
  created_at,
  confirmed_at,
  source_series_id,
  source_delivery_direction,    # bearish_delivery | bullish_delivery
  series_start_at,
  series_end_at,
  series_bar_ids[],
  series_length,
  delivery_open,
  body_zone_low,
  body_zone_high,
  wick_zone_low,
  wick_zone_high,
  lifecycle_status,             # candidate | confirmed | invalidated | expired | manual_review
  interaction_status,           # active | touched | mitigated | overrun
  setup_eligibility,            # unranked | eligible | filtered_out | invalidated
  first_touched_at,
  interaction_count,
  selected_refinement,          # opening_level | body_zone | wick_zone | lower_timeframe
  linked_liquidity_event_ids[],
  linked_sweep_ids[],
  linked_displacement_ids[],
  linked_mss_ids[],
  linked_fvg_ids[],
  dealing_range_id,
  normalized_position,
  session_tags[],
  calendar_event_ids[],
  reason_codes[],
  manual_review_reason[],
  parameters_version
}
```

### Refinement Range

opening level, body zone, wick zone을 분리한다.

```text
opening_level = delivery_open
body_zone     = [body_zone_low, body_zone_high]
wick_zone     = [wick_zone_low, wick_zone_high]
```

기본 저장은 세 가지를 모두 포함한다. strategy가 사용할 refinement를 선택한다.

```text
ob_refinement = opening_level | body_zone | wick_zone | lower_timeframe
```

### Interaction

confirmation 이후 봉만 retracement interaction으로 검사한다.

```text
wick_zone_overlap =
  later_bar.high >= wick_zone_low
  and later_bar.low <= wick_zone_high

body_zone_overlap =
  later_bar.high >= body_zone_low
  and later_bar.low <= body_zone_high

opening_level_touch =
  later_bar.low <= delivery_open <= later_bar.high
```

### Overrun과 Setup Invalidation

```text
bullish_ob_overrun =
  later_bar.low < wick_zone_low

bearish_ob_overrun =
  later_bar.high > wick_zone_high
```

close-through를 strategy invalidation으로 사용할 수 있다.

```text
bullish_setup_invalidated =
  configured_close_through_filter
  and later_bar.close < wick_zone_low

bearish_setup_invalidated =
  configured_close_through_filter
  and later_bar.close > wick_zone_high
```

overrun 또는 setup invalidation 뒤에도 raw OB 이력을 삭제하지 않는다. downstream [Breaker Block Rulebook](breaker_block_rulebook.md)에 전달한다.

### High-Probability Context

Episode 3의 gap, taken liquidity, MSS 결합을 optional strategy profile로 구현한다.

```text
high_probability_ob_profile =
  raw OB confirmed
  and linked FVG exists
  and linked liquidity taken event exists
  and linked MSS exists
```

displacement, dealing range 위치, HTF, session은 추가 ranking metadata이다.

## 필수 조건

- 입력 OHLC에 `timestamp, symbol, timeframe, open, high, low, close`가 있다.
- OHLC는 symbol과 timeframe별로 오름차순 정렬된다.
- series 구성 봉은 같은 symbol과 timeframe에 속한다.
- series 구성 봉은 expected timestamp interval 기준으로 연속적이다.
- bullish source series는 하나 이상의 contiguous down-close candle로 구성된다.
- bearish source series는 하나 이상의 contiguous up-close candle로 구성된다.
- `delivery_open = series.first.open`이다.
- bullish confirmation은 later `high > delivery_open`이다.
- bearish confirmation은 later `low < delivery_open`이다.
- 동일 가격 touch는 confirmation이 아니다.
- confirmation 봉 close 이후에만 raw OB를 확정한다.
- source series id와 구성 bar id를 보존한다.
- delivery open, body zone, wick zone을 분리해 저장한다.
- raw lifecycle, interaction status, setup eligibility를 분리한다.

## 무효 조건

- 모든 down-close candle을 자동 bullish OB로 확정한다.
- 모든 up-close candle을 자동 bearish OB로 확정한다.
- consecutive series를 마지막 반대색 봉 하나로 강제 축약한다.
- series 시작 open이 아니라 임의 candle open을 canonical delivery open으로 사용한다.
- 동일 가격 touch를 strict traversal로 처리한다.
- 미래 봉을 읽어 과거 시점에 OB를 미리 확정한다.
- 다른 symbol 또는 timeframe 봉을 한 series에 섞는다.
- 결측 또는 timestamp gap을 정상 contiguous series로 처리한다.
- FVG, MSS, displacement 존재만으로 raw OB를 확정한다.
- zone overrun 뒤 raw OB 이력을 삭제한다.
- opening level, body zone, wick zone을 구분하지 않는다.

## 예외 조건

- single candle series: 길이 `1`인 정상 series로 허용한다.
- doji: 기본 정책에서는 series를 종료한다. 다른 정책은 parameters version을 바꾼다.
- gap traversal: bar open이 delivery open 반대편에서 시작하면 `gap_traversal = true`를 기록한다.
- confirmation bar가 source series zone과 겹침: confirmation과 retracement interaction을 같은 사건으로 중복 계산하지 않는다.
- 복수 series 동시 confirmation: series별 raw OB를 저장하고 overlap group id를 추가한다.
- 중첩 OB: timeframe과 source series별 zone을 유지한다.
- FVG 없음: raw OB를 허용하고 setup ranking에서 분리한다.
- MSS 없음: raw OB를 허용하고 setup ranking에서 분리한다.
- liquidity event 없음: raw OB를 허용하고 setup ranking에서 분리한다.
- body와 wick 범위 차이가 큼: 세 범위를 모두 저장하고 strategy refinement를 선택한다.
- overrun OB: 이력을 유지하고 breaker candidate input으로 전달한다.
- HTF/LTF 중첩: timeframe별 OB를 독립 저장하고 parent-child link를 추가한다.

## 상태 변화(State Machine)

raw lifecycle, interaction, setup eligibility를 분리한다.

```text
lifecycle:
candidate -> confirmed | invalidated | expired | manual_review

interaction:
active -> touched -> mitigated | overrun

setup eligibility:
unranked -> eligible | filtered_out | invalidated
```

### candidate

contiguous directional series가 열리면 candidate를 생성하거나 갱신한다.

```text
down-close series -> bullish OB candidate
up-close series   -> bearish OB candidate
```

series가 이어지는 동안 bar id와 zone 범위를 갱신한다.

### confirmed

series 종료 이후 또는 configured confirmation window 안에서 delivery open 반대 방향 strict traversal이 발생하면 confirmed이다.

```text
down-close series + high > delivery_open -> confirmed bullish raw OB
up-close series   + low  < delivery_open -> confirmed bearish raw OB
```

```text
ob_confirmation_window_bars = configurable  # research default
```

자막은 모든 timeframe에 공통인 고정 window를 제시하지 않는다.

### invalidated

raw lifecycle invalidated는 다음 데이터 오류에 사용한다.

- source series OHLC 정정으로 방향 조건이 사라졌다.
- source series timestamp gap이 발견되었다.
- symbol 또는 timeframe 혼합이 발견되었다.
- source data version이 바뀌었다.

가격이 zone을 overrun한 경우 raw lifecycle을 invalidated로 덮어쓰지 않는다. interaction과 strategy eligibility를 갱신한다.

### expired

configured window 안에 delivery open 반대 방향 traversal이 없으면 candidate를 expired로 기록한다.

```text
bars_since_series_end > ob_confirmation_window_bars
and no strict traversal
-> lifecycle_status = expired
```

### manual_review

다음 중 하나이면 자동 결과와 함께 검토 사유를 남긴다.

- doji 포함 여부가 series 결과를 바꾼다.
- 복수 중첩 OB 중 primary setup zone을 선택해야 한다.
- body zone과 wick zone 차이가 커 refinement 선택이 필요하다.
- HTF와 LTF OB 방향이 충돌한다.
- aggressive lower-timeframe refinement를 적용할지 결정해야 한다.

## 코드 구현 규칙

### 입력 데이터

```text
required:
  closed_ohlc[]
  ob_confirmation_window_bars

optional:
  tick_size
  atr_14[]
  liquidity_events[]
  sweep_events[]
  displacement_events[]
  mss_events[]
  fvg_zones[]
  dealing_ranges[]
  session_metadata
  calendar_metadata
  refinement_parameters
```

### 필요 변수

```text
ob_id
series_id
symbol
timeframe
direction
source_delivery_direction
series_start_at
series_end_at
series_bar_ids[]
series_length
delivery_open
body_zone_low
body_zone_high
wick_zone_low
wick_zone_high
confirmed_at
confirmation_bar_id
gap_traversal
lifecycle_status
interaction_status
setup_eligibility
selected_refinement
first_touched_at
interaction_count
linked_liquidity_event_ids[]
linked_sweep_ids[]
linked_displacement_ids[]
linked_mss_ids[]
linked_fvg_ids[]
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
is_down_close = close < open
is_up_close = close > open

confirm_bullish_raw_ob =
  source_series.direction == bearish_delivery
  and later_bar.high > source_series.delivery_open

confirm_bearish_raw_ob =
  source_series.direction == bullish_delivery
  and later_bar.low < source_series.delivery_open

wick_zone_overlap =
  later_bar.high >= wick_zone_low
  and later_bar.low <= wick_zone_high

body_zone_overlap =
  later_bar.high >= body_zone_low
  and later_bar.low <= body_zone_high
```

### 의사코드

```text
for each symbol, timeframe:
  sort closed bars by timestamp
  active_series = null
  pending_series = []

  for bar in closed_bars:
    if timestamp gap:
      expire or close active_series
      apply warm-up policy
      continue

    candle_direction =
      down_close if close < open
      up_close   if close > open
      doji       otherwise

    if candle_direction continues active_series:
      append bar and update zones
    else:
      close active_series into pending_series
      start new series if candle_direction is not doji

    for series in pending_series:
      if bars_since(series.end_at) > confirmation_window:
        expire series
        continue

      if series is down-close and bar.high > series.delivery_open:
        emit confirmed bullish raw OB

      if series is up-close and bar.low < series.delivery_open:
        emit confirmed bearish raw OB

for each confirmed raw OB:
  track later opening-level, body-zone, wick-zone interactions
  record overrun without deleting raw history
  link optional FVG, liquidity, MSS, displacement, range metadata
  evaluate setup eligibility separately
```

confirmation bar가 새 반대 방향 series 일부가 될 수 있다. pending series confirmation 평가와 신규 series 갱신 순서를 parameters version에 고정하고 테스트한다.

## 최소 구현 버전

닫힌 OHLC만 사용한다.

- same symbol과 timeframe의 contiguous up-close 및 down-close series를 만든다.
- doji는 기본적으로 series를 종료한다.
- series 첫 open을 delivery open으로 저장한다.
- down-close series 뒤 `high > delivery_open`이면 bullish raw OB를 확정한다.
- up-close series 뒤 `low < delivery_open`이면 bearish raw OB를 확정한다.
- delivery open, body zone, wick zone을 모두 저장한다.
- 이후 zone touch와 overrun을 기록한다.
- raw OB와 setup eligibility를 분리한다.
- OB를 entry signal로 변환하지 않는다.

최소 출력:

```text
id, symbol, timeframe, direction,
source_series_id, series_start_at, series_end_at,
series_length, delivery_open,
body_zone_low, body_zone_high,
wick_zone_low, wick_zone_high,
confirmed_at, lifecycle_status, interaction_status,
first_touched_at, interaction_count,
reason_codes, parameters_version
```

## 고급 구현 버전

### HTF

- timeframe별 OB series와 zone을 독립 계산한다.
- LTF OB에 겹치는 HTF OB id를 parent link로 추가한다.
- HTF OB는 confirmation HTF 봉 close 이후에만 LTF에서 사용할 수 있다.
- HTF zone 내부 LTF FVG 또는 OB refinement를 별도 객체로 저장한다.

### 세션

- raw OB 정의 자체는 session으로 제한하지 않는다.
- London, New York, lunch hour, afternoon tag를 추가한다.
- `America/New_York` timezone과 DST-aware calendar를 사용한다.
- session별 interaction 성과를 별도 cohort로 분석한다.

### 유동성

- preceding BSL/SSL taken 또는 confirmed sweep id를 연결한다.
- liquidity event부터 OB confirmation까지 elapsed bars를 저장한다.
- target liquidity id를 setup metadata로 추가한다.

### Displacement와 MSS

- same-direction displacement 및 MSS id를 연결한다.
- raw OB를 삭제하지 않고 setup ranking을 조정한다.
- Episode 3 high-probability profile은 gap, taken liquidity, MSS를 요구할 수 있다.

### FVG

- OB opening level, body zone, wick zone과 FVG overlap을 각각 계산한다.
- overlap 가격 범위와 overlap ratio를 저장한다.
- lower-timeframe imbalance를 aggressive refinement metadata로 연결한다.
- OB와 FVG를 동일 zone으로 합치지 않는다.

### Refinement

```text
ob_refinement =
  opening_level
  | body_zone
  | wick_zone
  | lower_timeframe
```

- Episode 3 opening price reference를 `opening_level` profile로 구현한다.
- Episode 17 body preference를 `body_zone` profile로 구현한다.
- 넓은 interaction 분석에는 `wick_zone`을 사용한다.
- refinement가 바뀌면 parameters version도 바꾼다.

### SMT

- correlated symbol별 comparable liquidity event, MSS, OB interaction을 정렬한다.
- OB 모듈 자체는 SMT divergence를 확정하지 않는다.

### 경제지표

- configured event window 안 OB confirmation과 retracement에 calendar event id를 붙인다.
- news spike를 삭제하지 않고 별도 cohort로 분석한다.

## False Signal 제거 방법

- 모든 반대색 candle을 즉시 OB로 확정하지 않는다.
- consecutive series와 single candle series를 같은 모델로 처리한다.
- series 첫 open과 반대 방향 strict traversal을 요구한다.
- 동일 가격 touch를 confirmation으로 처리하지 않는다.
- source series timestamp 연속성을 검증한다.
- raw OB와 high-probability context를 분리한다.
- opening level, body zone, wick zone을 분리한다.
- FVG, liquidity, MSS, displacement를 link metadata로 저장한다.
- overrun raw OB 이력을 삭제하지 않는다.
- HTF/LTF zone을 중복 수익으로 계산하지 않는다.

## Pine Script 구현 가이드

Pine Script에서는 active series, pending series, confirmed OB 배열을 분리한다.

```text
is_down_close = close < open
is_up_close = close > open

on confirmed bar:
  update contiguous directional series

  for pending series:
    if down-close series and high > delivery_open:
      confirm bullish raw OB

    if up-close series and low < delivery_open:
      confirm bearish raw OB
```

- `barstate.isconfirmed`에서 raw OB를 확정한다.
- opening level은 line, body zone과 wick zone은 box로 분리 표시한다.
- single candle과 multi-candle series label을 구분한다.
- active, touched, overrun을 색상으로 구분한다.
- alert payload에 `ob_id`, direction, delivery open, zone boundaries, series length를 넣는다.
- 시각화 객체 수 제한 때문에 오래된 line과 box를 정리해도 event count는 유지한다.

## Python Scanner 구현 가이드

series, event, interaction, context table을 분리한다.

```text
ob_delivery_series
order_blocks
ob_interactions
ob_setup_eligibility
ob_fvg_links
ob_liquidity_links
ob_displacement_links
ob_mss_links
ob_context_metadata
```

- `groupby(symbol, timeframe)` 안에서 timestamp 순서를 보장한다.
- run-length encoding으로 contiguous down-close 및 up-close series를 만든다.
- doji series policy를 parameters version에 기록한다.
- series별 첫 open, body zone, wick zone을 계산한다.
- pending series마다 configured confirmation window 안 strict traversal을 평가한다.
- confirmation 이후 row만 retracement interaction으로 계산한다.
- FVG overlap은 opening level, body zone, wick zone별로 분리한다.
- liquidity, MSS, displacement, HTF, session, calendar metadata는 후속 join으로 연결한다.
- overrun OB를 breaker input table에 전달한다.

## 백테스트 시 주의사항

- confirmation 봉 close 이전에 raw OB를 알고 있었다고 가정하지 않는다.
- source series가 완전히 형성되기 전에 미래 길이를 알고 있었다고 가정하지 않는다.
- doji series 정책과 confirmation window를 versioning한다.
- single candle과 multi-candle series 성과를 분리한다.
- opening level, body zone, wick zone refinement 성과를 분리한다.
- wick touch, close interaction, overrun을 분리한다.
- gap, liquidity taken, MSS, displacement confluence 유무를 별도 cohort로 분석한다.
- HTF OB는 HTF confirmation 봉 close 이후에만 LTF에서 사용할 수 있다.
- overlapping OB를 독립 거래처럼 중복 집계하지 않는다.
- news event 포함 여부를 별도 cohort로 비교한다.
- vendor별 timestamp, session, DST, missing bar 정책을 기록한다.

## 최종 Rulebook 정의

Order Block은 같은 symbol과 timeframe의 연속 directional close series에서 시작 open을 `delivery_open`으로 저장하고, 이후 가격이 해당 level을 반대 방향으로 strict traversal할 때 state of delivery 변화 프록시로 확정되는 가격 행동 객체이다. bullish raw OB는 down-close series 뒤 `high > delivery_open`, bearish raw OB는 up-close series 뒤 `low < delivery_open`으로 판정한다. single candle은 길이 `1`인 series이며 consecutive series도 하나의 OB가 될 수 있다. opening level, body zone, wick zone, interaction history, setup eligibility를 분리하고, FVG, liquidity taken, MSS, displacement는 versioned ranking metadata로 연결한다.
