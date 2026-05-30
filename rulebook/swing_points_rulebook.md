# Swing Points Rulebook

공통 계약: [README.md](README.md)

## 목적

닫힌 OHLC만 사용해 ICT short-term swing high와 swing low를 탐지한다. 탐지 결과는 liquidity, BSL/SSL, sweep, MSS, dealing range, SMT 모듈이 공유하는 immutable reference로 저장한다.

## 객관적 정의

연속된 닫힌 봉 `i-1`, `i`, `i+1`에 대해 다음을 계산한다.

```text
short_term_swing_high(i) =
  high[i] > high[i - 1]
  and high[i] > high[i + 1]

short_term_swing_low(i) =
  low[i] < low[i - 1]
  and low[i] < low[i + 1]
```

신호는 오른쪽 봉 `i+1`이 닫힌 시점에만 확정한다.

고급 계층은 confirmed swing 목록에 같은 규칙을 재귀 적용한다.

```text
intermediate_term_high(k) =
  short_term_high[k] > short_term_high[k - 1]
  and short_term_high[k] > short_term_high[k + 1]

intermediate_term_low(k) =
  short_term_low[k] < short_term_low[k - 1]
  and short_term_low[k] < short_term_low[k + 1]

long_term_high(m) =
  intermediate_term_high[m] > intermediate_term_high[m - 1]
  and intermediate_term_high[m] > intermediate_term_high[m + 1]

long_term_low(m) =
  intermediate_term_low[m] < intermediate_term_low[m - 1]
  and intermediate_term_low[m] < intermediate_term_low[m + 1]
```

## 필수 조건

- `timestamp, symbol, timeframe, open, high, low, close`가 존재한다.
- timestamp는 symbol과 timeframe별로 오름차순 정렬된다.
- 기본 탐지에 사용하는 3개 봉이 모두 닫혔다.
- 3개 봉의 timestamp 간격이 해당 timeframe과 일치한다.
- 비교는 strict inequality를 사용한다.
- 확정 시각 `confirmed_at`은 오른쪽 봉의 close timestamp이다.

## 무효 조건

- 오른쪽 봉이 닫히기 전에 confirmed event를 생성한다.
- 비교 대상 OHLC 중 하나가 결측값이다.
- 세 봉 중 하나가 다른 symbol 또는 timeframe에 속한다.
- interval 누락이 있는데 인접 배열 index만으로 연속 봉으로 간주한다.
- 중심 봉과 인접 봉의 high 또는 low가 같지만 short-term swing으로 확정한다.
- 미래 봉을 사용하면서 중심 봉 시각에 거래 가능 신호가 있었다고 백테스트한다.

## 예외 조건

- equal highs/lows: strict inequality가 실패하므로 short-term swing으로 확정하지 않는다. 별도 equal-level detector에 전달한다.
- 거래소 휴장과 session gap: instrument calendar 기준으로 정상 gap인지 데이터 누락인지 분리한다.
- 한 봉이 swing high와 swing low를 동시에 만족: 두 event를 모두 저장한다.
- confirmed swing의 사후 관통: swing event를 삭제하지 않는다. liquidity 모듈에서 연결 level 상태를 `taken`으로 바꾼다.
- 사람이 선택한 HTF reference: 자동 탐지 결과와 별도로 `manual_review_reason`을 저장한다.

## 상태 변화(State Machine)

### candidate

중심 후보 봉 `i`가 닫혔고 다음 조건 중 하나를 만족하지만 오른쪽 봉 `i+1`이 아직 닫히지 않은 상태이다.

```text
high[i] > high[i - 1]
or
low[i] < low[i - 1]
```

### confirmed

오른쪽 봉 `i+1`이 닫힌 뒤 3캔들 객관적 정의를 만족한다. `pivot_at = timestamp[i]`, `confirmed_at = timestamp[i+1]`를 모두 저장한다.

### invalidated

candidate 상태였으나 오른쪽 봉이 닫힌 뒤 strict inequality를 만족하지 못한다. confirmed swing이 나중에 가격 관통을 당한 경우에는 이 상태를 사용하지 않는다.

### expired

candidate 생성 후 다음 확인 봉을 정상적으로 수신하지 못한 채 데이터 segment가 종료되거나, configured confirmation timeout을 넘긴다. 연속 OHLC의 최소 구현에서는 일반적으로 발생하지 않는다.

### manual_review

다음 중 하나이면 자동 결과와 함께 검토 사유를 남긴다.

- 데이터 누락인지 정상 session gap인지 자동 판별할 calendar가 없다.
- equal high/low cluster를 swing 우선순위에 포함할지 결정해야 한다.
- 복수 HTF swing 중 trading model의 기준 swing을 선택해야 한다.
- imbalance rebalance 문맥을 계층 우선순위에 적용해야 한다.

## 코드 구현 규칙

### 입력 데이터

```text
timestamp
symbol
timeframe
open
high
low
close
is_closed
optional: session_calendar
```

### 필요 변수

```text
left_bar
center_bar
right_bar
pivot_at
confirmed_at
direction = high | low
level
status
hierarchy = short_term | intermediate_term | long_term
reason_codes[]
parameters_version
```

### 조건식

```text
is_sth = center.high > left.high and center.high > right.high
is_stl = center.low < left.low and center.low < right.low
```

### 의사코드

```text
for each closed right_bar:
  left_bar = bars[-3]
  center_bar = bars[-2]
  right_bar = bars[-1]

  assert_same_symbol_timeframe(left_bar, center_bar, right_bar)
  assert_contiguous_or_calendar_gap(left_bar, center_bar, right_bar)

  if center_bar.high > left_bar.high and center_bar.high > right_bar.high:
    emit confirmed short_term swing_high(
      pivot_at=center_bar.timestamp,
      confirmed_at=right_bar.timestamp,
      level=center_bar.high
    )

  if center_bar.low < left_bar.low and center_bar.low < right_bar.low:
    emit confirmed short_term swing_low(
      pivot_at=center_bar.timestamp,
      confirmed_at=right_bar.timestamp,
      level=center_bar.low
    )

  send equal highs/lows to equal_level_detector
```

## 최소 구현 버전

OHLC 데이터만 사용한다.

- short-term swing high/low만 탐지한다.
- 좌우 1봉을 비교한다.
- 오른쪽 봉 close 시점에 확정한다.
- strict inequality를 사용한다.
- symbol과 timeframe별 레지스트리를 분리한다.
- 출력은 `pivot_at`, `confirmed_at`, `direction`, `level`, `status`를 포함한다.

## 고급 구현 버전

### HTF

- timeframe별 short-term swing을 독립 계산한다.
- short-term 목록에 재귀 규칙을 적용해 intermediate-term과 long-term swing을 생성한다.
- LTF swing과 HTF swing을 id로 연결한다.

### 세션

- swing 탐지 자체는 session으로 제한하지 않는다.
- killzone, 08:30, 13:30, lunch hour tag는 후행 filter metadata로 추가한다.

### 유동성

- confirmed swing high는 BSL source 후보로 전달한다.
- confirmed swing low는 SSL source 후보로 전달한다.
- 사후 관통은 liquidity module에서 `active -> taken`으로 처리한다.

### SMT

- correlated symbol의 aligned timestamp swing 목록을 비교한다.
- 어느 심볼이 prior swing을 갱신했는지 metadata로 저장한다.

### 경제지표

- swing 탐지 규칙은 바꾸지 않는다.
- event window 안에서 생성 또는 관통된 swing에 calendar metadata를 추가한다.

## False Signal 제거 방법

- 닫히지 않은 오른쪽 봉을 사용하지 않는다.
- `pivot_at`과 실제 사용 가능 시각인 `confirmed_at`을 분리한다.
- equal highs/lows를 strict swing으로 오분류하지 않는다.
- 캔들 종가 방향을 swing 조건에 추가하지 않는다.
- 데이터 gap을 정상 연속 봉처럼 처리하지 않는다.
- confirmed swing 사후 관통을 과거 event 삭제로 처리하지 않는다.
- session filter와 swing 정의를 섞지 않는다.

## Pine Script 구현 가이드

현재 닫힌 봉을 오른쪽 확인 봉으로 사용한다.

```text
is_sth = high[1] > high[2] and high[1] > high[0]
is_stl = low[1] < low[2] and low[1] < low[0]
```

- marker는 중심 봉에 표시하므로 `offset = -1`을 사용한다.
- alert는 marker가 표시된 중심 봉이 아니라 현재 오른쪽 확인 봉 close에서 발생시킨다.
- `barstate.isconfirmed`를 사용한다.
- timeframe별 배열에 event id, `pivot_at`, `confirmed_at`, level을 저장한다.
- intermediate-term과 long-term 계층은 배열에 저장된 confirmed swing끼리 비교한다.

## Python Scanner 구현 가이드

row `t`가 오른쪽 확인 봉일 때 중심 봉은 `t-1`이다.

```text
is_sth[t] = high[t - 1] > high[t - 2] and high[t - 1] > high[t]
is_stl[t] = low[t - 1] < low[t - 2] and low[t - 1] < low[t]
```

- `symbol, timeframe, timestamp` 순서로 정렬한다.
- `groupby(symbol, timeframe)` 안에서 계산한다.
- expected timeframe interval과 timestamp 차이를 비교한다.
- 결과 테이블에 `pivot_at`과 `confirmed_at`을 모두 저장한다.
- intermediate-term과 long-term event는 short-term event 테이블 위에서 다시 계산한다.

## 백테스트 시 주의사항

- 중심 봉 시각에 swing을 알고 있었다고 가정하면 look-ahead bias이다.
- 진입 판단은 최소한 `confirmed_at` 이후에만 허용한다.
- 확인 봉 종가 체결을 가정할지 다음 봉 시가 체결을 가정할지 명시한다.
- 데이터 vendor의 session gap과 누락 봉 처리 정책을 기록한다.
- equal high/low 처리 정책을 parameters version에 포함한다.
- 좌우 2봉 pivot과 ICT 최소 3캔들 swing을 같은 결과처럼 비교하지 않는다.
- HTF swing을 LTF에 전달할 때 HTF 확인 봉이 닫힌 이후에만 사용할 수 있다.

## 최종 Rulebook 정의

ICT short-term swing point는 같은 symbol과 timeframe의 연속된 닫힌 3개 OHLC 봉에서 중심 high가 양쪽 high보다 높거나 중심 low가 양쪽 low보다 낮을 때, 오른쪽 확인 봉 close 시점에 생성되는 immutable 가격 reference이다.

