# BSL / SSL Rulebook

공통 계약: [README.md](README.md)

## 목적

[Liquidity Rulebook](liquidity_rulebook.md)이 생성한 stop pool reference를 `BSL | SSL`로 손실 없이 분류한다. 분류 결과는 directional target 목록과 taken event를 생성하고, Liquidity Sweep, Daily Bias, SMT, scanner 모듈에 전달한다.

이 Rulebook은 새로운 liquidity source를 만들지 않는다. 실제 stop 주문 수량을 추정하지 않으며, 진입 방향이나 반전을 확정하지 않는다.

## 객관적 정의

### 분류 계약

```text
classify(reference):
  if reference.side == upper:
    liquidity_side = BSL
    assumed_stop_side = buy_stop
    trigger_price = reference.price_high

  if reference.side == lower:
    liquidity_side = SSL
    assumed_stop_side = sell_stop
    trigger_price = reference.price_low
```

- BSL: high 기반 upper liquidity reference 위에 buy stops가 존재할 가능성을 나타내는 label
- SSL: low 기반 lower liquidity reference 아래에 sell stops가 존재할 가능성을 나타내는 label

실제 주문은 OHLC로 관측할 수 없다. `assumed_stop_side`는 설명 모델이며 order book 사실값이 아니다.

### 출력 객체

```text
bsl_ssl_reference = {
  id,
  liquidity_reference_id,
  symbol,
  timeframe,
  liquidity_side,       # BSL | SSL
  assumed_stop_side,    # buy_stop | sell_stop
  source_kind,
  source_ids[],
  price_low,
  price_high,
  trigger_price,
  available_at,
  lifecycle_status,     # candidate | confirmed | invalidated | expired | manual_review
  availability,         # inactive | active | taken
  target_eligibility,   # eligible | ineligible | manual_review
  taken_at,
  distance_ticks,
  distance_atr,
  reason_codes[],
  parameters_version
}
```

### Taken Event

Liquidity Rulebook의 strict traversal을 BSL/SSL 의미로 변환한다.

```text
bsl_taken(reference, bar) =
  reference.liquidity_side == BSL
  and reference.availability == active
  and bar.timestamp >= reference.available_at
  and bar.high > reference.trigger_price

ssl_taken(reference, bar) =
  reference.liquidity_side == SSL
  and reference.availability == active
  and bar.timestamp >= reference.available_at
  and bar.low < reference.trigger_price
```

동일 가격 touch는 기본 taken event가 아니다.

### Directional Target Eligibility

BSL/SSL label과 현재 시점의 target 자격을 분리한다.

```text
bsl_target_eligible(reference, close) =
  reference.liquidity_side == BSL
  and reference.availability == active
  and reference.trigger_price > close

ssl_target_eligible(reference, close) =
  reference.liquidity_side == SSL
  and reference.availability == active
  and reference.trigger_price < close
```

분류 label은 immutable이다. 현재 가격이 바뀌면 `target_eligibility`만 다시 계산한다.

### Distance

```text
distance_ticks = abs(trigger_price - current_close) / tick_size
distance_atr   = abs(trigger_price - current_close) / ATR(14)
```

ATR이 없으면 `distance_atr = null`로 유지한다.

### 반대편 후보 목록

한쪽 liquidity가 taken되면 반대편 active target을 조회할 수 있다.

```text
taken BSL -> query eligible SSL references below current price
taken SSL -> query eligible BSL references above current price
```

이는 후보 조회 규칙이다. 가격이 반드시 반대편 목표까지 이동한다는 예측 규칙이 아니다.

## 필수 조건

- upstream liquidity reference가 존재한다.
- upstream lifecycle이 `confirmed`이다.
- upstream reference가 look-ahead 없이 생성되었다.
- `symbol`, `timeframe`, `side`, `price_low`, `price_high`, `available_at`, `availability`가 존재한다.
- `side`는 `upper | lower` 중 하나이다.
- `price_low <= price_high`이다.
- upper reference는 BSL로만 분류한다.
- lower reference는 SSL로만 분류한다.
- BSL trigger는 `price_high`, SSL trigger는 `price_low`를 사용한다.
- target 목록에는 `availability = active`이고 현재 가격 방향 조건을 만족하는 reference만 포함한다.
- taken event는 source `available_at` 이후의 닫힌 bar로만 확정한다.
- 모든 결과에 upstream `liquidity_reference_id`를 보존한다.

## 무효 조건

- upstream reference 없이 BSL/SSL 객체를 독립 생성한다.
- `side = upper`를 SSL, `side = lower`를 BSL로 분류한다.
- 아직 확정되지 않은 swing 또는 period boundary를 active BSL/SSL로 사용한다.
- 이미 `taken` 또는 `expired`인 reference를 신규 directional target으로 사용한다.
- BSL에 `price_low`, SSL에 `price_high`를 traversal trigger로 사용하여 cluster zone 일부 touch를 taken으로 오분류한다.
- 동일 가격 touch를 strict traversal로 처리한다.
- target eligibility와 immutable BSL/SSL label을 혼동하여 label 자체를 뒤집는다.
- BSL을 long entry, SSL을 short entry로 자동 변환한다.
- taken event만으로 sweep 또는 reversal을 확정한다.
- OHLC만 사용하면서 실제 stop 주문 수량이나 주문 주체를 출력한다.

## 예외 조건

- single-price source: `price_low == price_high`이므로 BSL/SSL trigger도 같은 가격이다.
- equal-level cluster: BSL은 cluster 상단 `price_high`, SSL은 cluster 하단 `price_low`를 strict traversal trigger로 사용한다.
- source 중첩: swing high와 prior day high가 같은 가격대에 있으면 하나를 삭제하지 않고 upstream `source_ids[]`와 `confluence_tags[]`를 유지한다.
- 현재 가격 반대편에 남은 active reference: label은 유지하고 `target_eligibility = ineligible`로 둔다. 관통 event 누락 여부는 데이터 검증 대상으로 기록한다.
- 지연 수신 또는 정정 데이터: upstream version이 바뀌면 downstream BSL/SSL snapshot을 재계산하고 history에 version을 남긴다.
- ATR 결측: tick distance만 계산한다.
- tick size 결측: distance tick 계산을 생략하고 `manual_review_reason = missing_tick_size`를 남긴다.
- manual HTF reference: upstream `source_kind = manual`을 유지하고 자동 source와 분리한다.
- expiry: upstream Liquidity Rulebook의 strategy-specific expiry를 그대로 전달한다.

## 상태 변화(State Machine)

BSL/SSL은 upstream lifecycle과 availability를 보존하고, target 자격을 별도 계산한다.

```text
lifecycle:
candidate -> confirmed | invalidated | expired | manual_review

availability:
inactive -> active -> taken

target_eligibility:
eligible | ineligible | manual_review
```

### candidate

다음 조건 중 하나이면 candidate이다.

- upstream liquidity reference가 candidate이다.
- upstream side가 아직 확정되지 않았다.
- period boundary가 닫히지 않았다.
- manual source의 upper/lower 방향 검토가 끝나지 않았다.

candidate는 directional target 목록에 넣지 않는다.

### confirmed

upstream reference가 confirmed이고 `side`, zone, `available_at`이 유효하면 분류를 확정한다.

```text
upper -> BSL
lower -> SSL
```

upstream availability가 active이면 현재 가격 기준 `target_eligibility`를 계산한다.

### invalidated

다음 중 하나이면 invalidated이다.

- upstream reference가 invalidated되었다.
- upstream side와 BSL/SSL label이 충돌한다.
- upstream zone이 `price_low > price_high`이다.
- look-ahead 또는 데이터 정정으로 source 계약이 깨졌다.
- equal cluster 재구성으로 기존 downstream version이 더 이상 유효하지 않다.

가격 관통은 invalidated가 아니라 `taken`이다.

### expired

upstream reference가 expiry 정책에 따라 expired되면 BSL/SSL reference도 expired된다. 과거 event는 삭제하지 않는다.

최소 구현:

```text
expiry_policy = upstream liquidity expiry policy
```

### manual_review

다음 중 하나이면 자동 결과와 함께 검토 사유를 남긴다.

- manual HTF source의 upper/lower 방향이 명시되지 않았다.
- tick size가 없어 distance tick을 계산할 수 없다.
- 현재 가격 반대편에 active reference가 남아 traversal event 누락 가능성이 있다.
- upstream source version 충돌을 자동 병합할 수 없다.
- 복수 BSL/SSL 중 주요 draw 하나를 선택해야 한다.

### taken

taken은 availability 종결 상태이다.

```text
active BSL + later bar.high > trigger_price -> taken BSL
active SSL + later bar.low < trigger_price  -> taken SSL
```

다음 event를 생성한다.

```text
bsl_ssl_taken_event = {
  reference_id,
  liquidity_reference_id,
  symbol,
  timeframe,
  liquidity_side,
  trigger_price,
  taken_at,
  traversal_price,
  traversal_ticks,
  source_kind,
  reason_codes[]
}
```

taken event는 Liquidity Sweep Rulebook으로 전달한다. BSL/SSL 모듈은 reclaim과 reversal을 확정하지 않는다.

## 코드 구현 규칙

### 입력 데이터

```text
required:
  liquidity_reference_history[]
  liquidity_reference_snapshot[]
  closed_ohlc[]
  current_close

optional:
  tick_size
  atr_14
  htf_reference_links[]
  session_metadata
  calendar_metadata
  correlated_symbol_references[]
```

### 필요 변수

```text
reference_id
liquidity_reference_id
symbol
timeframe
liquidity_side = BSL | SSL
assumed_stop_side = buy_stop | sell_stop
source_kind
source_ids[]
price_low
price_high
trigger_price
available_at
lifecycle_status
availability
target_eligibility
taken_at
traversal_price
traversal_ticks
distance_ticks
distance_atr
reason_codes[]
manual_review_reason[]
parameters_version
```

### 조건식

```text
is_bsl = reference.side == upper
is_ssl = reference.side == lower

bsl_trigger = reference.price_high
ssl_trigger = reference.price_low

bsl_taken = is_bsl and availability == active and bar.high > bsl_trigger
ssl_taken = is_ssl and availability == active and bar.low < ssl_trigger

bsl_target_eligible = is_bsl and availability == active and bsl_trigger > close
ssl_target_eligible = is_ssl and availability == active and ssl_trigger < close

traversal_ticks =
  (bar.high - bsl_trigger) / tick_size if is_bsl
  (ssl_trigger - bar.low) / tick_size if is_ssl
```

### 의사코드

```text
for liquidity_reference in liquidity_reference_snapshot:
  if liquidity_reference.lifecycle_status != confirmed:
    skip target classification

  if liquidity_reference.side == upper:
    emit_or_update BSL(
      liquidity_reference_id = liquidity_reference.id,
      trigger_price = liquidity_reference.price_high,
      assumed_stop_side = buy_stop
    )

  else if liquidity_reference.side == lower:
    emit_or_update SSL(
      liquidity_reference_id = liquidity_reference.id,
      trigger_price = liquidity_reference.price_low,
      assumed_stop_side = sell_stop
    )

  else:
    emit manual_review(reason = missing_or_invalid_side)

for closed_bar in bars_after(reference.available_at):
  if reference is active BSL and closed_bar.high > reference.trigger_price:
    mark reference taken
    emit taken BSL event

  if reference is active SSL and closed_bar.low < reference.trigger_price:
    mark reference taken
    emit taken SSL event

for active_reference in bsl_ssl_snapshot:
  recompute target_eligibility using current_close
  compute distance_ticks and distance_atr

for taken_event in new_taken_events:
  send event to sweep_classifier
  query opposite-side eligible targets
```

## 최소 구현 버전

OHLC와 upstream confirmed swing liquidity만 사용한다.

- upper swing reference를 BSL로 분류한다.
- lower swing reference를 SSL로 분류한다.
- BSL trigger는 swing high, SSL trigger는 swing low이다.
- strict traversal로 `active -> taken`을 기록한다.
- 현재 close 위 active BSL과 현재 close 아래 active SSL 목록을 출력한다.
- target까지 tick distance를 계산한다.
- taken event를 sweep classifier에 전달한다.
- label을 entry signal로 변환하지 않는다.

최소 출력:

```text
id, liquidity_reference_id, symbol, timeframe,
liquidity_side, assumed_stop_side, source_kind,
trigger_price, available_at, availability,
target_eligibility, taken_at, distance_ticks,
reason_codes, parameters_version
```

## 고급 구현 버전

### HTF

- timeframe별 BSL/SSL snapshot을 독립 유지한다.
- LTF candidate에 가까운 active HTF BSL/SSL id를 연결한다.
- `distance_ticks`, `distance_atr`, timeframe, source_kind, confluence count를 ranking feature로 출력한다.
- HTF draw 하나를 자동 확정하지 않는다.

### 세션

- 완료된 prior day, London, overnight, configured session high/low source를 분류한다.
- `America/New_York` timezone과 DST-aware calendar를 사용한다.
- developing session high/low는 confirmed target과 분리한다.
- killzone 안 taken event에 session metadata를 붙인다.

### 유동성

- relative equal highs BSL cluster와 relative equal lows SSL cluster를 분류한다.
- taken BSL 뒤 eligible SSL, taken SSL 뒤 eligible BSL 후보 목록을 출력한다.
- IRL/ERL tag와 source hierarchy를 upstream에서 전달받는다.
- sweep, breakout, continuation 판정은 후행 모듈에 위임한다.

### SMT

- correlated symbol별 BSL/SSL taken event를 timestamp 정렬한다.
- 한 심볼만 comparable BSL 또는 SSL을 회수한 경우 SMT candidate를 생성한다.
- BSL/SSL 모듈 자체는 divergence를 확정하지 않는다.

### 경제지표

- event window 안에서 발생한 taken BSL/SSL에 calendar metadata를 붙인다.
- 경제지표는 BSL/SSL 분류 규칙을 바꾸지 않는다.
- 필터 사용 여부와 window 크기는 strategy parameter로 노출한다.

## False Signal 제거 방법

- upstream liquidity reference 없이 BSL/SSL을 만들지 않는다.
- upper만 BSL, lower만 SSL로 분류한다.
- source label과 trade direction을 분리한다.
- candidate source를 active target으로 사용하지 않는다.
- BSL trigger는 cluster 상단, SSL trigger는 cluster 하단을 사용한다.
- 동일 가격 touch와 strict traversal을 구분한다.
- target eligibility와 immutable label을 분리한다.
- taken reference를 신규 target에서 제외한다.
- taken event만으로 sweep과 reversal을 확정하지 않는다.
- 현재 가격 반대편 active reference는 event 누락 여부를 검증한다.
- distance와 source metadata 없이 주요 draw를 자동 선택하지 않는다.
- actual stop 주문 수량을 OHLC에서 추정한 사실값처럼 출력하지 않는다.

## Pine Script 구현 가이드

Pine Script에서는 Liquidity Rulebook 배열의 side를 BSL/SSL label로 변환한다.

```text
for active liquidity reference:
  if side == upper:
    liquidity_side = BSL
    trigger = price_high
    target_eligible = trigger > close
    taken = high > trigger

  if side == lower:
    liquidity_side = SSL
    trigger = price_low
    target_eligible = trigger < close
    taken = low < trigger
```

- `barstate.isconfirmed`에서 taken event를 확정한다.
- BSL과 SSL은 서로 다른 색상과 배열로 표시한다.
- active target, taken history, ineligible reference를 시각적으로 구분한다.
- marker에는 source kind와 distance tick을 표시할 수 있다.
- alert는 `BSL taken`, `SSL taken`까지만 발생시키고 sweep alert와 분리한다.
- 배열 제한 때문에 오래된 taken line의 시각화는 정리할 수 있지만 event count는 별도 유지한다.

## Python Scanner 구현 가이드

분류 snapshot과 taken event table을 분리한다.

```text
bsl_ssl_reference_history
bsl_ssl_reference_snapshot
bsl_ssl_taken_events
bsl_ssl_opposite_target_candidates
```

- upstream `liquidity_reference_id`를 foreign key로 보존한다.
- `groupby(symbol, timeframe)` 안에서 분류와 traversal을 계산한다.
- 현재 close마다 target eligibility와 distance를 재계산한다.
- taken event는 최초 strict traversal bar만 기록한다.
- source zone이 cluster이면 BSL은 `price_high`, SSL은 `price_low`를 trigger로 사용한다.
- current snapshot에는 active reference를, history에는 taken, expired, invalidated를 포함한다.
- upstream parameters version과 BSL/SSL classifier version을 함께 저장한다.
- 주요 draw ranking을 추가하면 feature, model version, 선택 근거를 별도 table에 기록한다.

## 백테스트 시 주의사항

- swing `pivot_at`이 아니라 upstream `available_at` 이후에만 BSL/SSL을 사용한다.
- 현재 봉 high/low로 taken을 판정하고 같은 봉 내부에서 reclaim, 진입, 손절 순서를 임의로 가정하지 않는다.
- 동일 가격 touch와 strict traversal을 분리한다.
- cluster zone의 어느 경계를 trigger로 사용했는지 기록한다.
- taken BSL과 bearish reversal, taken SSL과 bullish reversal을 동일 사건으로 취급하지 않는다.
- 반대편 target 후보가 있다는 이유만으로 가격 도달을 보장하지 않는다.
- 복수 BSL/SSL 중 가장 가까운 level만 선택하는 정책은 별도 전략으로 백테스트한다.
- period boundary source는 해당 기간 종료 이후에만 사용한다.
- timezone, DST, session calendar, tick size, ATR 계산 정책을 기록한다.
- source 중첩을 중복 수익으로 계산하지 않는다.
- actual stop 주문 수량을 알고 있었다고 가정하지 않는다.

## 최종 Rulebook 정의

BSL/SSL은 look-ahead 없이 확정된 liquidity reference의 `upper | lower` 위치를 각각 `BSL | SSL`로 변환하는 immutable 의미 label이다. BSL은 상단 buy-stop 후보, SSL은 하단 sell-stop 후보를 나타낸다. 현재 가격 방향 조건을 만족하는 active reference만 directional target이 되며, strict traversal이 발생하면 `taken` event를 생성한다. taken event는 반전 확정이 아니므로 sweep, displacement, MSS 판정은 후행 모듈에서 수행한다.
