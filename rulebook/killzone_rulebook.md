# Killzone Rulebook

공통 계약: [README.md](README.md)

## 객관적 판별 규칙

### 필수 조건
- timestamp가 `America/New_York` local time으로 변환됐다.
- 후보 시각이 config의 `[start, end)` 구간에 속하는지 계산한다.
- `window_name`, `start`, `end`, timezone을 출력한다.

### 무효 조건
- timezone 없는 timestamp이다.
- 시작과 종료가 정의되지 않았다.

### 필터 조건
- 시간 창은 코드 상수가 아니라 versioned config로 둔다.
- calendar event overlap을 별도 metadata로 기록한다.

### 체크리스트
- [ ] DST가 적용됐는가?
- [ ] window config version이 있는가?
- [ ] 창 밖 후보 처리 정책이 명시됐는가?

## 코드 구현 규칙

### 입력 데이터
timezone-aware timestamp, killzone config

### 필요 조건
시장별 config가 필요하다.

### 의사코드
```text
local_time = convert(timestamp, America/New_York).time
for window in configured_windows:
  if window.start <= local_time < window.end: attach window.name
```

### False Signal 필터
UTC 고정 offset, 날짜 경계 오류, inclusive end 중복을 방지한다.

### 주관적 요소
전략별 관찰 창 선택.

### 객관적 요소
시간 포함 여부와 창 이름.

### 최소 구현 버전
config 기반 배경 표시와 후보 tag.

### 고급 구현 버전
시장별 session calendar, 휴장일, calendar event join.

