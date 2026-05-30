# Power of Three

대응 Rulebook: [power_of_three_rulebook.md](../rulebook/power_of_three_rulebook.md)

## 개념 정의

Power of Three는 accumulation, manipulation, distribution의 세 단계 모델이다. Episode 9와 Episode 10은 이 세 요소를 직접 연결한다.

## ICT가 중요하게 보는 이유

당일 또는 세션의 price delivery를 opening price 기준으로 구조화한다. 진입 패턴 하나가 아니라 시간에 따라 진행되는 상태 모델이다.

## 차트에서 발견 방법

최소 구현은 opening price 주변 제한 폭의 consolidation을 accumulation 후보, 반대 방향 liquidity sweep을 manipulation 후보, displacement와 range expansion을 distribution 후보로 둔다.

## 다른 개념과의 연결

Opening Price -> accumulation -> Judas Swing/manipulation -> displacement/distribution. Daily Bias가 예상 distribution 방향을 제한한다.

## 실제 차트 예시

5분 OHLC에서 bearish context를 선택한다. midnight open 주변 consolidation, 상단 sweep, 이후 하락 displacement와 세션 low 확장을 순서대로 표시한다.

## 실전 활용

scanner는 단계별 timestamp와 미완료 상태를 저장한다. 완성된 패턴만 남기면 실시간 alert를 구현할 수 없다.

## 흔한 오해

- 자막 검색 시 정확한 `"power of three"`만 찾으면 일부 표기가 누락된다. Episode 9에는 `"Power Three"` 표기도 있다.
- 모든 consolidation이 accumulation은 아니다.

