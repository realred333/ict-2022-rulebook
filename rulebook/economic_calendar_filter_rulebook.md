# Economic Calendar Filter Rulebook

공통 계약: [README.md](README.md)

## 객관적 판별 규칙

### 필수 조건
- calendar row에 `timestamp`, `timezone`, `currency`, `impact`, `title`, `source`가 있다.
- event timestamp를 `America/New_York`으로 정규화한다.
- 후보와 이벤트 사이 분 차이를 계산한다.

### 무효 조건
- source 또는 timezone이 없다.
- calendar fetch 실패를 이벤트 없음으로 처리한다.

### 필터 조건
- impact는 최소 `high | medium | low | unknown`으로 정규화한다.
- 기본 event window는 config의 `minutes_before`, `minutes_after`이다.
- Python Scanner가 외부 데이터를 join하고 Pine에는 결과 feed 또는 수동 표시를 사용한다.

### 체크리스트
- [ ] source와 fetch 시각을 저장했는가?
- [ ] timezone을 정규화했는가?
- [ ] fetch 실패와 이벤트 없음이 구분되는가?

## 코드 구현 규칙

### 입력 데이터
calendar event feed, OHLC candidate timestamp, window config

### 필요 조건
외부 calendar adapter와 캐시가 필요하다.

### 의사코드
```text
events = normalize_calendar(feed, America/New_York)
for candidate in candidates:
  attach events where -before <= minutes(candidate - event) <= after
  add reason_code by impact
```

### False Signal 필터
중복 이벤트, timezone 이중 변환, fetch 실패 은폐를 방지한다.

### 주관적 요소
사용할 provider, 통화 매핑, event window.

### 객관적 요소
이벤트 시각, impact, title, 후보와의 시간 차이.

### 최소 구현 버전
CSV calendar import와 후보 timestamp join.

### 고급 구현 버전
provider adapter, 캐시, revision tracking, holiday calendar.

