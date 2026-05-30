# Opening Price Rulebook

공통 계약: [README.md](README.md)

## 객관적 판별 규칙

### 필수 조건
- OHLC timestamp를 `America/New_York`으로 변환한다.
- 최소 레벨은 New York local time `00:00`을 포함하는 첫 봉의 open이다.
- 레벨 유형을 `ny_midnight`, `daily_open`, `event_0830`으로 분리한다.

### 무효 조건
- timezone이 없거나 DST 변환이 적용되지 않았다.
- 요청한 레벨 시각을 포함하는 봉이 없다.

### 필터 조건
- instrument 거래 시간에 따라 midnight 봉 부재를 명시적으로 처리한다.
- fallback을 사용할 경우 `fallback_reason`을 저장한다.

### 체크리스트
- [ ] timezone이 New York 기준인가?
- [ ] 레벨 유형이 구분됐는가?
- [ ] 거래일별로 레벨이 갱신되는가?

## 코드 구현 규칙

### 입력 데이터
timestamp, open, instrument session calendar, timezone

### 필요 조건
timezone-aware timestamp가 필요하다.

### 의사코드
```text
local_ts = convert(timestamp, America/New_York)
for each trading_date:
  ny_midnight_open = first_bar_containing(00:00).open
```

### False Signal 필터
고정 UTC offset과 서로 다른 open 유형 혼용을 금지한다.

### 주관적 요소
시장별 fallback 정책.

### 객관적 요소
timezone 변환, 기준 시각, open 가격.

### 최소 구현 버전
New York midnight open 수평선.

### 고급 구현 버전
복수 open 레벨과 거래소 session calendar.

