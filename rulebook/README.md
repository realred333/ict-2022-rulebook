# ICT Rulebook

## 목적

이 폴더는 Pine Script와 Python Scanner가 공유할 판별 계약을 정의한다. Rulebook 기본값은 수익성을 보장하는 값이 아니다. 자막의 정성 표현을 백테스트 가능한 초기값으로 바꾼 `research default`이다.

## 공통 입력 데이터

필수 OHLC schema:

```text
timestamp, symbol, timeframe, open, high, low, close
```

선택 schema:

```text
volume, tick_size, session_timezone, calendar_events, correlated_symbol_ohlc
```

## 공통 기본 파라미터

```text
timezone = "America/New_York"
swing_left = 1
swing_right = 1
atr_length = 14
median_body_length = 20
equal_level_tolerance = max(1 * tick_size, 0.10 * ATR(14))
displacement_body_factor = 1.50
displacement_body_ratio = 0.60
sweep_reclaim_bars = 3
candidate_expiry_bars = configurable per timeframe
```

## 공통 계산

```text
body = abs(close - open)
range = high - low
body_ratio = body / range if range > 0 else 0
equilibrium = (range_high + range_low) / 2
normalized_position = (price - range_low) / (range_high - range_low)
```

## 공통 상태 모델

```text
candidate -> confirmed | invalidated | expired | manual_review
```

zone 개념은 confirmed 이후 `mitigated` 상태를 추가할 수 있다. 완전 자동화할 수 없는 조건은 삭제하지 않고 `manual_review` metadata에 남긴다.

## 구현 원칙

- Pine Script: 차트 overlay, alert, 단일 또는 제한된 외부 심볼 확인에 사용한다.
- Python Scanner: 다중 심볼, 다중 timeframe, calendar join, 후보 저장, 백테스트에 사용한다.
- look-ahead를 금지한다. 기본 ICT short-term swing은 `swing_right = 1`이므로 오른쪽 1개 봉이 닫힌 뒤에만 확정한다.
- DST 처리는 고정 UTC offset이 아니라 `America/New_York` timezone으로 처리한다.
- 모든 후보는 `symbol`, `timeframe`, `detected_at`, `price_low`, `price_high`, `direction`, `status`, `reason_codes`, `parameters_version`을 기록한다.

## Rulebook 목록

각 개념 문서는 동일한 이름의 `_rulebook.md`와 연결된다.
