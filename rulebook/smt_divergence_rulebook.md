# SMT Divergence Rulebook

공통 계약: [README.md](README.md)

## 객관적 판별 규칙

### 필수 조건
- 비교 심볼 쌍과 기준 timeframe이 config에 정의됐다.
- timestamp를 같은 timezone과 bar interval로 정렬한다.
- bullish SMT: A가 prior swing low 아래로 가고 B는 대응 prior swing low를 깨지 않는다.
- bearish SMT: A가 prior swing high 위로 가고 B는 대응 prior swing high를 깨지 않는다.

### 무효 조건
- 비교 구간의 bar가 누락됐다.
- 대응 swing 시각 차이가 허용 window를 넘는다.
- 임의 심볼 쌍을 config 없이 비교한다.

### 필터 조건
- 기본 대응 window는 timeframe 기준 설정 봉 수이다.
- 양 방향 divergence와 어느 심볼이 갱신했는지 저장한다.

### 체크리스트
- [ ] 심볼 쌍 config가 있는가?
- [ ] timestamp 정렬이 완료됐는가?
- [ ] prior swing id가 양쪽 모두 있는가?

## 코드 구현 규칙

### 입력 데이터
두 심볼 OHLC, swing points, symbol pair config, alignment tolerance

### 필요 조건
동일 timezone의 다중 심볼 데이터가 필요하다.

### 의사코드
```text
align bars for A and B
if A.low < A.prior_swing_low and B.low >= B.prior_swing_low: emit bullish_smt
if A.high > A.prior_swing_high and B.high <= B.prior_swing_high: emit bearish_smt
```

### False Signal 필터
누락 bar, session 불일치, 허용 window 밖 swing 비교를 제외한다.

### 주관적 요소
비교 심볼 쌍과 swing 대응 window.

### 객관적 요소
상대 higher low/lower low 또는 lower high/higher high.

### 최소 구현 버전
ES/NQ 같은 config 쌍의 동시간 swing 갱신 불일치.

### 고급 구현 버전
복수 지수, FX pair, lead-lag 허용 window, sweep/MSS 결합.

