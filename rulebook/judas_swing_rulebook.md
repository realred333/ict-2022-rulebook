# Judas Swing Rulebook

공통 계약: [README.md](README.md)

## 객관적 판별 규칙

### 필수 조건
- 활성 killzone 안에서 발생한다.
- opening price 한쪽의 active liquidity를 sweep한다.
- 반대 방향 displacement와 MSS가 sweep 이후 발생한다.

### 무효 조건
- killzone 밖에서 발생했다.
- sweep 뒤 같은 방향으로 확장하고 반대 방향 MSS가 없다.
- 기준 opening price가 정의되지 않았다.

### 필터 조건
- sweep 대상이 BSL인지 SSL인지, open에서 몇 tick 떨어졌는지 기록한다.
- 경제 이벤트 overlap을 metadata로 추가한다.

### 체크리스트
- [ ] opening price id가 있는가?
- [ ] sweep, displacement, MSS id가 연결됐는가?
- [ ] 시간 창 안에서 순서대로 발생했는가?

## 코드 구현 규칙

### 입력 데이터
OHLC, opening price, killzone, liquidity sweep, displacement, MSS

### 필요 조건
각 이벤트가 timestamp와 id를 가져야 한다.

### 의사코드
```text
inside killzone:
  if sweep one side of open: create judas_candidate
  if opposite displacement and MSS after sweep: confirm
  if window ends first: expire
```

### False Signal 필터
시간 창 밖 sweep, MSS 없는 fake candidate, 이벤트 순서 역전을 제외한다.

### 주관적 요소
허용 sweep 거리와 확인 대기 시간.

### 객관적 요소
opening price, 시간 창, sweep, displacement, MSS 순서.

### 최소 구현 버전
opening price + killzone + sweep + MSS 상태 머신.

### 고급 구현 버전
calendar event, HTF bias, FVG retracement entry 연결.

