# OTE Rulebook

공통 계약: [README.md](README.md)

## 목적

OTE(Optimal Trade Entry)를 재현 가능한 direction-aware retracement zone으로 계산한다. range 선택, anchor refinement, raw zone, touch interaction, FVG 또는 OB overlap, setup eligibility를 분리한다.

## 객관적 정의

### Corpus Default

Episode 19, 22, 38의 반복 근거를 기준으로 기본 OTE retracement 구간을 정의한다.

```text
ote_ratio_near = 0.62
ote_ratio_far  = 0.79
```

```text
derived_ratio_midpoint =
  (ote_ratio_near + ote_ratio_far) / 2
  = 0.705
```

`0.705`는 전체 정제 자막에서 직접 검색되지 않는다. 별도 ICT threshold가 아니라 파생 metadata이다.

Episode 41 정제본과 대응 원본 대조본의 `62 to 70` 전사는 `transcript_conflict`로 저장한다. corpus default를 근거 없이 자동 변경하지 않는다.

### Anchor Range

```text
anchor_range = {
  range_id,
  symbol,
  timeframe,
  direction,                 # bullish | bearish
  anchor_low,
  anchor_high,
  anchor_low_source_id,
  anchor_high_source_id,
  anchor_policy,             # swing_wick | body_refined
  selected_at,
  parameters_version
}
```

```text
valid_anchor_range = anchor_high > anchor_low
span = anchor_high - anchor_low
```

### Anchor Policy

최소 구현:

```text
swing_wick:
  anchor_low  = selected dealing range low
  anchor_high = selected dealing range high
```

Episode 22 refinement:

```text
body_refined:
  anchor_low  = lowest open or close in lower swing
  anchor_high = highest open or close in upper swing
```

### Bullish OTE

low에서 high로 진행한 leg의 되돌림을 high에서 아래 방향으로 계산한다.

```text
bullish_price(r) = anchor_high - r * span

bullish_ote_zone_low  = bullish_price(0.79)
bullish_ote_zone_high = bullish_price(0.62)
bullish_ote_midpoint  = bullish_price(0.705)
```

### Bearish OTE

high에서 low로 진행한 leg의 되돌림을 low에서 위 방향으로 계산한다.

```text
bearish_price(r) = anchor_low + r * span

bearish_ote_zone_low  = bearish_price(0.62)
bearish_ote_zone_high = bearish_price(0.79)
bearish_ote_midpoint  = bearish_price(0.705)
```

### Premium / Discount Consistency

```text
bullish OTE zone -> normalized position below 0.50 from current high retracement frame
bearish OTE zone -> normalized position above 0.50 from current low retracement frame
```

가격축 기준으로 저장할 때는 [Premium / Discount / Equilibrium Rulebook](premium_discount_equilibrium_rulebook.md)의 normalized position도 함께 기록한다.

### 출력 객체

```text
ote_zone = {
  id,
  symbol,
  timeframe,
  direction,
  created_at,
  confirmed_at,
  range_id,
  anchor_policy,
  anchor_low,
  anchor_high,
  anchor_low_source_id,
  anchor_high_source_id,
  span,
  ratio_near,                    # 0.62
  ratio_far,                     # 0.79
  derived_ratio_midpoint,        # 0.705
  zone_low,
  zone_high,
  derived_midpoint_price,
  lifecycle_status,              # candidate | confirmed | invalidated | expired | manual_review
  interaction_status,            # active | touched | traversed | expired
  setup_eligibility,             # unranked | eligible | filtered_out | manual_review
  first_touched_at,
  interaction_count,
  normalized_zone_low,
  normalized_zone_high,
  premium_discount_consistent,
  linked_fvg_ids[],
  linked_ob_ids[],
  linked_mss_ids[],
  linked_sweep_ids[],
  target_liquidity_ids[],
  session_tags[],
  calendar_event_ids[],
  transcript_conflicts[],
  reason_codes[],
  manual_review_reason[],
  parameters_version
}
```

## 필수 조건

- 입력 range에 `range_id, symbol, timeframe, direction, anchor_low, anchor_high`가 있다.
- `anchor_high > anchor_low`이다.
- range endpoint는 사용 시점에 이미 확정되어 있다.
- range 선택 시각과 anchor policy를 저장한다.
- 기본 ratio는 `0.62`, `0.79`이다.
- `0.705`는 `derived_ratio_midpoint`로 저장한다.
- bullish와 bearish 계산식을 분리한다.
- raw zone confirmation과 이후 interaction을 분리한다.
- FVG, OB, MSS, sweep은 optional metadata로 연결한다.
- range가 바뀌면 기존 zone을 덮어쓰지 않고 새 `range_id`와 parameters version으로 계산한다.

## 무효 조건

- `anchor_high <= anchor_low`이다.
- range endpoint가 미확정이다.
- 미래 봉으로 range를 선택하고 과거 시점에 OTE를 알고 있었다고 가정한다.
- bullish와 bearish 가격 계산 방향을 뒤집는다.
- range id 또는 anchor policy를 저장하지 않는다.
- 사후 차트에 맞게 anchor를 조용히 변경한다.
- `0.705`를 직접 인용된 독립 threshold로 기록한다.
- Episode 41 전사 충돌을 근거 없이 자동 교정한다.
- OTE touch를 단독 entry signal로 처리한다.

## 예외 조건

- `body_refined` anchor가 wick range 밖으로 계산되면 input provenance를 검토하고 `manual_review`를 추가한다.
- lower swing 또는 upper swing 내부 open-close 후보가 복수 동률이면 source id를 모두 보존한다.
- range가 갱신되면 이전 zone history를 유지하고 새 zone을 생성한다.
- bullish와 bearish range 후보가 동시에 있으면 range별 OTE zone을 독립 저장한다.
- OTE zone이 FVG 또는 OB와 겹치지 않아도 raw zone은 유효하다.
- interaction bar가 zone 전체를 gap으로 통과하면 `gap_traversal = true`를 저장한다.
- Episode 41 `62 to 70` 전사는 `transcript_conflict`로 남기고 영상 확인 전 별도 strategy profile로 강제하지 않는다.

## 상태 변화(State Machine)

raw lifecycle, interaction, setup eligibility를 분리한다.

```text
lifecycle:
candidate -> confirmed | invalidated | expired | manual_review

interaction:
active -> touched | traversed | expired

setup eligibility:
unranked -> eligible | filtered_out | manual_review
```

### candidate

유효한 selected dealing range를 입력받으면 OTE candidate를 생성한다.

```text
range selected
and anchor provenance exists
-> candidate
```

### confirmed

range endpoint가 모두 확정됐고 direction-aware 계산을 완료하면 raw OTE zone을 confirmed로 기록한다.

```text
valid anchor range
+ confirmed endpoints
+ direction-aware 0.62~0.79 calculation
-> confirmed raw OTE zone
```

touch는 zone confirmation 조건이 아니다.

### invalidated

- source range가 OHLC 정정으로 무효화됐다.
- endpoint provenance가 사라졌다.
- symbol 또는 timeframe 혼합이 발견됐다.
- anchor policy 구현 오류가 발견됐다.
- parameters version 변경으로 재계산이 필요하다.

### expired

configured interaction window 안 touch가 없으면 interaction을 expired로 기록할 수 있다.

```text
bars_since_zone_confirmation > ote_interaction_window_bars
and no touch
-> interaction_status = expired
```

고정 interaction window는 자막에 없으므로 `research default`이다.

### manual_review

- 복수 range 중 primary narrative range를 선택해야 한다.
- wick anchor와 body-refined anchor 결과가 크게 다르다.
- HTF와 LTF OTE zone 방향이 충돌한다.
- Episode 41 `62 to 70` 전사를 영상으로 확인해야 한다.
- overlap FVG 또는 OB 중 primary setup을 선택해야 한다.

## 코드 구현 규칙

### 입력 데이터

```text
required:
  selected_dealing_ranges[]
  ote_ratio_near = 0.62
  ote_ratio_far = 0.79
  anchor_policy

optional:
  closed_ohlc[]
  swing_points[]
  premium_discount_metadata
  fvg_zones[]
  order_blocks[]
  mss_events[]
  sweep_events[]
  target_liquidity_events[]
  daily_bias_metadata
  session_metadata
  calendar_metadata
  ote_interaction_window_bars
```

### 필요 변수

```text
ote_id
symbol
timeframe
direction
range_id
anchor_policy
anchor_low
anchor_high
anchor_low_source_id
anchor_high_source_id
span
ratio_near
ratio_far
derived_ratio_midpoint
zone_low
zone_high
derived_midpoint_price
confirmed_at
lifecycle_status
interaction_status
setup_eligibility
first_touched_at
interaction_count
gap_traversal
linked_fvg_ids[]
linked_ob_ids[]
linked_mss_ids[]
linked_sweep_ids[]
target_liquidity_ids[]
transcript_conflicts[]
reason_codes[]
manual_review_reason[]
parameters_version
```

### 조건식

```text
span = anchor_high - anchor_low
valid_range = span > 0

if direction == bullish:
  zone_low  = anchor_high - 0.79 * span
  zone_high = anchor_high - 0.62 * span
  midpoint  = anchor_high - 0.705 * span

if direction == bearish:
  zone_low  = anchor_low + 0.62 * span
  zone_high = anchor_low + 0.79 * span
  midpoint  = anchor_low + 0.705 * span

zone_overlap =
  later_bar.high >= zone_low
  and later_bar.low <= zone_high

fvg_overlap =
  ote.zone_high >= fvg.zone_low
  and ote.zone_low <= fvg.zone_high

ob_overlap =
  ote.zone_high >= ob.zone_low
  and ote.zone_low <= ob.zone_high
```

### 의사코드

```text
for range in selected_dealing_ranges:
  assert range endpoints were confirmed before range.selected_at
  anchors = select_anchors(range, anchor_policy)

  if anchors.high <= anchors.low:
    invalidate candidate
    continue

  span = anchors.high - anchors.low

  if range.direction == bullish:
    zone_low = anchors.high - 0.79 * span
    zone_high = anchors.high - 0.62 * span
    midpoint = anchors.high - 0.705 * span

  if range.direction == bearish:
    zone_low = anchors.low + 0.62 * span
    zone_high = anchors.low + 0.79 * span
    midpoint = anchors.low + 0.705 * span

  emit confirmed raw OTE zone

for each confirmed OTE zone:
  track only later zone interactions
  link optional premium/discount, FVG, OB, MSS, sweep, target, session metadata
  evaluate setup eligibility separately
```

## 최소 구현 버전

Dealing Range Rulebook 출력만 사용한다.

- 완료된 selected range를 입력받는다.
- `swing_wick` anchor policy를 사용한다.
- bullish와 bearish 가격 계산을 분리한다.
- `0.62`, `0.79`, derived `0.705` 가격을 저장한다.
- confirmation 이후 touch와 interaction count를 기록한다.
- FVG 또는 OB overlap이 없어도 raw zone을 저장한다.
- OTE touch를 entry signal로 자동 변환하지 않는다.

최소 출력:

```text
id, symbol, timeframe, direction, range_id,
anchor_policy, anchor_low, anchor_high,
ratio_near, ratio_far, derived_ratio_midpoint,
zone_low, zone_high, derived_midpoint_price,
confirmed_at, lifecycle_status, interaction_status,
first_touched_at, interaction_count,
reason_codes, parameters_version
```

## 고급 구현 버전

### HTF

- timeframe별 dealing range와 OTE zone을 독립 계산한다.
- LTF zone에 겹치는 HTF OTE id를 parent link로 추가한다.
- HTF endpoint 확정 봉 close 이후에만 LTF setup에 사용한다.

### Anchor Refinement

- `swing_wick`과 Episode 22 `body_refined` anchor를 모두 계산한다.
- 두 zone의 가격 차이, tick distance, ATR 비율을 저장한다.
- anchor policy별 성과를 별도 cohort로 분석한다.

### 유동성

- preceding sweep 또는 BSL/SSL taken id를 연결한다.
- 방향별 target liquidity id와 distance를 저장한다.
- OTE zone touch 전후 target 상태 변화를 기록한다.

### FVG, OB, MSS

- OTE zone과 FVG, OB overlap 범위와 overlap ratio를 저장한다.
- same-direction MSS id를 연결한다.
- Episode 41 결합 패턴은 별도 strategy profile로 평가한다.
- confluence가 없어도 raw OTE zone을 삭제하지 않는다.

### 세션, SMT, 경제지표

- London, New York, afternoon 등 session tag를 추가한다.
- correlated symbol의 comparable retracement와 SMT metadata를 연결한다.
- configured event window 안 OTE touch에 calendar event id를 붙인다.
- news spike를 삭제하지 않고 별도 cohort로 분석한다.

## False Signal 제거 방법

- 확정된 range endpoint만 사용한다.
- range id, selection time, anchor policy를 저장한다.
- bullish와 bearish 계산식을 분리한다.
- 사후 range 변경을 새 version으로 기록한다.
- `0.705`를 derived metadata로 표시한다.
- Episode 41 transcript conflict를 보존한다.
- raw zone과 touch interaction을 분리한다.
- FVG, OB, MSS, liquidity, session은 setup metadata로 연결한다.
- overlap 없이 OTE touch만 발생한 사례를 별도 cohort로 분석한다.

## Pine Script 구현 가이드

- selected range endpoint 확정 이후에만 OTE zone을 생성한다.
- `barstate.isconfirmed`에서 interaction을 갱신한다.
- bullish와 bearish zone을 box로 구분한다.
- `0.62`, derived `0.705`, `0.79`를 line으로 표시할 수 있다.
- anchor policy와 range id를 label에 표시한다.
- wick 및 body-refined zone을 동시에 표시할 때 색상 또는 line style을 구분한다.
- alert payload에 OTE id, direction, range id, anchor policy, zone boundaries를 포함한다.

## Python Scanner 구현 가이드

다음 table을 분리한다.

```text
ote_anchor_ranges
ote_zones
ote_interactions
ote_setup_eligibility
ote_fvg_links
ote_ob_links
ote_context_metadata
```

- `groupby(symbol, timeframe)` 안에서 timestamp 순서를 보장한다.
- selected dealing range와 endpoint confirmation time을 검증한다.
- `swing_wick`과 `body_refined` 계산 함수를 분리한다.
- ratio와 anchor policy를 parameters version에 기록한다.
- confirmation 이후 row만 interaction으로 계산한다.
- optional FVG, OB, MSS, liquidity, HTF, session, calendar metadata는 후속 join으로 연결한다.
- transcript conflict는 문서 metadata 또는 parameters registry에 보존한다.

## 백테스트 시 주의사항

- endpoint 확정 시각 이전에 range를 알고 있었다고 가정하지 않는다.
- range selected time 이전에 OTE zone을 거래하지 않는다.
- range를 사후 변경하면 새 zone id와 parameters version을 만든다.
- wick anchor와 body-refined anchor 성과를 분리한다.
- bullish와 bearish cohort를 분리한다.
- zone touch, midpoint touch, full traversal, gap traversal을 분리한다.
- FVG, OB, MSS, liquidity confluence 유무를 별도 cohort로 분석한다.
- HTF endpoint는 HTF 봉 close 이후에만 LTF에서 사용한다.
- overlapping OTE zone을 독립 거래처럼 중복 집계하지 않는다.
- `0.62~0.79` corpus default와 실험 profile을 섞지 않는다.

## 최종 Rulebook 정의

OTE는 versioned dealing range anchor에 대해 방향별로 계산하는 `0.62~0.79` retracement zone이다. bullish leg는 high에서 아래 방향으로, bearish leg는 low에서 위 방향으로 계산한다. `0.705`는 두 경계의 산술 중간값인 derived metadata이다. 최소 구현은 확정 dealing range endpoint를 사용하고, Episode 22의 open-close body anchor는 별도 refinement policy로 추가한다. raw zone, touch interaction, FVG 또는 OB overlap, setup eligibility를 분리하며 Episode 41의 `62 to 70` 전사는 transcript conflict로 보존한다.
