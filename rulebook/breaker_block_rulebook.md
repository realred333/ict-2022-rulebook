# Breaker Block Rulebook

공통 계약: [README.md](README.md)

## 목적

Breaker Block을 일반적인 support/resistance flip 또는 모든 invalidated OB와 분리한다. 같은 symbol과 timeframe에서 liquidity taken event, 반대 방향 short-term swing strict traversal, source candle zone, 이후 retest를 재현 가능한 객체로 저장한다.

## 객관적 정의

### Candle Direction

```text
is_down_close = close < open
is_up_close   = close > open
```

### Strict Traversal

```text
strict_above(level, bar) = bar.high > level
strict_below(level, bar) = bar.low < level
```

동일 가격 touch는 traversal이 아니다. 자막은 `trades back down below`를 설명하므로 최소 raw 규칙은 종가 이탈을 강제하지 않는다. close-through는 strategy filter로 별도 저장한다.

### Bearish Breaker

```text
bearish_breaker_confirmed =
  linked_bsl_taken exists
  and strict_below(break_reference_short_term_low.level, later_bar)
  and bearish_source_candle exists
```

```text
bearish_source_candle =
  lowest down-close candle
  in configured source window
  before the upward move that clears the linked short-term high
```

### Bullish Breaker

```text
bullish_breaker_confirmed =
  linked_ssl_taken exists
  and strict_above(break_reference_short_term_high.level, later_bar)
  and bullish_source_candle exists
```

```text
bullish_source_candle =
  highest up-close candle
  in configured source window
  before the downward move that clears the linked short-term low
```

bullish 규칙은 bearish 자막 정의를 반전한 구현 대칭 규칙이다.

### Source Candle Selection

`lowest`와 `highest`는 source window 내부 candle low와 high를 사용한다.

```text
bearish_source_candle = argmin(bar.low for down-close bar in source_window)
bullish_source_candle = argmax(bar.high for up-close bar in source_window)
```

동률이면 모두 보존하고 `manual_review`를 추가한다. source window 선택은 versioned `research default`이다.

### Zone

```text
wick_zone_low  = source_candle.low
wick_zone_high = source_candle.high

body_zone_low  = min(source_candle.open, source_candle.close)
body_zone_high = max(source_candle.open, source_candle.close)

midpoint = (wick_zone_low + wick_zone_high) / 2

lower_half = [wick_zone_low, midpoint]
upper_half = [midpoint, wick_zone_high]
```

Episode 10의 bearish 사례는 lower half를 refinement로 사용한다. 최소 구현은 wick zone, body zone, midpoint, half-zone을 모두 저장한다.

### OB Link

```text
source_ob_id = matching order block id or null
```

Order Block Rulebook의 overrun history는 breaker 후보 생성에 사용할 수 있다. 그러나 raw breaker confirmation에 `source_ob_id`를 강제하지 않는다.

### 출력 객체

```text
breaker_block = {
  id,
  symbol,
  timeframe,
  direction,                       # bullish | bearish
  created_at,
  confirmed_at,
  source_candle_id,
  source_candle_selection,         # lowest_down_close | highest_up_close
  source_window_start_at,
  source_window_end_at,
  wick_zone_low,
  wick_zone_high,
  body_zone_low,
  body_zone_high,
  midpoint,
  lower_half_low,
  lower_half_high,
  upper_half_low,
  upper_half_high,
  linked_liquidity_event_id,
  linked_bsl_taken_id,
  linked_ssl_taken_id,
  liquidity_source_swing_id,
  break_reference_swing_id,
  reversal_traversal_bar_id,
  source_ob_id,                    # nullable
  symmetry_inference,
  close_through_confirmed,
  lifecycle_status,                # candidate | confirmed | invalidated | expired | manual_review
  interaction_status,              # active | touched | rejected | overrun
  setup_eligibility,               # unranked | eligible | filtered_out | manual_review
  first_touched_at,
  interaction_count,
  linked_sweep_ids[],
  linked_displacement_ids[],
  linked_mss_ids[],
  linked_fvg_ids[],
  target_liquidity_ids[],
  dealing_range_id,
  session_tags[],
  calendar_event_ids[],
  reason_codes[],
  manual_review_reason[],
  parameters_version
}
```

## 필수 조건

- 입력 OHLC에 `timestamp, symbol, timeframe, open, high, low, close`가 있다.
- OHLC는 symbol과 timeframe별 오름차순으로 정렬된다.
- liquidity source는 사용 시점에 확정된 swing에서 생성된 BSL 또는 SSL이다.
- liquidity event는 동일 가격 touch가 아니라 strict traversal로 `taken` 상태가 된다.
- bearish breaker는 linked BSL taken event가 있다.
- bearish breaker는 이후 확정 short-term low 아래 strict traversal이 있다.
- bearish source는 linked high를 제거한 상승 이동 직전 source window의 lowest down-close candle이다.
- bullish breaker는 linked SSL taken event가 있다.
- bullish breaker는 이후 확정 short-term high 위 strict traversal이 있다.
- bullish source는 linked low를 제거한 하락 이동 직전 source window의 highest up-close candle이다.
- source candle, liquidity source swing, break reference swing, traversal bar id를 보존한다.
- zone 생성 시점과 이후 retest interaction을 분리한다.
- 방향 대칭 규칙으로 생성한 bullish breaker에는 `symmetry_inference = true`를 저장한다.

## 무효 조건

- liquidity taken event 없이 일반 flip을 raw breaker로 확정한다.
- 반대편 short-term swing strict traversal 없이 raw breaker를 확정한다.
- 동일 가격 touch를 strict traversal로 처리한다.
- source candle을 찾지 못했는데 raw breaker를 확정한다.
- 모든 invalidated OB를 자동으로 breaker로 승격한다.
- `source_ob_id`가 없다는 이유만으로 유효한 breaker pattern을 삭제한다.
- 미래 봉을 읽어 과거 시점에 breaker를 미리 확정한다.
- 다른 symbol 또는 timeframe의 liquidity event와 swing을 섞는다.
- source zone과 retest interaction을 하나의 event로 중복 계산한다.

## 예외 조건

- source window 안 source candle 후보가 여러 개로 동률이면 모두 저장하고 `manual_review`를 추가한다.
- source window에 expected 방향 candle이 없으면 candidate를 `manual_review` 또는 `expired`로 보낸다.
- reversal traversal bar가 source zone도 동시에 통과하면 confirmation과 later retest로 중복 계산하지 않는다.
- source OB가 있으면 `source_ob_id`를 연결하고 없으면 `null`을 허용한다.
- OB overrun input만 있고 liquidity taken 또는 reversal traversal이 없으면 breaker로 확정하지 않는다.
- gap으로 reversal level 반대편에서 시작하면 `gap_traversal = true`를 저장한다.
- raw breaker가 zone을 재방문하지 않아도 confirmed history를 유지한다.
- narrative를 자동 확정할 수 없으면 raw breaker를 보존하고 setup eligibility를 `manual_review`로 둔다.

## 상태 변화(State Machine)

raw lifecycle, zone interaction, setup eligibility를 분리한다.

```text
lifecycle:
candidate -> confirmed | invalidated | expired | manual_review

interaction:
active -> touched -> rejected | overrun

setup eligibility:
unranked -> eligible | filtered_out | manual_review
```

### candidate

liquidity taken event와 반전 기준 short-term swing을 연결할 수 있으면 candidate를 생성한다.

```text
BSL taken -> bearish breaker candidate
SSL taken -> bullish breaker candidate
```

### confirmed

```text
bearish candidate
+ break reference short-term low 아래 strict traversal
+ bearish source candle 식별
-> confirmed bearish breaker

bullish candidate
+ break reference short-term high 위 strict traversal
+ bullish source candle 식별
-> confirmed bullish breaker
```

confirmation은 traversal 봉 close 이후에만 사용할 수 있다.

### invalidated

raw lifecycle invalidated는 데이터 또는 provenance 오류에 사용한다.

- source swing이 OHLC 정정으로 확정 해제됐다.
- symbol 또는 timeframe 혼합이 발견됐다.
- source candle이 source window 밖으로 잘못 연결됐다.
- timestamp 정렬 또는 데이터 version이 바뀌었다.

zone overrun은 raw history를 삭제하지 않고 interaction과 setup eligibility를 갱신한다.

### expired

configured window 안 반대 방향 traversal이 없으면 candidate를 expired로 기록한다.

```text
bars_since_liquidity_taken > breaker_confirmation_window_bars
and no opposite swing traversal
-> lifecycle_status = expired
```

고정 window는 자막에 없으므로 `research default`이다.

### manual_review

- primary liquidity pool 선택에 따라 결과가 달라진다.
- break reference short-term swing 후보가 여러 개다.
- source window 선택에 따라 source candle이 달라진다.
- source candle 후보가 동률이다.
- HTF narrative와 raw breaker 방향이 충돌한다.
- bullish 방향 대칭 규칙을 영상 차트로 추가 검증해야 한다.

## 코드 구현 규칙

### 입력 데이터

```text
required:
  closed_ohlc[]
  confirmed_swing_points[]
  bsl_ssl_events[]
  breaker_confirmation_window_bars
  breaker_source_window_policy

optional:
  sweep_events[]
  displacement_events[]
  mss_events[]
  fvg_zones[]
  order_blocks[]
  dealing_ranges[]
  daily_bias_metadata
  session_metadata
  calendar_metadata
```

### 필요 변수

```text
breaker_id
symbol
timeframe
direction
source_candle_id
source_candle_selection
source_window_start_at
source_window_end_at
wick_zone_low
wick_zone_high
body_zone_low
body_zone_high
midpoint
linked_liquidity_event_id
liquidity_source_swing_id
break_reference_swing_id
reversal_traversal_bar_id
source_ob_id
symmetry_inference
gap_traversal
close_through_confirmed
lifecycle_status
interaction_status
setup_eligibility
first_touched_at
interaction_count
reason_codes[]
manual_review_reason[]
parameters_version
```

### 조건식

```text
confirm_bearish_breaker =
  liquidity_event.kind == BSL
  and liquidity_event.status == taken
  and later_bar.low < break_reference_short_term_low.level
  and bearish_source_candle exists

confirm_bullish_breaker =
  liquidity_event.kind == SSL
  and liquidity_event.status == taken
  and later_bar.high > break_reference_short_term_high.level
  and bullish_source_candle exists

zone_overlap =
  later_bar.high >= wick_zone_low
  and later_bar.low <= wick_zone_high

bearish_zone_overrun = later_bar.high > wick_zone_high
bullish_zone_overrun = later_bar.low < wick_zone_low
```

### 의사코드

```text
for each symbol, timeframe:
  sort closed bars by timestamp

  for liquidity_event in bsl_ssl_events where status == taken:
    candidate = create_breaker_candidate(liquidity_event)
    reference = select_confirmed_opposite_short_term_swing(candidate)

    for later closed bar within confirmation_window:
      if candidate is bearish and later.low < reference.low:
        source = lowest_down_close_before_linked_high_clear(candidate)
        confirm bearish breaker if source exists

      if candidate is bullish and later.high > reference.high:
        source = highest_up_close_before_linked_low_clear(candidate)
        confirm bullish breaker if source exists
        set symmetry_inference = true

for each confirmed breaker:
  create wick, body, midpoint, lower-half, upper-half ranges
  link matching overrun OB if one exists
  track only later zone interactions
  attach optional sweep, displacement, MSS, FVG, target, range, session metadata
  evaluate setup eligibility separately
```

## 최소 구현 버전

닫힌 OHLC와 선행 BSL/SSL Rulebook 출력을 사용한다.

- confirmed swing 기반 BSL/SSL `taken` event를 입력받는다.
- taken event 이후 반대편 short-term swing strict traversal을 확인한다.
- bearish는 lowest down-close, bullish는 highest up-close source candle을 선택한다.
- source candle wick zone, body zone, midpoint, half-zone을 저장한다.
- zone confirmation 이후 retest와 overrun을 추적한다.
- `source_ob_id`는 nullable link로 저장한다.
- breaker를 entry signal로 자동 변환하지 않는다.

최소 출력:

```text
id, symbol, timeframe, direction,
source_candle_id, source_candle_selection,
wick_zone_low, wick_zone_high,
body_zone_low, body_zone_high, midpoint,
linked_liquidity_event_id, liquidity_source_swing_id,
break_reference_swing_id, reversal_traversal_bar_id,
source_ob_id, symmetry_inference,
confirmed_at, lifecycle_status, interaction_status,
first_touched_at, interaction_count,
reason_codes, parameters_version
```

## 고급 구현 버전

### HTF

- HTF target liquidity와 LTF breaker를 parent-child link로 연결한다.
- HTF breaker zone과 LTF refinement zone을 독립 저장한다.
- HTF 봉 close 이전에는 해당 HTF breaker를 LTF signal에 사용하지 않는다.

### 세션

- raw breaker 정의를 session으로 제한하지 않는다.
- London, New York, lunch hour, afternoon tag를 추가한다.
- Episode 39 afternoon 사례를 별도 cohort로 분석한다.

### 유동성

- BSL/SSL source kind, timeframe, source pivot id를 저장한다.
- confirmed sweep 여부와 reclaim 시간을 metadata로 연결한다.
- target liquidity까지 distance와 elapsed bars를 저장한다.

### Displacement, MSS, FVG

- reversal 방향 displacement와 MSS id를 연결한다.
- retest 주변 FVG overlap을 계산한다.
- 해당 사건이 없어도 raw breaker를 삭제하지 않고 ranking을 낮춘다.

### Order Block

- matching overrun OB가 있으면 `source_ob_id`를 연결한다.
- breaker와 OB를 동일 객체로 합치지 않는다.
- OB provenance가 없는 breaker와 있는 breaker의 성과를 분리한다.

### SMT와 경제지표

- correlated symbol liquidity event와 divergence를 metadata로 연결한다.
- configured event window 안 breaker confirmation과 retest에 calendar event id를 붙인다.
- news spike를 삭제하지 않고 별도 cohort로 분석한다.

## False Signal 제거 방법

- 유동성 taken event를 요구한다.
- 반대편 short-term swing strict traversal을 요구한다.
- 동일 가격 touch를 traversal로 처리하지 않는다.
- source candle id와 source window를 저장한다.
- invalidated OB와 confirmed breaker를 동일시하지 않는다.
- raw zone 생성과 retest interaction을 분리한다.
- narrative, target liquidity, displacement, MSS, FVG를 setup metadata로 분리한다.
- source window와 confirmation window를 parameters version에 기록한다.

## Pine Script 구현 가이드

- `barstate.isconfirmed`에서 swing, liquidity taken, breaker confirmation을 갱신한다.
- breaker candidate와 confirmed zone 배열을 분리한다.
- wick zone은 box, midpoint와 half-zone은 line 또는 별도 box로 표시한다.
- `source_ob_id` 존재 여부와 `symmetry_inference`를 label에 표시한다.
- active, touched, rejected, overrun 상태를 시각적으로 구분한다.
- alert payload에 breaker id, direction, source candle id, liquidity event id, zone boundaries를 포함한다.

## Python Scanner 구현 가이드

다음 table을 분리한다.

```text
breaker_candidates
breaker_blocks
breaker_interactions
breaker_setup_eligibility
breaker_ob_links
breaker_context_metadata
```

- `groupby(symbol, timeframe)` 안에서 timestamp 순서를 보장한다.
- BSL/SSL event table과 confirmed short-term swing table을 join한다.
- source candle selection policy와 window를 versioning한다.
- confirmation 이후 row만 interaction으로 계산한다.
- optional OB, sweep, displacement, MSS, FVG, target, session, calendar metadata는 후속 join으로 연결한다.
- nullable `source_ob_id`를 허용한다.

## 백테스트 시 주의사항

- liquidity source swing은 `confirmed_at` 이후에만 사용한다.
- break reference swing도 `confirmed_at` 이후에만 사용한다.
- traversal 봉 close 이전에 breaker를 알고 있었다고 가정하지 않는다.
- source candle은 confirmation 이후 식별되므로 과거 source candle 시점에 진입했다고 계산하지 않는다.
- confirmation과 같은 봉의 zone overlap을 later retest로 중복 계산하지 않는다.
- source window, confirmation window, refinement 선택을 versioning한다.
- wick touch, body touch, midpoint touch, half-zone touch를 분리한다.
- bullish symmetry inference와 bearish direct-source 규칙의 성과를 분리한다.
- source OB link 유무에 따른 성과를 분리한다.
- overlapping breaker를 독립 거래처럼 중복 집계하지 않는다.

## 최종 Rulebook 정의

Breaker Block은 확정 BSL 또는 SSL이 strict traversal로 제거된 뒤 가격이 반대편 확정 short-term swing을 strict traversal했을 때 생성되는 source candle zone이다. bearish source는 linked high를 제거한 상승 이동 직전 source window의 lowest down-close candle이고, bullish source는 방향 대칭 구현 규칙에 따른 highest up-close candle이다. source OB overrun 이력은 nullable provenance link이며 hard dependency가 아니다. raw zone, 이후 retest interaction, narrative 기반 setup eligibility를 분리한다.
