# Market Structure Shift Rulebook

공통 계약: [README.md](README.md)

## 목적

look-ahead 없이 확정된 short-term swing과 BSL/SSL `taken` event를 입력으로 받아 contextual Market Structure Shift(MSS)를 탐지한다.

ICT 2022 Mentorship Episode 3은 swing high 위 거래가 필요하지만 해당 high 위 종가 마감은 필수가 아니라고 설명한다. Episode 6은 energetic displacement와 종가 이탈을 사용하는 더 강한 거래 후보 프로필을 설명한다.

따라서 이 Rulebook은 다음을 분리한다.

```text
strict traversal MSS        = 자막 기반 최소 판정
close-confirmed MSS         = 강화 tag
displacement-confirmed MSS  = 강화 tag
```

## 객관적 정의

### Upstream Event

MSS 후보의 기본 문맥은 [BSL / SSL Rulebook](bsl_ssl_rulebook.md)의 `taken` event이다.

```text
taken BSL -> expected MSS direction = bearish
taken SSL -> expected MSS direction = bullish
```

confirmed [Liquidity Sweep Rulebook](liquidity_sweep_rulebook.md) link는 유용한 confluence이지만 MSS 최소 판정의 필수 조건은 아니다. sweep reclaim window는 별도의 `research default`이며, canonical MSS 구조 판정을 바꾸지 않는다.

### Raw Structure Traversal

모든 strict swing traversal을 별도 event로 저장한다.

```text
bullish_structure_traversal =
  swing.kind == short_term_swing_high
  and swing.confirmed_at < bar.closed_at
  and bar.high > swing.level

bearish_structure_traversal =
  swing.kind == short_term_swing_low
  and swing.confirmed_at < bar.closed_at
  and bar.low < swing.level
```

동일 가격 touch는 traversal이 아니다.

```text
bar.high == swing_high.level -> no bullish traversal
bar.low  == swing_low.level  -> no bearish traversal
```

### Contextual MSS

```text
bullish_mss =
  compatible taken SSL event exists
  and bullish_structure_traversal

bearish_mss =
  compatible taken BSL event exists
  and bearish_structure_traversal
```

compatibility 기본 조건:

```text
taken_event.symbol == swing.symbol == bar.symbol
taken_event.timeframe == swing.timeframe == bar.timeframe
taken_event.taken_at <= traversal_bar.closed_at
bars_between(taken_event, traversal_bar) <= mss_context_lookback_bars
```

`mss_context_lookback_bars`는 자막 고정값이 아니라 versioned `research default`이다.

### Close Confirmation

종가 이탈은 별도 tag이다.

```text
bullish_close_confirmed =
  bullish_mss
  and bar.close > swing_high.level

bearish_close_confirmed =
  bearish_mss
  and bar.close < swing_low.level
```

```text
break_confirmation =
  traversal_only
  | close_confirmed
```

Episode 3의 최소 MSS를 보존하기 위해 `close_confirmed`를 canonical 필수 조건으로 강제하지 않는다.

### Displacement Confirmation

[Displacement Rulebook](displacement_rulebook.md)의 event를 별도로 연결한다.

```text
bullish_displacement_confirmed =
  bullish_mss
  and linked_displacement.direction == bullish

bearish_displacement_confirmed =
  bearish_mss
  and linked_displacement.direction == bearish
```

Displacement 모듈의 `structure_effect`는 종가 돌파 기반의 강화 metadata이다. `structure_effect = none`이어도 MSS strict traversal은 존재할 수 있다.

### 출력 객체

```text
mss_event = {
  id,
  symbol,
  timeframe,
  direction,                    # bullish | bearish
  lifecycle_status,             # candidate | confirmed | invalidated | expired | manual_review
  classification,               # contextual_mss
  detected_at,
  traversal_at,
  traversal_bar_id,
  traversal_price,
  break_confirmation,           # traversal_only | close_confirmed
  close_price,
  broken_swing_id,
  broken_swing_kind,
  broken_swing_strength,
  broken_swing_level,
  broken_swing_pivot_at,
  broken_swing_confirmed_at,
  eligible_broken_swing_ids[],
  primary_reference_policy,
  taken_event_id,
  liquidity_reference_id,
  swept_side,                   # BSL | SSL
  linked_sweep_id,
  linked_displacement_ids[],
  displacement_confirmed,
  linked_fvg_ids[],
  linked_order_block_ids[],
  bars_from_liquidity_taken,
  reason_codes[],
  manual_review_reason[],
  parameters_version
}
```

raw traversal은 MSS와 분리해 저장한다.

```text
structure_traversal_event = {
  id,
  symbol,
  timeframe,
  direction,
  detected_at,
  traversal_bar_id,
  traversal_price,
  break_confirmation,
  broken_swing_id,
  context_status,               # none | compatible_taken_event | linked_mss
  linked_mss_id,
  reason_codes[],
  parameters_version
}
```

## 필수 조건

- 입력 OHLC에 `timestamp, symbol, timeframe, open, high, low, close`가 있다.
- OHLC는 symbol과 timeframe별로 오름차순 정렬된다.
- broken swing은 [Swing Points Rulebook](swing_points_rulebook.md)에서 look-ahead 없이 확정되었다.
- `swing.confirmed_at < traversal_bar.closed_at`이다.
- bullish MSS reference는 confirmed short-term swing high이다.
- bearish MSS reference는 confirmed short-term swing low이다.
- bullish MSS에는 compatible taken SSL event가 있다.
- bearish MSS에는 compatible taken BSL event가 있다.
- traversal은 strict inequality로 판정한다.
- scanner event는 traversal 봉이 닫힌 뒤 확정한다.
- broken swing id, taken event id, traversal bar id를 보존한다.
- `break_confirmation`을 저장한다.
- reference 선택 정책과 parameters version을 저장한다.
- displacement, sweep, FVG link는 MSS 본체와 분리한다.

## 무효 조건

- 미래 봉을 사용해 swing이 확정되기 전에 MSS를 표시한다.
- `high == swing_high.level` 또는 `low == swing_low.level`을 strict traversal로 처리한다.
- opposing-side taken event 없이 raw swing traversal을 contextual MSS로 저장한다.
- bullish MSS에 taken BSL을 연결하거나 bearish MSS에 taken SSL을 연결한다.
- symbol 또는 timeframe이 다른 upstream event를 결합한다.
- 미래 FVG 또는 displacement를 미리 읽어 MSS 발생 시점에 이미 알고 있었다고 가정한다.
- `close_confirmed`를 Episode 3 기반 최소 MSS의 필수 조건이라고 설명한다.
- traversal event, MSS event, displacement event를 하나의 table row 의미로 합친다.
- 복수 reference 중 primary 선택 정책을 기록하지 않는다.

## 예외 조건

- same-level touch: MSS가 아니다. touch interaction만 선택적으로 저장한다.
- wick-only traversal: 유효 MSS가 될 수 있다. `break_confirmation = traversal_only`로 저장한다.
- close-confirmed traversal: `break_confirmation = close_confirmed`로 저장한다.
- same-bar liquidity taken과 MSS traversal: OHLC만으로 발생 순서를 완전히 복원할 수 없다. `manual_review_reason = intrabar_order_unknown`을 남긴다.
- gap traversal: bar open이 swing level 바깥에서 시작하면 `gap_traversal = true`를 기록한다.
- 복수 swing 동시 traversal: 모든 eligible id를 저장하고 primary reference 선택 정책을 기록한다.
- confirmed sweep 없음: MSS 최소 판정을 허용한다. `linked_sweep_id = null`로 저장한다.
- displacement 없음: MSS 최소 판정을 허용한다. `displacement_confirmed = false`로 저장한다.
- FVG 없음: MSS 최소 판정을 허용한다. retracement setup ranking에서 별도 평가한다.
- upstream BSL/SSL 정정: linked MSS를 invalidated하고 재계산한다.
- HTF/LTF 중첩: timeframe별 MSS를 독립 저장하고 parent-child link를 추가한다.
- intermediate-term 또는 long-term swing break: 고급 hierarchy profile에서 별도 event로 저장한다.

## Reference 선택 정책

### 최소 정책

최소 구현은 short-term swing만 사용한다.

```text
mss_reference_strength = short_term
primary_reference_policy = most_recent_eligible_confirmed_swing
```

eligible reference:

```text
bullish candidate:
  confirmed short-term swing highs
  where swing.confirmed_at < traversal_bar.closed_at

bearish candidate:
  confirmed short-term swing lows
  where swing.confirmed_at < traversal_bar.closed_at
```

복수 reference를 한 봉에서 관통하면 모두 저장하고 가장 최근 `pivot_at`을 primary로 둔다.

이 정책은 재현 가능한 초기 구현을 위한 `research default`이다. ICT가 모든 chart context에 대해 하나의 reference selector를 고정한 것은 아니다.

### 고급 정책

다음을 optional profile로 분리한다.

```text
reference_strength =
  short_term
  | intermediate_term
  | long_term
  | multi_strength

reference_window =
  before_liquidity_taken
  | after_liquidity_taken
  | both
```

Episode 3과 Episode 6의 사례는 reference가 liquidity taken 전후에 있을 수 있음을 보여준다. hard-coded ordering을 강제하지 않는다.

## 상태 변화(State Machine)

```text
candidate -> confirmed | invalidated | expired | manual_review
```

### candidate

새로운 BSL/SSL taken event를 받으면 반대 방향 MSS candidate를 연다.

```text
taken BSL:
  direction = bearish
  expected_reference_kind = short_term_swing_low

taken SSL:
  direction = bullish
  expected_reference_kind = short_term_swing_high
```

candidate는 compatible swing registry와 신규 닫힌 봉을 관찰한다.

```text
mss_candidate = {
  id,
  taken_event_id,
  direction,
  opened_at,
  expected_reference_kind,
  lifecycle_status = candidate,
  expires_after_bars,
  parameters_version
}
```

### confirmed

configured context window 안에 opposite-direction strict swing traversal이 발생하면 confirmed이다.

```text
bullish candidate + high > eligible swing_high.level -> confirmed bullish MSS
bearish candidate + low  < eligible swing_low.level  -> confirmed bearish MSS
```

추가 tag:

```text
close_confirmed
linked_sweep_id
linked_displacement_ids[]
displacement_confirmed
linked_fvg_ids[]
```

### invalidated

다음 중 하나이면 invalidated이다.

- upstream taken event가 look-ahead 또는 데이터 정정으로 invalidated되었다.
- broken swing이 traversal 시점 이후에 확정된 것으로 판명되었다.
- broken swing source version이 바뀌었다.
- side 매핑 오류가 발견되었다.
- symbol 또는 timeframe 혼합이 발견되었다.

후속 가격이 예상 방향과 다르게 움직였다는 이유만으로 과거 confirmed MSS를 삭제하지 않는다. 전략 결과를 별도 기록한다.

### expired

candidate가 configured window 안에 MSS traversal로 이어지지 않으면 expired이다.

```text
bars_since_taken > mss_context_lookback_bars
and no compatible traversal
-> lifecycle_status = expired
```

기본 window는 timeframe별 설정값이다.

```text
mss_context_lookback_bars = configurable  # research default
```

자막은 모든 timeframe에 공통인 고정 봉 수를 제시하지 않는다.

### manual_review

다음 중 하나이면 자동 결과와 함께 검토 사유를 남긴다.

- same-bar taken과 traversal의 intrabar 순서가 중요하다.
- 복수 swing 중 narrative상 primary reference를 선택해야 한다.
- HTF와 LTF shift가 충돌한다.
- gap traversal을 일반 traversal과 같은 전략으로 처리할지 결정해야 한다.
- wick-only traversal을 특정 전략 cohort에서 제외할지 결정해야 한다.

## 코드 구현 규칙

### 입력 데이터

```text
required:
  closed_ohlc[]
  confirmed_short_term_swings[]
  bsl_ssl_taken_events[]
  mss_context_lookback_bars

optional:
  confirmed_sweep_events[]
  displacement_events[]
  fair_value_gap_events[]
  order_block_events[]
  intermediate_term_swings[]
  long_term_swings[]
  htf_context[]
  session_metadata
  calendar_metadata
  correlated_symbol_mss_events[]
```

### 필요 변수

```text
mss_id
candidate_id
symbol
timeframe
direction = bullish | bearish
lifecycle_status
classification
detected_at
traversal_at
traversal_bar_id
traversal_price
close_price
break_confirmation
gap_traversal
broken_swing_id
broken_swing_kind
broken_swing_strength
broken_swing_level
broken_swing_pivot_at
broken_swing_confirmed_at
eligible_broken_swing_ids[]
primary_reference_policy
taken_event_id
liquidity_reference_id
swept_side
linked_sweep_id
linked_displacement_ids[]
displacement_confirmed
linked_fvg_ids[]
linked_order_block_ids[]
bars_from_liquidity_taken
reason_codes[]
manual_review_reason[]
parameters_version
```

### 조건식

```text
is_bullish_candidate =
  taken_event.liquidity_side == SSL

is_bearish_candidate =
  taken_event.liquidity_side == BSL

is_preconfirmed_reference =
  swing.confirmed_at < bar.closed_at

is_bullish_traversal =
  is_preconfirmed_reference
  and swing.kind == short_term_swing_high
  and bar.high > swing.level

is_bearish_traversal =
  is_preconfirmed_reference
  and swing.kind == short_term_swing_low
  and bar.low < swing.level

is_bullish_close_confirmed =
  is_bullish_traversal
  and bar.close > swing.level

is_bearish_close_confirmed =
  is_bearish_traversal
  and bar.close < swing.level
```

### 의사코드

```text
for each new BSL_SSL_taken_event:
  if liquidity_side == BSL:
    open MSS candidate(direction = bearish)

  if liquidity_side == SSL:
    open MSS candidate(direction = bullish)

for each symbol, timeframe:
  sort closed bars by timestamp

  for bar in newly_closed_bars:
    detect and store every raw strict structure traversal

    for candidate in compatible open MSS candidates:
      if bars_since(candidate.opened_at) > mss_context_lookback_bars:
        expire candidate
        continue

      references = eligible pre-confirmed short-term swings

      if candidate.direction == bullish:
        traversed = references.highs where bar.high > swing.level

      if candidate.direction == bearish:
        traversed = references.lows where bar.low < swing.level

      if traversed is not empty:
        primary = choose by versioned primary_reference_policy
        emit confirmed contextual MSS
        store all eligible broken swing ids
        tag close_confirmed if close crossed primary level
        link compatible sweep and same-direction displacement when available

for each confirmed MSS:
  link same-leg FVG and OB candidates in downstream join tables
```

## 최소 구현 버전

닫힌 OHLC, confirmed short-term swing, BSL/SSL taken event만 사용한다.

- taken BSL 뒤 bearish MSS candidate를 연다.
- taken SSL 뒤 bullish MSS candidate를 연다.
- bullish는 confirmed short-term swing high 위 `high > level`을 검사한다.
- bearish는 confirmed short-term swing low 아래 `low < level`을 검사한다.
- 동일 가격 touch를 제외한다.
- traversal 봉 close 이후 event를 확정한다.
- broken swing id와 taken event id를 저장한다.
- 종가 이탈 여부를 별도 tag로 저장한다.
- 가장 최근 eligible confirmed short-term swing을 primary reference로 둔다.
- primary reference 정책과 context window를 parameters version에 저장한다.
- MSS를 entry signal로 변환하지 않는다.

최소 출력:

```text
id, symbol, timeframe, direction, lifecycle_status,
detected_at, traversal_at, traversal_bar_id, traversal_price,
break_confirmation, close_price,
broken_swing_id, broken_swing_level, broken_swing_confirmed_at,
taken_event_id, swept_side, bars_from_liquidity_taken,
reason_codes, parameters_version
```

## 고급 구현 버전

### Close-confirmed 전략 프로필

최소 MSS table은 유지하고 전략 filter에서 다음을 요구할 수 있다.

```text
strategy_requires_close_confirmation = true

bullish setup:
  MSS direction == bullish
  and break_confirmation == close_confirmed

bearish setup:
  MSS direction == bearish
  and break_confirmation == close_confirmed
```

이 설정은 Episode 6의 강화 모델을 연구하는 profile이다.

### Displacement 프로필

```text
strategy_requires_displacement = true
```

- bullish MSS에는 bullish displacement를 연결한다.
- bearish MSS에는 bearish displacement를 연결한다.
- displacement event timestamp와 MSS traversal timestamp의 관계를 저장한다.
- 동일 leg 판정 정책을 parameters version에 기록한다.
- displacement가 없다는 이유로 canonical MSS event를 삭제하지 않는다.

### HTF

- timeframe별 swing registry와 MSS event를 독립 유지한다.
- HTF MSS는 HTF 봉 close 이후에만 LTF 전략에서 사용할 수 있다.
- LTF event에 가까운 HTF structure context id를 연결한다.
- HTF와 LTF 방향이 충돌하면 `manual_review` 또는 strategy filter를 사용한다.

### 세션

- MSS 정의 자체는 session으로 제한하지 않는다.
- New York killzone, London, lunch hour, afternoon tag를 metadata로 추가한다.
- `America/New_York` timezone과 DST-aware calendar를 사용한다.
- session별 MSS 성과를 별도 cohort로 분석한다.

### 유동성

- BSL/SSL taken event를 필수 foreign key로 보존한다.
- confirmed sweep id가 있으면 link한다.
- sweep extreme과 MSS traversal 사이의 elapsed bars와 ticks를 기록한다.
- relative equal highs/lows, old high/low, session high/low source kind를 보존한다.
- run_or_breakout candidate와 MSS를 잘못 결합하지 않는다.

### FVG와 Order Block

- MSS 본체와 retracement zone을 분리한다.
- 같은 방향 displacement leg 내부 FVG ids를 연결한다.
- OB 후보는 OB Rulebook에서 판정하고 id만 연결한다.
- MSS가 있다는 이유만으로 FVG 또는 OB를 자동 생성하지 않는다.

### SMT

- correlated symbol별 comparable liquidity taken과 MSS timestamp를 정렬한다.
- 한 symbol만 comparable MSS가 발생한 경우 confidence metadata를 추가한다.
- MSS 모듈 자체는 SMT divergence를 확정하지 않는다.

### 경제지표

- configured event window 안 MSS에 calendar event id를 붙인다.
- news driver는 MSS 정의를 바꾸지 않는다.
- gap traversal과 일반 traversal을 분리해 분석한다.

### Multi-strength

- intermediate-term 및 long-term swing traversal을 별도 strength로 저장한다.
- short-term event와 상위 strength event를 합치지 않는다.
- hierarchy별 broken swing id와 `confirmed_at`을 보존한다.
- reference strength별 성과를 별도 cohort로 분석한다.

## False Signal 제거 방법

- look-ahead 없이 확정된 swing만 사용한다.
- touch와 strict traversal을 구분한다.
- raw structure traversal과 contextual MSS를 분리한다.
- bullish MSS에는 SSL taken, bearish MSS에는 BSL taken을 연결한다.
- same symbol과 timeframe event만 기본 결합한다.
- close confirmation과 displacement confirmation을 별도 tag로 저장한다.
- close-confirmed 전략은 canonical MSS table을 덮어쓰지 않는다.
- 복수 broken swing을 저장하고 primary 선택 정책을 versioning한다.
- same-bar liquidity taken과 traversal에는 intrabar 순서 불확실성을 기록한다.
- FVG, OB, MSS를 동일 사건으로 취급하지 않는다.
- HTF/LTF event를 중복 수익으로 계산하지 않는다.

## Pine Script 구현 가이드

Pine Script에서는 confirmed swing 배열과 active MSS candidate 배열을 분리한다.

```text
on confirmed bar:
  if active BSL taken:
    open bearish MSS candidate

  if active SSL taken:
    open bullish MSS candidate

for candidate in active candidates:
  if bullish:
    if high > selected_confirmed_swing_high.level:
      confirm bullish MSS
      close_confirmed = close > selected_confirmed_swing_high.level

  if bearish:
    if low < selected_confirmed_swing_low.level:
      confirm bearish MSS
      close_confirmed = close < selected_confirmed_swing_low.level
```

- `barstate.isconfirmed`에서 scanner-compatible event를 확정한다.
- traversal-only와 close-confirmed marker를 다른 색상 또는 label로 표시한다.
- broken swing line과 taken liquidity line을 함께 표시한다.
- alert payload에 `mss_id`, `taken_event_id`, `broken_swing_id`, `break_confirmation`을 넣는다.
- intrabar alert를 추가하면 provisional event로 분리하고 repaint 가능성을 명시한다.
- 배열 제한을 고려해 오래된 시각화 객체만 정리하고 event count는 유지한다.

## Python Scanner 구현 가이드

event table과 link table을 분리한다.

```text
structure_traversal_events
mss_candidates
mss_events
mss_liquidity_links
mss_sweep_links
mss_displacement_links
mss_fvg_links
mss_order_block_links
mss_context_metadata
```

- `groupby(symbol, timeframe)` 안에서 timestamp 순서를 보장한다.
- swing registry join은 `swing.confirmed_at < traversal_bar.closed_at`을 지킨다.
- raw structure traversal을 먼저 생성한다.
- compatible BSL/SSL taken event와 join해 contextual MSS를 만든다.
- same-level touch를 제외한다.
- `high > level`, `low < level` strict inequality를 사용한다.
- close-confirmed 여부는 별도 boolean 또는 enum으로 저장한다.
- taken event와 traversal 사이 elapsed bars를 계산한다.
- 복수 reference traversal을 모두 link table에 저장한다.
- primary reference 선택 정책을 parameters version에 기록한다.
- sweep, displacement, FVG, OB는 후속 join으로 연결한다.
- HTF event는 해당 HTF 봉 close 이후 LTF row에만 forward-fill한다.

## 백테스트 시 주의사항

- broken swing이 traversal 전에 확정되어 있었는지 검증한다.
- MSS event는 traversal 봉 close 이전에 알고 있었다고 가정하지 않는다.
- intrabar 전략을 테스트하려면 lower timeframe 또는 tick data가 필요하다.
- Episode 3 기반 traversal-only cohort와 Episode 6 기반 close-confirmed displacement cohort를 분리한다.
- `mss_context_lookback_bars`가 바뀌면 결과도 달라진다.
- primary reference selector가 바뀌면 event count와 결과도 달라진다.
- wick-only, close-confirmed, gap traversal을 별도 cohort로 분석한다.
- confirmed sweep 유무를 별도 cohort로 분석한다.
- displacement 및 FVG 유무를 별도 cohort로 분석한다.
- multi-strength event를 같은 거래 기회로 중복 집계하지 않는다.
- HTF MSS는 HTF close 이후에만 사용할 수 있다.
- calendar event 전후 gap을 일반 price delivery와 분리한다.
- vendor별 session, timestamp, DST 정책을 기록한다.

## 최종 Rulebook 정의

Market Structure Shift는 compatible opposing-side BSL 또는 SSL `taken` event 뒤, look-ahead 없이 사전 확정된 반대 방향 short-term swing을 strict traversal할 때 확정되는 문맥 기반 구조 변화 event이다. bullish MSS는 sell-side liquidity taken 뒤 swing high 위 `high > level`, bearish MSS는 buy-side liquidity taken 뒤 swing low 아래 `low < level`로 판정한다. Episode 3의 최소 정의를 보존하기 위해 swing level 바깥 종가 마감을 강제하지 않으며, 종가 이탈, confirmed sweep, same-direction displacement, FVG는 별도 강화 metadata로 연결한다.
