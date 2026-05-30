# Daily Bias Rulebook

공통 계약: [README.md](README.md)

## 객관적 판별 규칙

### 필수 조건
- 출력 상태는 `bullish | bearish | neutral | manual_review` 중 하나이다.
- HTF active BSL/SSL, 최근 HTF sweep, HTF dealing range 위치를 수집한다.
- 근거와 충돌 근거를 reason code로 저장한다.

### 무효 조건
- 매일 강제로 bullish 또는 bearish를 선택한다.
- HTF 데이터가 부족한데 확정 방향을 반환한다.

### 필터 조건
- bullish 후보: SSL sweep 이후 상단 active BSL이 있고 HTF bullish delivery 조건이 있다.
- bearish 후보: BSL sweep 이후 하단 active SSL이 있고 HTF bearish delivery 조건이 있다.
- 양쪽 근거가 충돌하면 neutral 또는 manual_review이다.

### 체크리스트
- [ ] neutral 상태를 허용하는가?
- [ ] HTF timeframe과 근거 id가 저장됐는가?
- [ ] intraday entry와 bias를 분리했는가?

## 코드 구현 규칙

### 입력 데이터
HTF OHLC, liquidity registry, sweep, dealing range, optional order flow state

### 필요 조건
최소 일봉과 1시간 또는 설정 HTF 데이터가 필요하다.

### 의사코드
```text
collect bullish_reasons and bearish_reasons
if bullish only: bullish
else if bearish only: bearish
else if no decisive evidence: neutral
else: manual_review
```

### False Signal 필터
단일 intraday candle과 사후 결과로 bias를 역산하지 않는다.

### 주관적 요소
draw on liquidity 우선순위와 충돌 해소.

### 객관적 요소
활성 HTF 레벨, sweep, range 위치, reason code.

### 최소 구현 버전
HTF liquidity와 sweep 기반 4상태 분류.

### 고급 구현 버전
institutional order flow proxy, calendar, session별 confidence metadata.

