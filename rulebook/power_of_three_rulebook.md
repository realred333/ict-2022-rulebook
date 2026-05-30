# Power of Three Rulebook

공통 계약: [README.md](README.md)

## 객관적 판별 규칙

### 필수 조건
- 상태는 `waiting -> accumulation_candidate -> manipulation_candidate -> distribution_confirmed | expired`이다.
- accumulation 후보: opening price 주변 설정 폭 안에서 설정 시간 이상 머문다.
- manipulation 후보: 예상 distribution 반대쪽 liquidity를 sweep한다.
- distribution 확인: 예상 방향 displacement와 range expansion이 발생한다.

### 무효 조건
- manipulation 이전에 예상 방향 목표가 먼저 충족된다.
- 세션 종료 전 distribution 확인이 없다.
- daily bias가 neutral인데 방향을 확정값처럼 사용한다.

### 필터 조건
- 기본 accumulation 폭과 시간은 시장별 config로 둔다.
- Judas Swing, MSS, FVG 발생 여부를 연결한다.

### 체크리스트
- [ ] 세 단계를 timestamp 순서대로 기록했는가?
- [ ] opening price id가 있는가?
- [ ] 미완료 상태도 저장하는가?

## 코드 구현 규칙

### 입력 데이터
OHLC, opening price, liquidity, sweep, displacement, session config, optional bias

### 필요 조건
timezone-aware intraday 데이터가 필요하다.

### 의사코드
```text
if price remains near open for configured duration: accumulation_candidate
if opposite-side liquidity is swept: manipulation_candidate
if directional displacement expands range: distribution_confirmed
```

### False Signal 필터
단순 횡보, sweep 없는 확장, 세션 밖 완성을 분리한다.

### 주관적 요소
accumulation 폭, 지속 시간, 예상 distribution 방향.

### 객관적 요소
open 거리, 체류 시간, sweep, displacement, range expansion.

### 최소 구현 버전
config 기반 3단계 상태 머신과 수동 검토 출력.

### 고급 구현 버전
daily bias, Judas Swing, FVG entry, calendar event 결합.

