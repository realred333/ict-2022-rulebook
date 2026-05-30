# Displacement Rulebook

공통 계약: [README.md](README.md)

## 목적

닫힌 OHLC에서 짧은 시간 안에 발생한 방향성 repricing을 객관적 프록시로 탐지한다. 최소 구현은 단일 봉의 상대 body 크기와 종가 위치를 사용한다. 고급 구현은 인접한 방향성 봉을 impulse leg로 묶고, 선행 sweep, 구조 돌파, FVG를 연결한다.

ICT 자막은 displacement를 `energetic`, `quick`, `sudden`, `obvious`, `animated`한 움직임으로 설명하지만 고정 수치 threshold를 제시하지 않는다. 이 Rulebook의 수치는 백테스트 가능한 `research default`이다.

## 객관적 정의

### 기본 계산

```text
body = abs(close - open)
range = high - low
body_ratio = body / range if range > 0 else null

baseline_body =
  median(body of previous median_body_length closed bars)

body_size_ratio =
  body / baseline_body if baseline_body > 0 else null
```

기본 파라미터:

```text
median_body_length = 20
displacement_body_factor = 1.50
displacement_body_ratio = 0.60
```

현재 봉은 baseline 계산에서 제외한다.

### Raw Displacement Bar Candidate

```text
is_raw_displacement_bar =
  range > 0
  and baseline_body > 0
  and body >= displacement_body_factor * baseline_body
  and body_ratio >= displacement_body_ratio

direction =
  bullish if close > open
  bearish if close < open
  none    if close == open
```

`direction = none`이면 raw displacement로 확정하지 않는다.

### 출력 객체

```text
displacement_event = {
  id,
  symbol,
  timeframe,
  direction,             # bullish | bearish
  detected_at,
  start_at,
  end_at,
  price_low,
  price_high,
  body,
  range,
  body_ratio,
  baseline_body,
  body_size_ratio,
  lifecycle_status,      # candidate | confirmed | invalidated | expired | manual_review
  event_kind,            # raw_bar | impulse_leg | structure_confirming_leg
  linked_sweep_id,
  broken_swing_id,
  structure_effect,      # none | broke_swing_high | broke_swing_low
  fvg_ids[],
  reason_codes[],
  parameters_version
}
```

### Structure Effect

raw displacement와 MSS 관련 구조 효과를 분리한다.

```text
bullish_structure_effect =
  direction == bullish
  and close > confirmed_swing_high.level

bearish_structure_effect =
  direction == bearish
  and close < confirmed_swing_low.level
```

- bullish structure effect: `broke_swing_high`
- bearish structure effect: `broke_swing_low`
- swing 돌파가 없으면 `none`

Displacement 모듈은 structure effect metadata를 기록한다. 최종 MSS 판정은 Market Structure Shift Rulebook 책임이다.

### Displacement Leg

자막은 displacement를 단일 봉뿐 아니라 range와 leg로도 설명한다. 최소 구현에서는 raw bar만 필수이다. 고급 구현은 다음 객체를 추가한다.

```text
displacement_leg = {
  id,
  direction,
  start_at,
  end_at,
  price_low,
  price_high,
  duration_bars,
  cumulative_move_ticks,
  cumulative_move_atr,
  member_bar_ids[],
  linked_sweep_id,
  broken_swing_id,
  structure_effect,
  fvg_ids[]
}
```

구조 확인형 leg anchor:

```text
bearish:
  start = preceding BSL sweep extreme or selected local high
  end   = close below confirmed short-term swing low

bullish:
  start = preceding SSL sweep extreme or selected local low
  end   = close above confirmed short-term swing high
```

Episode 6의 displacement high/low 설명을 구현하기 위한 계약이다. 자동 anchor 선택이 모호하면 `manual_review`를 사용한다.

### Optional Impulse Leg Aggregation

인접 방향성 봉 결합 규칙은 자막에 고정되어 있지 않다. 고급 구현의 research parameter이다.

```text
impulse_leg_max_pause_bars = 1
impulse_leg_max_duration_bars = configurable
```

raw displacement bar를 seed로 삼고 같은 방향 repricing을 묶는다. 결합 정책과 version을 반드시 저장한다.

### FVG 연결

```text
linked_fvg =
  fvg.created_at >= displacement.start_at
  and fvg.created_at <= displacement.end_at
  and fvg.direction == displacement.direction
```

FVG 탐지는 Fair Value Gap Rulebook 책임이다. Displacement 모듈은 참여 여부와 id만 연결한다.

## 필수 조건

- 입력 OHLC에 `timestamp, symbol, timeframe, open, high, low, close`가 있다.
- OHLC는 symbol과 timeframe별로 오름차순 정렬된다.
- 현재 봉이 닫힌 뒤에만 displacement event를 확정한다.
- baseline에는 현재 봉과 미래 봉을 포함하지 않는다.
- baseline은 같은 symbol과 timeframe의 이전 닫힌 봉으로만 계산한다.
- `range > 0`이다.
- `baseline_body > 0`이다.
- `close != open`이다.
- `median_body_length`, `body factor`, `body ratio threshold`를 parameters version에 저장한다.
- structure effect를 기록할 때 broken swing은 돌파 전에 확정되어 있어야 한다.
- linked sweep을 기록할 때 sweep id와 시간 순서를 보존한다.
- linked FVG를 기록할 때 FVG id와 방향을 보존한다.

## 무효 조건

- 현재 봉 또는 미래 봉을 baseline에 포함한다.
- 결측 OHLC 또는 데이터 gap을 정상 연속 데이터로 처리한다.
- range가 `0`인데 body ratio를 계산한다.
- baseline body가 `0`인데 size ratio를 정상값처럼 계산한다.
- wick 비중이 큰 spike를 body ratio 검사 없이 displacement로 확정한다.
- `close == open`인 doji를 방향성 displacement로 처리한다.
- swing이 확정되기 전에 structure effect를 붙인다.
- displacement candidate만으로 MSS 또는 진입을 확정한다.
- FVG가 있다는 이유만으로 displacement가 있다고 가정한다.
- research default를 ICT가 직접 고정한 값이라고 설명한다.

## 예외 조건

- 이전 20봉 부족: 최소 구현에서는 `candidate`를 만들지 않고 `reason = insufficient_history`를 기록한다.
- 데이터 gap 직후 봉: `manual_review` 또는 configured warm-up 재시작을 사용한다.
- baseline body가 `0`: non-zero body window 정책을 별도 parameter로 둘 수 있지만 기본 구현은 skip한다.
- news spike: raw displacement는 유지하고 calendar metadata를 붙인다. 거래 필터 적용은 전략 책임이다.
- wick spike: body threshold를 충족해도 body ratio 미달이면 제외한다.
- 복수 swing 동시 돌파: 모든 broken swing id를 저장하고 primary id 선택은 hierarchy 정책에 맡긴다.
- sweep 없는 displacement: raw event는 허용한다. `linked_sweep_id = null`로 저장한다.
- FVG 없는 displacement: raw event는 허용한다. setup ranking에서 별도 평가한다.
- 단일 봉 threshold 미달이지만 시각적으로 명확한 multi-bar repricing: 고급 impulse aggregator 또는 `manual_review`로 보낸다.
- HTF/LTF 중첩: timeframe별 event를 독립 유지하고 parent-child link를 추가한다.

## 상태 변화(State Machine)

```text
candidate -> confirmed | invalidated | expired | manual_review
```

### candidate

닫힌 봉이 방향성을 가지며 일부 조건을 만족하지만 최종 proxy 기준을 모두 만족하지 않은 상태를 선택적으로 기록할 수 있다.

```text
candidate =
  close != open
  and range > 0
  and body >= displacement_candidate_factor * baseline_body
```

기본값:

```text
displacement_candidate_factor = 1.00  # research default
```

최소 scanner는 candidate 저장을 생략하고 confirmed raw bar만 출력할 수 있다.

### confirmed

raw bar 기준을 모두 만족하면 confirmed이다.

```text
body >= 1.50 * baseline_body
and body_ratio >= 0.60
and close != open
```

다음을 추가 평가한다.

- linked sweep 존재 여부
- 사전 확정 swing 종가 돌파 여부
- FVG 참여 여부
- session과 calendar metadata

### invalidated

다음 중 하나이면 invalidated이다.

- look-ahead baseline이 사용되었다.
- 데이터 정정으로 threshold 조건을 더 이상 만족하지 않는다.
- symbol 또는 timeframe 혼합이 발견되었다.
- broken swing이 돌파 시점 이후에 확정된 것으로 판명되었다.
- upstream sweep link가 데이터 정정으로 invalidated되었다.

raw displacement 자체는 유효하지만 sweep link만 무효인 경우 event를 삭제하지 않고 link version만 invalidated한다.

### expired

candidate가 configured confirmation window 안에 confirmed event로 발전하지 못한 상태이다.

최소 구현:

```text
candidate_storage = disabled
```

고급 impulse leg 구현:

```text
candidate_expiry_bars = configurable
```

### manual_review

다음 중 하나이면 자동 결과와 함께 검토 사유를 남긴다.

- multi-bar repricing이 명확하지만 단일 봉 threshold를 충족하지 않는다.
- impulse leg anchor를 자동 선택할 수 없다.
- 복수 swing 돌파 중 primary structure reference를 선택해야 한다.
- news spike를 전략에서 제외할지 결정해야 한다.
- 데이터 gap 직후 event를 사용할지 결정해야 한다.

## 코드 구현 규칙

### 입력 데이터

```text
required:
  closed_ohlc[]
  median_body_length
  displacement_body_factor
  displacement_body_ratio

optional:
  tick_size
  atr_14[]
  confirmed_swings[]
  liquidity_sweep_events[]
  fair_value_gap_events[]
  session_metadata
  calendar_metadata
  impulse_leg_parameters
```

### 필요 변수

```text
displacement_id
symbol
timeframe
direction = bullish | bearish
detected_at
start_at
end_at
price_low
price_high
body
range
body_ratio
baseline_body
body_size_ratio
range_atr
lifecycle_status
event_kind
linked_sweep_id
broken_swing_id
structure_effect
fvg_ids[]
session_tags[]
calendar_event_ids[]
reason_codes[]
manual_review_reason[]
parameters_version
```

### 조건식

```text
body = abs(close - open)
range = high - low
body_ratio = body / range if range > 0 else null
baseline_body = median(previous median_body_length bodies)
body_size_ratio = body / baseline_body if baseline_body > 0 else null

is_bullish = close > open
is_bearish = close < open

is_raw_displacement =
  range > 0
  and baseline_body > 0
  and body_size_ratio >= displacement_body_factor
  and body_ratio >= displacement_body_ratio
  and (is_bullish or is_bearish)

broke_swing_high =
  is_bullish
  and swing.confirmed_at < bar.timestamp
  and close > swing_high.level

broke_swing_low =
  is_bearish
  and swing.confirmed_at < bar.timestamp
  and close < swing_low.level
```

### 의사코드

```text
for each symbol, timeframe:
  sort closed bars by timestamp

  for bar in closed_bars:
    history = previous median_body_length closed bars
    if history is insufficient:
      continue

    body = abs(bar.close - bar.open)
    range = bar.high - bar.low
    baseline = median(history.body)

    if range <= 0 or baseline <= 0:
      continue

    body_ratio = body / range
    body_size_ratio = body / baseline

    if body_size_ratio >= body_factor and body_ratio >= body_ratio_min:
      direction = bullish if bar.close > bar.open else bearish
      emit confirmed raw displacement bar

      attach nearest preceding compatible sweep if configured
      attach pre-confirmed swing ids broken by close
      attach same-direction FVG ids when available

for raw displacement seeds in advanced mode:
  aggregate configurable impulse legs
  build structure-confirming leg from sweep extreme to broken swing close
```

## 최소 구현 버전

닫힌 OHLC만 사용한다.

- symbol과 timeframe별 이전 `20`봉 body median을 계산한다.
- 현재 봉은 baseline에서 제외한다.
- `body >= 1.50 * baseline`과 `body / range >= 0.60`을 적용한다.
- bullish와 bearish를 분리한다.
- raw displacement bar만 출력한다.
- 수치를 parameters version에 저장한다.
- displacement를 entry signal로 변환하지 않는다.

최소 출력:

```text
id, symbol, timeframe, direction, detected_at,
price_low, price_high, body, range, body_ratio,
baseline_body, body_size_ratio, lifecycle_status,
event_kind, reason_codes, parameters_version
```

## 고급 구현 버전

### HTF

- timeframe별 baseline과 event를 독립 계산한다.
- LTF displacement에 가까운 HTF context id를 연결한다.
- HTF/LTF 이벤트를 합치지 않고 parent-child relationship으로 저장한다.
- symbol과 timeframe별 threshold를 백테스트 parameter로 노출한다.

### 세션

- displacement 정의 자체는 session으로 제한하지 않는다.
- killzone, London, New York, lunch hour tag를 metadata로 추가한다.
- `America/New_York` timezone과 DST-aware calendar를 사용한다.
- session별 baseline을 별도 실험할 경우 parameters version을 분리한다.

### 유동성

- preceding BSL sweep 뒤 bearish displacement를 연결한다.
- preceding SSL sweep 뒤 bullish displacement를 연결한다.
- sweep-to-displacement bars와 elapsed time을 계산한다.
- 반대 방향 displacement가 없으면 confluence 부재로 기록한다.

### SMT

- correlated symbol별 displacement timestamp와 방향을 정렬한다.
- SMT 모듈이 liquidity sweep 불일치와 repricing 차이를 비교할 수 있도록 metadata를 전달한다.
- Displacement 모듈 자체는 SMT를 확정하지 않는다.

### 경제지표

- configured event window 안 displacement에 calendar event id를 붙인다.
- news driver event를 삭제하지 않고 별도 cohort로 분석한다.
- calendar filter 적용 여부는 strategy parameter로 기록한다.

### Impulse Leg

- raw displacement bar를 seed로 configurable multi-bar leg를 생성한다.
- leg high/low, duration, cumulative move tick, ATR 비율을 저장한다.
- liquidity raid extreme과 structure-breaking close를 anchor로 structure-confirming leg를 생성한다.
- leg 내부 same-direction FVG ids를 연결한다.
- anchor 선택이 모호하면 `manual_review`를 사용한다.

## False Signal 제거 방법

- baseline 계산에서 현재 봉과 미래 봉을 제외한다.
- same symbol과 timeframe의 닫힌 봉만 사용한다.
- 데이터 gap 뒤 warm-up 정책을 적용한다.
- range와 baseline이 `0`인 봉을 제외한다.
- body ratio로 wick spike를 필터링한다.
- raw displacement와 structure-confirming displacement를 구분한다.
- displacement, MSS, FVG를 동일 사건으로 취급하지 않는다.
- FVG 존재만으로 displacement를 추론하지 않는다.
- sweep 없는 displacement를 삭제하지 않고 context 부재로 기록한다.
- news spike를 삭제하지 않고 metadata로 분리한다.
- research default를 versioned parameter로 관리한다.

## Pine Script 구현 가이드

Pine Script에서 현재 닫힌 봉을 이전 body baseline과 비교한다.

```text
body = math.abs(close - open)
range = high - low
baseline = ta.median(body[1], median_body_length)
body_ratio = range > 0 ? body / range : na

is_displacement =
  baseline > 0
  and body >= displacement_body_factor * baseline
  and body_ratio >= displacement_body_ratio
  and close != open
```

- baseline에 현재 봉이 포함되지 않도록 `body[1]`을 사용한다.
- `barstate.isconfirmed`에서 event를 확정한다.
- bullish와 bearish marker를 분리한다.
- linked sweep, broken swing, FVG 여부를 label 또는 alert metadata로 표시한다.
- 배열에는 event id, 방향, 시각, high/low, ratios를 저장한다.
- multi-bar leg는 배열 제한을 고려해 선택 기능으로 둔다.

## Python Scanner 구현 가이드

event table과 context link를 분리한다.

```text
displacement_events
displacement_impulse_legs
displacement_structure_links
displacement_fvg_links
displacement_context_metadata
```

- `groupby(symbol, timeframe)` 안에서 `body.shift(1).rolling(20).median()`을 계산한다.
- 현재 행이 rolling baseline에 포함되지 않도록 `shift(1)`을 먼저 적용한다.
- timestamp interval gap을 검증하고 gap 뒤 warm-up 정책을 적용한다.
- vectorized raw bar 탐지와 event table 생성을 분리한다.
- broken swing join은 `swing.confirmed_at < displacement.detected_at` 조건을 지킨다.
- preceding sweep join은 방향 호환성과 configured 최대 시간 차이를 기록한다.
- FVG join은 event 생성 이후 별도 link table에서 수행한다.
- impulse leg aggregation은 parameters version을 저장한다.

## 백테스트 시 주의사항

- 현재 displacement 봉 종가 이전에 event를 알고 있었다고 가정하지 않는다.
- baseline에 현재 봉 또는 미래 봉이 들어가면 look-ahead bias이다.
- `1.50`, `0.60`, `20`은 research default이다. symbol과 timeframe별 sensitivity test를 수행한다.
- news spike 포함 여부를 별도 cohort로 비교한다.
- raw displacement와 structure-confirming displacement 성과를 분리한다.
- FVG가 없는 displacement와 displacement가 없는 FVG를 각각 기록한다.
- sweep-to-displacement 최대 허용 시간은 전략 parameter로 명시한다.
- multi-bar leg 결합 정책이 바뀌면 결과도 달라진다.
- HTF event는 HTF 봉 close 이후에만 LTF에서 사용할 수 있다.
- 진입은 displacement 종가 체결 또는 후속 retracement 체결 중 무엇을 가정하는지 명시한다.
- 데이터 vendor의 gap, session, DST 처리 정책을 기록한다.

## 최종 Rulebook 정의

Displacement는 같은 symbol과 timeframe의 닫힌 OHLC에서 현재 봉을 제외한 이전 body median 대비 body가 충분히 크고, range 안에서 body 비율이 충분히 높은 방향성 repricing 프록시이다. 기본 threshold는 ICT의 고정 수치가 아니라 versioned research parameter이다. 최소 구현은 raw bar event를 생성하고, 고급 구현은 sweep, 사전 확정 swing 종가 돌파, impulse leg, FVG를 별도 metadata로 연결한다.
