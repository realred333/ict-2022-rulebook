# Liquidity Rulebook

공통 계약: [README.md](README.md)

## 목적

닫힌 OHLC와 확정 source만 사용해 stop 주문이 집중될 가능성이 있는 가격대를 `liquidity reference candidate`로 등록하고 추적한다. 결과는 BSL/SSL, sweep, daily bias, scanner 모듈이 공유하는 레지스트리로 저장한다.

실제 주문 수량과 주문 주체는 OHLC로 관측할 수 없다. 이 Rulebook은 주문 존재를 확정하지 않고 재현 가능한 가격 프록시만 정의한다.

## 객관적 정의

### Stop Pool Reference

기본 객체는 stop 기반 가격 reference이다.

```text
liquidity_reference = {
  id,
  symbol,
  timeframe,
  side,               # upper | lower
  source_kind,        # swing | equal_cluster | period_boundary | manual
  source_ids[],
  price_low,
  price_high,
  available_at,
  lifecycle_status,   # candidate | confirmed | invalidated | expired | manual_review
  availability,       # inactive | active | taken
  taken_at,
  reason_codes[],
  parameters_version
}
```

방향 규칙:

```text
confirmed swing_high -> upper liquidity candidate
confirmed swing_low  -> lower liquidity candidate
```

관통 규칙:

```text
upper_taken(level, bar) = bar.timestamp >= level.available_at
                          and bar.high > level.price_high

lower_taken(level, bar) = bar.timestamp >= level.available_at
                          and bar.low < level.price_low
```

동일 가격 touch는 별도 interaction으로 저장할 수 있지만 기본 `taken`은 strict inequality를 사용한다.

### Relative Equal Level Cluster

자막은 relative equal highs/lows를 반복하지만 고정 수치 tolerance를 제시하지 않는다. 다음 값은 백테스트 가능한 `research default`이다.

```text
equal_level_tolerance = max(1 * tick_size, 0.10 * ATR(14))
```

같은 symbol, timeframe, side에 속한 확정 source 집합 `S`가 다음을 만족하면 cluster를 만든다.

```text
equal_cluster(S) =
  count(S) >= 2
  and max(S.level) - min(S.level) <= equal_level_tolerance
```

transitive chaining으로 과도하게 넓은 cluster가 생기지 않도록 항상 전체 `max - min`을 검사한다.

### Period Boundary Reference

완료된 기간의 high와 low를 source로 등록할 수 있다.

```text
completed prior day high     -> upper candidate
completed prior day low      -> lower candidate
completed prior session high -> upper candidate
completed prior session low  -> lower candidate
completed overnight high     -> upper candidate
completed overnight low      -> lower candidate
```

진행 중인 기간의 최종 high/low를 미리 확정하지 않는다.

### Delivery Objective

Episode 2는 stops와 imbalance를 가격 전달 목표로 구분한다. Episode 3과 6은 range 내부 imbalance를 IRL 문맥에서 다룬다. 코드에서는 서로 다른 객체를 union으로 묶는다.

```text
delivery_objective =
  stop_pool_reference
  | imbalance_reference
```

`imbalance_reference`의 생성과 mitigation 규칙은 FVG Rulebook의 책임이다. liquidity registry는 이를 stop pool로 변환하지 않는다.

### Internal / External 위치 Tag

selected dealing range가 있을 때만 계산한다.

```text
external_upper(level, range) = level.price_high >= range.high
external_lower(level, range) = level.price_low <= range.low

internal_stop_pool(level, range) =
  range.low < level.price_low
  and level.price_high < range.high

internal_imbalance(zone, range) =
  zone.price_low > range.low
  and zone.price_high < range.high
```

range를 자동 선택할 수 없으면 `range_location = unclassified`와 `manual_review_reason`을 남긴다.

## 필수 조건

- 입력 OHLC에 `timestamp, symbol, timeframe, open, high, low, close`가 있다.
- OHLC는 symbol과 timeframe별로 오름차순 정렬된다.
- source는 look-ahead 없이 확정되었다.
- 모든 reference에 `side`, `price_low`, `price_high`, `available_at`, `source_kind`, `source_ids[]`가 있다.
- `price_low <= price_high`이다.
- single-price source는 `price_low == price_high`이다.
- equal cluster는 같은 symbol, timeframe, side의 source만 포함한다.
- equal cluster의 tolerance와 ATR 계산 시점을 기록한다.
- period boundary source는 기간 종료 이후에만 `confirmed`가 된다.
- `taken` 판정은 `available_at` 이후의 bar로만 수행한다.
- internal/external tag는 selected dealing range id와 함께 저장한다.

## 무효 조건

- 미래 swing 확인 봉을 미리 읽어 source를 생성한다.
- 진행 중인 일자나 세션의 최종 high/low를 확정 source처럼 사용한다.
- 다른 symbol 또는 timeframe의 source를 하나의 cluster로 합친다.
- upper와 lower source를 하나의 cluster로 합친다.
- equal cluster의 전체 폭이 tolerance보다 넓다.
- 가격 관통 이후의 `taken` level을 신규 active target으로 재사용한다.
- OHLC만 사용하면서 실제 stop 수량, 체결 주체, order book 상태를 출력한다.
- imbalance zone을 stop pool 객체로 저장한다.

## 예외 조건

- 동일 가격 touch: `touched_at` interaction으로 저장할 수 있지만 기본 `taken`으로 처리하지 않는다.
- source 중첩: swing high와 prior day high가 같은 가격대에 있으면 reference를 삭제하지 않고 `source_ids[]`와 `confluence_tags[]`를 병합한다.
- 이미 관통된 source의 지연 등록: `available_at` 이전 관통은 해당 source로 인한 taken event로 보지 않는다. 등록 시점 이후 target eligibility를 별도로 평가한다.
- equal cluster 재구성: 새 source가 추가되면 기존 cluster id를 유지하고 versioned membership history를 기록한다.
- ATR 결측: `tick_size`가 있으면 `1 * tick_size`만 사용한다. tick size도 없으면 `manual_review`로 보낸다.
- range 경계와 level 중첩: boundary는 external로 분류한다.
- expiry: 자막은 고정 만료 봉 수를 제시하지 않는다. 최소 구현은 자동 만료를 사용하지 않고, 고급 구현은 strategy parameter로 노출한다.
- 사람이 지정한 HTF level: 자동 source와 분리해 `source_kind = manual`과 사유를 기록한다.

## 상태 변화(State Machine)

Liquidity reference는 lifecycle과 target availability를 분리한다.

```text
lifecycle:
candidate -> confirmed | invalidated | expired | manual_review

availability:
inactive -> active -> taken
```

### candidate

다음 중 하나가 아직 확정되지 않은 상태이다.

- swing source가 오른쪽 확인 봉을 기다린다.
- equal cluster가 두 번째 확정 source를 기다린다.
- period boundary source가 기간 종료를 기다린다.
- selected dealing range가 없어 IRL/ERL 위치 판정을 기다린다.

candidate는 신규 거래 목표로 사용하지 않는다.

### confirmed

source가 look-ahead 없이 확정되고 필수 필드가 채워졌다. target으로 사용할 수 있으면 `availability = active`로 변경한다.

```text
swing source:
  available_at = swing.confirmed_at

equal cluster:
  available_at = max(member.confirmed_at)

period boundary:
  available_at = period.closed_at
```

### invalidated

candidate 또는 confirmed reference가 데이터 계약을 위반한 경우이다.

- upstream source가 look-ahead로 생성되었다.
- source OHLC가 정정되어 생성 조건을 더 이상 만족하지 않는다.
- cluster membership이 바뀌어 전체 폭이 tolerance를 초과한다.
- symbol, timeframe, side 혼합이 발견되었다.

가격이 level을 관통한 경우에는 `invalidated`가 아니라 `taken`을 사용한다.

### expired

reference가 strategy-specific expiry 정책을 넘겨 신규 목표에서 제외된 상태이다. 이벤트 이력은 삭제하지 않는다.

최소 구현의 기본값:

```text
expiry_policy = none
```

고급 구현에서는 timeframe, session, source 종류별 expiry를 parameter로 노출한다.

### manual_review

다음 중 하나이면 자동 결과와 함께 검토 사유를 남긴다.

- HTF narrative에서 어느 active level을 주요 draw로 선택할지 결정해야 한다.
- selected dealing range가 없어 IRL/ERL 위치를 분류할 수 없다.
- tick size와 ATR이 모두 없어 equal-level tolerance를 계산할 수 없다.
- session calendar가 없어 gap 또는 session boundary를 확정할 수 없다.
- manual HTF level이 자동 source와 충돌한다.

### taken

`taken`은 liquidity 전용 availability 종결 상태이다.

```text
active upper reference + later bar.high > price_high -> taken
active lower reference + later bar.low < price_low   -> taken
```

`taken`은 반전을 뜻하지 않는다. reclaim 여부는 Liquidity Sweep Rulebook, continuation 여부는 후행 구조 모듈에서 판정한다.

## 코드 구현 규칙

### 입력 데이터

```text
required:
  ohlc[]
  confirmed_swings[]

optional:
  tick_size
  atr_14[]
  completed_period_levels[]
  selected_dealing_ranges[]
  imbalance_references[]
  session_calendar
  strategy_expiry_policy
```

### 필요 변수

```text
reference_id
symbol
timeframe
side = upper | lower
source_kind = swing | equal_cluster | period_boundary | manual
source_ids[]
price_low
price_high
available_at
touched_at
taken_at
lifecycle_status
availability
range_id
range_location = internal_range | external_range | unclassified
objective_kind = stop_pool | imbalance
distance_ticks
reason_codes[]
manual_review_reason[]
parameters_version
```

### 조건식

```text
is_upper_source = swing.direction == high
is_lower_source = swing.direction == low

cluster_width = max(member.level) - min(member.level)
is_equal_cluster = member_count >= 2 and cluster_width <= tolerance

upper_taken = bar.timestamp >= available_at and bar.high > price_high
lower_taken = bar.timestamp >= available_at and bar.low < price_low

distance_ticks = abs(reference_price - current_price) / tick_size
```

### 의사코드

```text
for swing in confirmed_swings:
  register stop_pool_reference(
    side = upper if swing.direction == high else lower,
    source_kind = swing,
    source_ids = [swing.id],
    price_low = swing.level,
    price_high = swing.level,
    available_at = swing.confirmed_at,
    lifecycle_status = confirmed,
    availability = active
  )

for each symbol, timeframe, side:
  tolerance = max(tick_size, 0.10 * ATR(14))
  clusters = cluster_confirmed_sources_without_transitive_overflow(tolerance)
  for cluster in clusters where count(cluster.members) >= 2:
    register_or_update equal_cluster reference(
      price_low = min(cluster.member_levels),
      price_high = max(cluster.member_levels),
      available_at = max(cluster.member_confirmed_at)
    )

for period in completed_periods:
  register upper period_boundary at period.high after period.closed_at
  register lower period_boundary at period.low after period.closed_at

for bar in closed_bars_after(reference.available_at):
  if reference.side == upper and bar.high > reference.price_high:
    set availability = taken
    set taken_at = bar.timestamp
  if reference.side == lower and bar.low < reference.price_low:
    set availability = taken
    set taken_at = bar.timestamp

for objective in active_references:
  if selected_dealing_range exists:
    attach internal_range or external_range tag
  else:
    attach unclassified tag
```

## 최소 구현 버전

OHLC, confirmed short-term swing, tick size만 사용한다.

- confirmed swing high를 active upper reference로 등록한다.
- confirmed swing low를 active lower reference로 등록한다.
- strict traversal로 `active -> taken`을 기록한다.
- symbol과 timeframe별 레지스트리를 분리한다.
- 과거 reference를 삭제하지 않고 상태 이력을 저장한다.
- 자동 expiry, cluster, session, IRL/ERL ranking은 사용하지 않는다.

최소 출력:

```text
id, symbol, timeframe, side, source_kind, source_ids,
price_low, price_high, available_at, availability, taken_at,
reason_codes, parameters_version
```

## 고급 구현 버전

### HTF

- timeframe별 registry를 독립 계산한다.
- LTF 후보에 가까운 active HTF reference id를 연결한다.
- HTF draw 하나를 자동 확정하지 않고 ranking feature를 출력한다.
- distance, timeframe, source_kind, confluence count를 feature로 저장한다.

### 세션

- `America/New_York` timezone과 DST-aware calendar를 사용한다.
- 완료된 prior day, London, overnight, configured session high/low를 source로 등록한다.
- 진행 중인 세션 level은 `developing` tag로 분리하고 completed source와 혼동하지 않는다.

### 유동성

- equal highs/lows cluster를 추가한다.
- selected dealing range가 있을 때 IRL/ERL tag를 추가한다.
- FVG module의 imbalance reference를 `delivery_objective` 목록에 join한다.
- taken event를 sweep과 breakout classifier에 전달한다.

### SMT

- correlated symbol별 active reference와 taken event를 timestamp 정렬한다.
- 한 심볼만 prior high/low를 관통한 경우 SMT 모듈에 divergence candidate를 전달한다.
- liquidity module 자체는 SMT를 확정하지 않는다.

### 경제지표

- calendar event 전후 configured window 안에서 생성되거나 taken된 reference에 event metadata를 붙인다.
- event metadata는 level 정의를 바꾸지 않는다.
- 필터 적용 여부는 strategy parameter로 기록한다.

## False Signal 제거 방법

- actual order와 OHLC proxy를 구분한다.
- candidate source를 active target으로 사용하지 않는다.
- swing `pivot_at`이 아니라 `confirmed_at` 이후에만 reference를 활성화한다.
- 완료되지 않은 일자와 세션의 high/low를 확정 source로 사용하지 않는다.
- equal-level tolerance를 versioned parameter로 저장한다.
- cluster는 pairwise chaining이 아니라 전체 폭으로 검증한다.
- upper와 lower, symbol, timeframe을 섞지 않는다.
- 동일 가격 touch와 strict traversal을 구분한다.
- `taken` level을 신규 target에서 제외한다.
- stop pool과 imbalance zone을 같은 객체로 합치지 않는다.
- liquidity taken만으로 reversal을 확정하지 않는다.
- IRL/ERL tag가 없다는 이유로 source 자체를 삭제하지 않는다.

## Pine Script 구현 가이드

Pine Script에서는 제한된 레지스트리를 배열로 관리한다.

```text
on confirmed bar:
  if confirmed swing_high[1]:
    push upper reference(level = high[1], available_at = time)
  if confirmed swing_low[1]:
    push lower reference(level = low[1], available_at = time)

for active reference in arrays:
  if side == upper and high > price_high:
    mark taken
  if side == lower and low < price_low:
    mark taken
```

- swing marker의 pivot 시각과 reference의 사용 가능 시각을 분리한다.
- `barstate.isconfirmed`에서만 taken 상태를 확정한다.
- 배열 크기 제한 때문에 오래된 taken reference의 시각화만 정리하고 event 통계는 별도 집계한다.
- equal cluster tolerance, 최대 보관 개수, period source 사용 여부를 input으로 노출한다.
- IRL/ERL은 사용자가 선택한 dealing range 또는 별도 range module 출력이 있을 때만 표시한다.

## Python Scanner 구현 가이드

event table과 current snapshot을 분리한다.

```text
liquidity_reference_history
liquidity_reference_snapshot
liquidity_taken_events
delivery_objective_snapshot
```

- `groupby(symbol, timeframe)` 안에서 source를 생성한다.
- swing table의 `confirmed_at`을 reference `available_at`으로 사용한다.
- equal cluster는 interval 또는 sweep-line 방식으로 계산하되 전체 cluster 폭을 재검증한다.
- completed period table은 timezone-aware calendar로 생성한다.
- taken event는 최초 관통 bar를 기록한다.
- 동일 source가 swing, prior day, equal cluster에 중복 포함되면 source id를 보존한다.
- snapshot에는 active reference만, history에는 confirmed, invalidated, expired, taken을 모두 남긴다.
- ranking model을 추가할 경우 feature와 strategy version을 함께 저장한다.

## 백테스트 시 주의사항

- swing 중심 봉 시각에 liquidity source를 알고 있었다고 가정하지 않는다.
- prior day high/low는 다음 거래일 시작 이후, session high/low는 해당 session 종료 이후에만 확정 source로 사용한다.
- ATR tolerance는 해당 시점까지 이용 가능한 데이터로만 계산한다.
- touch와 traversal, traversal과 reclaim, reclaim과 reversal을 각각 분리한다.
- 같은 level을 여러 source가 지지할 때 중복 target으로 수익을 이중 계산하지 않는다.
- expiry 정책이 없으면 오래된 level이 계속 남는다. 성능 비교 시 expiry parameter를 명시한다.
- timezone, DST, 거래소 휴장, session calendar를 기록한다.
- `draw on liquidity` ranking은 수익성 가설이다. rulebook 기본 registry와 분리해 검증한다.
- actual stop 수량을 알고 있었다고 가정하는 분석을 OHLC 백테스트에 포함하지 않는다.

## 최종 Rulebook 정의

Liquidity reference는 look-ahead 없이 확정된 swing, equal-level cluster 또는 완료된 기간 경계에서 생성되며, 실제 주문 수량을 확정하지 않고 stop 집중 가능성이 있는 상단 또는 하단 가격대를 나타내는 versioned OHLC 프록시이다. reference는 사용 가능 시점 이후 strict traversal이 발생하면 `active -> taken`으로 변경되고, 반전 여부는 후행 sweep, displacement, MSS 규칙에서 별도로 판정한다.
