# Liquidity Sweep Rulebook

공통 계약: [README.md](README.md)

## 목적

[BSL / SSL Rulebook](bsl_ssl_rulebook.md)이 생성한 최초 `taken BSL | taken SSL` event를 입력으로 받아, 관통 뒤 range 안쪽 reclaim이 발생했는지 추적한다. reclaim이 확인되면 liquidity sweep으로 확정하고, reclaim 없이 관통 방향 확장이 이어지면 `run_or_breakout_candidate`로 분리한다.

실제 stop 체결 수량과 주문 주체는 OHLC로 확인할 수 없다. 이 Rulebook은 stop hunt를 직접 증명하지 않고 `strict traversal + reclaim` 가격 프록시를 정의한다.

## 객관적 정의

### Sweep Candidate

BSL/SSL taken event가 발생하면 sweep candidate를 연다.

```text
sweep_candidate = {
  id,
  bsl_ssl_reference_id,
  taken_event_id,
  liquidity_reference_id,
  symbol,
  timeframe,
  swept_side,          # BSL | SSL
  expected_reversal,   # bearish | bullish
  trigger_price,
  traversal_at,
  traversal_price,
  sweep_extreme,
  lifecycle_status,    # candidate | confirmed | invalidated | expired | manual_review
  classification,      # pending | sweep | run_or_breakout_candidate
  reclaimed_at,
  bars_to_reclaim,
  traversal_ticks,
  traversal_atr,
  reason_codes[],
  parameters_version
}
```

방향 매핑:

```text
taken BSL -> BSL sweep candidate -> expected_reversal = bearish
taken SSL -> SSL sweep candidate -> expected_reversal = bullish
```

`expected_reversal`은 후행 확인 방향이다. 진입 신호 또는 반전 확정값이 아니다.

### Strict Traversal

upstream BSL/SSL Rulebook의 taken event를 재사용한다.

```text
bsl_traversal =
  prior availability == active
  and bar.high > bsl.trigger_price

ssl_traversal =
  prior availability == active
  and bar.low < ssl.trigger_price
```

동일 가격 touch는 traversal이 아니다.

### Reclaim

```text
bsl_reclaim =
  swept_side == BSL
  and close < trigger_price

ssl_reclaim =
  swept_side == SSL
  and close > trigger_price
```

traversal bar 자체 종가가 reclaim 조건을 만족하면 `bars_to_reclaim = 0`으로 확정한다.

### Sweep Confirmation

```text
confirmed_bsl_sweep =
  taken BSL event
  and exists closed bar j in [traversal_bar, traversal_bar + sweep_reclaim_bars]
      where close[j] < trigger_price

confirmed_ssl_sweep =
  taken SSL event
  and exists closed bar j in [traversal_bar, traversal_bar + sweep_reclaim_bars]
      where close[j] > trigger_price
```

기본값:

```text
sweep_reclaim_bars = 3  # research default
```

자막은 고정 reclaim 봉 수를 제시하지 않는다. 이 값은 백테스트 가능한 초기 파라미터이다.

### Run 또는 Breakout 후보

```text
run_or_breakout_candidate =
  taken event
  and no reclaim within sweep_reclaim_bars
```

Episode 23은 sweep을 range 안쪽으로 reverse하는 움직임, run을 level을 지나 continue하는 움직임으로 구분한다. 최소 구현은 reclaim 부재를 지속형 후보로 표시한다. 최종 breakout 구조 판정은 후행 전략 또는 별도 classifier 책임이다.

### Sweep 폭

```text
bsl_traversal_ticks = (sweep_extreme - trigger_price) / tick_size
ssl_traversal_ticks = (trigger_price - sweep_extreme) / tick_size

traversal_atr = abs(sweep_extreme - trigger_price) / ATR(14)
```

candidate가 열린 동안 extreme을 갱신한다.

## 필수 조건

- upstream `bsl_ssl_taken_event`가 존재한다.
- upstream reference가 traversal 직전 `active`였다.
- upstream reference는 look-ahead 없이 확정되었다.
- `taken_event_id`, `bsl_ssl_reference_id`, `liquidity_reference_id`를 보존한다.
- `symbol`, `timeframe`, `swept_side`, `trigger_price`, `traversal_at`이 존재한다.
- `swept_side`는 `BSL | SSL` 중 하나이다.
- BSL reclaim은 `close < trigger_price`이다.
- SSL reclaim은 `close > trigger_price`이다.
- reclaim은 닫힌 봉으로만 확정한다.
- traversal bar부터 configured reclaim window를 계산한다.
- `sweep_reclaim_bars`와 parameters version을 저장한다.
- same-bar reclaim을 허용하고 `bars_to_reclaim = 0`으로 기록한다.

## 무효 조건

- upstream taken event 없이 sweep candidate를 독립 생성한다.
- 동일 가격 touch를 strict traversal로 처리한다.
- 이미 taken된 같은 reference의 반복 관통을 신규 sweep으로 중복 생성한다.
- BSL과 SSL reclaim 방향을 뒤집는다.
- reclaim이 없는데 confirmed sweep으로 처리한다.
- 미래 reclaim 봉을 미리 읽어 traversal 시점에 sweep을 확정한다.
- `sweep_reclaim_bars = 3`을 ICT 자막의 고정 규칙이라고 설명한다.
- confirmed sweep을 reversal 또는 entry와 동일하게 처리한다.
- OHLC만으로 stop 체결 수량이나 주문 주체를 출력한다.

## 예외 조건

- same-bar reclaim: OHLC만으로 봉 내부 traversal과 reclaim 순서를 완전히 재구성할 수 없다. `intrabar_order = unknown` metadata를 남긴다.
- gap traversal: bar open이 trigger 바깥에서 시작하면 `gap_traversal = true`를 기록한다. 일반 wick sweep과 분리해 분석한다.
- cluster zone: BSL trigger는 zone 상단, SSL trigger는 zone 하단을 upstream에서 전달받는다.
- 복수 reference 동시 traversal: reference별 candidate를 생성하되 동일 bar의 parent traversal group id를 공유한다.
- candidate 진행 중 extreme 갱신: BSL은 최대 high, SSL은 최소 low를 저장한다.
- upstream 정정: source version이 바뀌면 candidate를 invalidated하고 재계산한다.
- tick size 결측: traversal tick 계산을 생략하고 `manual_review_reason = missing_tick_size`를 남긴다.
- ATR 결측: traversal ATR 계산을 생략한다.
- HTF level과 LTF level 중첩: reference별 event를 보존하고 confluence metadata로 묶는다.
- window 만료 뒤 뒤늦은 reclaim: expired candidate 이력은 유지하고 `late_reclaim` interaction으로 별도 저장한다.

## 상태 변화(State Machine)

```text
candidate -> confirmed | invalidated | expired | manual_review

classification:
pending -> sweep | run_or_breakout_candidate
```

### candidate

최초 taken event를 수신하면 candidate를 연다.

```text
candidate.opened_at = taken_event.taken_at
candidate.trigger_price = taken_event.trigger_price
candidate.sweep_extreme = taken_event.traversal_price
candidate.classification = pending
```

candidate가 열린 동안:

- reclaim 조건을 검사한다.
- sweep extreme을 갱신한다.
- window 안의 닫힌 봉 수를 센다.

### confirmed

configured window 안에 reclaim이 발생하면 confirmed이다.

```text
BSL candidate + close < trigger_price -> confirmed sweep
SSL candidate + close > trigger_price -> confirmed sweep
```

다음을 저장한다.

```text
classification = sweep
reclaimed_at
bars_to_reclaim
sweep_extreme
traversal_ticks
traversal_atr
```

### invalidated

다음 중 하나이면 invalidated이다.

- upstream BSL/SSL reference가 데이터 정정으로 invalidated되었다.
- taken event가 look-ahead 또는 side 매핑 오류로 생성되었다.
- trigger price가 upstream version과 일치하지 않는다.
- symbol 또는 timeframe 혼합이 발견되었다.

가격이 계속 확장한 경우에는 invalidated가 아니라 expired와 `run_or_breakout_candidate`를 사용한다.

### expired

window 안에 reclaim이 없으면 expired이다.

```text
bars_since_traversal > sweep_reclaim_bars
and no reclaim
-> lifecycle_status = expired
-> classification = run_or_breakout_candidate
```

expired event는 삭제하지 않는다.

### manual_review

다음 중 하나이면 자동 결과와 함께 검토 사유를 남긴다.

- same-bar reclaim의 실제 intrabar 순서가 중요하다.
- gap traversal을 일반 sweep과 같은 전략으로 처리할지 결정해야 한다.
- 복수 HTF/LTF sweep 중 narrative상 핵심 사건을 선택해야 한다.
- tick size와 ATR이 모두 없어 sweep 폭을 계산할 수 없다.
- late reclaim을 별도 전략에서 허용할지 결정해야 한다.

## 코드 구현 규칙

### 입력 데이터

```text
required:
  bsl_ssl_taken_events[]
  bsl_ssl_reference_history[]
  closed_ohlc[]
  sweep_reclaim_bars

optional:
  tick_size
  atr_14[]
  htf_reference_links[]
  session_metadata
  calendar_metadata
  correlated_symbol_taken_events[]
```

### 필요 변수

```text
sweep_id
taken_event_id
bsl_ssl_reference_id
liquidity_reference_id
symbol
timeframe
swept_side = BSL | SSL
expected_reversal = bearish | bullish
trigger_price
traversal_at
traversal_price
sweep_extreme
reclaimed_at
bars_to_reclaim
traversal_ticks
traversal_atr
gap_traversal
intrabar_order
lifecycle_status
classification
reason_codes[]
manual_review_reason[]
parameters_version
```

### 조건식

```text
is_bsl_candidate = taken_event.liquidity_side == BSL
is_ssl_candidate = taken_event.liquidity_side == SSL

is_bsl_reclaim = close < trigger_price
is_ssl_reclaim = close > trigger_price

within_window = bars_since_traversal <= sweep_reclaim_bars

confirm_bsl_sweep = is_bsl_candidate and within_window and is_bsl_reclaim
confirm_ssl_sweep = is_ssl_candidate and within_window and is_ssl_reclaim

expire_to_run_candidate = not reclaimed and bars_since_traversal > sweep_reclaim_bars
```

### 의사코드

```text
for taken_event in new_bsl_ssl_taken_events:
  open sweep_candidate(
    taken_event_id = taken_event.id,
    swept_side = taken_event.liquidity_side,
    expected_reversal = bearish if BSL else bullish,
    trigger_price = taken_event.trigger_price,
    traversal_at = taken_event.taken_at,
    traversal_price = taken_event.traversal_price,
    sweep_extreme = taken_event.traversal_price,
    lifecycle_status = candidate,
    classification = pending
  )

for candidate in open_candidates:
  for closed_bar from traversal_bar through reclaim_window:
    update sweep_extreme

    if candidate is BSL and closed_bar.close < trigger_price:
      confirm sweep(candidate, reclaimed_at = closed_bar.timestamp)
      break

    if candidate is SSL and closed_bar.close > trigger_price:
      confirm sweep(candidate, reclaimed_at = closed_bar.timestamp)
      break

  if no reclaim and bars_since_traversal > sweep_reclaim_bars:
    expire candidate
    set classification = run_or_breakout_candidate

for confirmed_sweep in new_sweeps:
  send sweep to displacement and MSS context join
  query opposite-side active targets
```

## 최소 구현 버전

OHLC와 BSL/SSL taken event만 사용한다.

- 최초 strict traversal event마다 candidate를 연다.
- same-bar reclaim을 포함한다.
- 기본 `sweep_reclaim_bars = 3`을 사용한다.
- BSL은 trigger 아래 close, SSL은 trigger 위 close로 reclaim한다.
- confirmed sweep과 expired run candidate를 분리한다.
- traversal tick, bars to reclaim, sweep extreme을 저장한다.
- sweep을 entry signal로 변환하지 않는다.

최소 출력:

```text
id, taken_event_id, bsl_ssl_reference_id, liquidity_reference_id,
symbol, timeframe, swept_side, expected_reversal, trigger_price,
traversal_at, sweep_extreme, lifecycle_status, classification,
reclaimed_at, bars_to_reclaim, traversal_ticks,
reason_codes, parameters_version
```

## 고급 구현 버전

### HTF

- HTF BSL/SSL sweep과 LTF sweep을 독립 저장한다.
- LTF candidate에 겹치는 HTF reference id를 연결한다.
- timeframe, source kind, IRL/ERL, confluence count를 ranking feature로 출력한다.

### 세션

- completed London, New York, Asia, overnight, prior day high/low sweep을 분류한다.
- `America/New_York` timezone과 DST-aware calendar를 사용한다.
- killzone 안 traversal과 reclaim에 session metadata를 붙인다.
- developing session high/low와 completed source를 혼동하지 않는다.

### 유동성

- equal highs/lows cluster sweep을 지원한다.
- sweep extreme, traversal depth, reclaim 속도를 score feature로 출력한다.
- taken BSL 뒤 active SSL, taken SSL 뒤 active BSL 후보를 조회한다.
- run_or_breakout candidate를 continuation 분석에 전달한다.

### SMT

- correlated symbol별 comparable BSL/SSL traversal과 sweep을 timestamp 정렬한다.
- 한 심볼만 comparable liquidity를 sweep한 경우 SMT candidate metadata를 만든다.
- Sweep 모듈 자체는 SMT를 확정하지 않는다.

### 경제지표

- economic event 전후 configured window의 sweep에 calendar metadata를 붙인다.
- event metadata는 sweep 정의를 바꾸지 않는다.
- news gap traversal을 일반 wick sweep과 분리해 분석한다.

## False Signal 제거 방법

- active reference의 최초 strict traversal만 candidate로 사용한다.
- touch, taken, sweep, run_or_breakout_candidate를 분리한다.
- BSL과 SSL reclaim 방향을 뒤집지 않는다.
- cluster zone의 바깥 경계를 trigger로 사용한다.
- same-bar reclaim의 intrabar 순서 불확실성을 기록한다.
- reclaim window를 versioned parameter로 저장한다.
- expired 뒤 late reclaim을 과거 confirmed sweep으로 소급 변경하지 않는다.
- sweep만으로 reversal과 entry를 확정하지 않는다.
- displacement와 MSS를 독립 후행 조건으로 기록한다.
- HTF/LTF 중첩 event를 중복 수익으로 계산하지 않는다.

## Pine Script 구현 가이드

Pine Script에서는 taken event 발생 시 제한된 candidate 배열을 연다.

```text
on confirmed bar:
  if active BSL taken:
    open BSL candidate(trigger, bar_index, extreme = high)

  if active SSL taken:
    open SSL candidate(trigger, bar_index, extreme = low)

for candidate in open candidates:
  bars_elapsed = bar_index - traversal_bar_index

  if BSL:
    extreme = max(extreme, high)
    if close < trigger:
      confirm sweep

  if SSL:
    extreme = min(extreme, low)
    if close > trigger:
      confirm sweep

  if bars_elapsed > sweep_reclaim_bars and not reclaimed:
    expire as run_or_breakout_candidate
```

- `barstate.isconfirmed`에서 reclaim을 확정한다.
- active candidate, confirmed sweep, expired run candidate를 다른 색상으로 표시한다.
- alert는 `BSL sweep confirmed`, `SSL sweep confirmed`, `run candidate`를 분리한다.
- same-bar reclaim에는 `intrabar_order_unknown` tag를 표시한다.
- 오래된 시각화 객체만 정리하고 event count는 유지한다.

## Python Scanner 구현 가이드

event table을 분리한다.

```text
liquidity_sweep_candidates
liquidity_sweep_events
liquidity_run_or_breakout_candidates
liquidity_sweep_context_links
```

- upstream `taken_event_id`를 foreign key로 보존한다.
- `groupby(symbol, timeframe)` 안에서 candidate window를 평가한다.
- traversal bar를 window의 `0`번째 봉으로 포함한다.
- BSL은 window close의 `< trigger`, SSL은 `> trigger`를 검사한다.
- candidate 진행 중 extreme을 갱신한다.
- 최초 reclaim만 confirmed event로 저장한다.
- late reclaim은 interaction table에 별도 저장한다.
- sweep 이후 displacement, MSS event를 시간순으로 join한다.
- session, HTF, calendar, SMT metadata는 별도 context table로 유지한다.

## 백테스트 시 주의사항

- traversal bar close 이전에 same-bar sweep을 알고 있었다고 가정하지 않는다.
- OHLC만으로 same-bar traversal과 reclaim의 실제 순서를 확정하지 않는다.
- reclaim window가 바뀌면 결과도 달라진다. parameter version을 반드시 저장한다.
- touch, strict traversal, reclaim, reversal entry를 각각 분리한다.
- expired candidate를 실패한 reversal과 동일하게 해석하지 않는다.
- run_or_breakout_candidate는 최종 continuation 확정값이 아니다.
- 복수 level 동시 sweep을 독립 거래처럼 중복 집계하지 않는다.
- period boundary source는 해당 기간 종료 이후에만 사용한다.
- gap traversal과 일반 wick traversal을 분리해 통계를 낸다.
- sweep 이후 displacement와 MSS 발생 순서를 기록한다.

## 최종 Rulebook 정의

Liquidity sweep은 look-ahead 없이 확정된 active BSL 또는 SSL의 최초 strict traversal 뒤, configured reclaim window 안에 종가가 trigger 안쪽으로 복귀할 때 확정되는 반전형 가격 사건이다. reclaim window는 자막의 고정값이 아니라 versioned research parameter이다. reclaim이 없으면 sweep으로 확정하지 않고 `run_or_breakout_candidate`로 분리하며, 반전과 진입 여부는 후행 displacement, MSS, retracement 규칙에서 판정한다.
