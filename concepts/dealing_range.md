# Dealing Range

대응 Rulebook: [dealing_range_rulebook.md](../rulebook/dealing_range_rulebook.md)

## 개념 정의

Dealing Range는 분석에 사용할 기준 저점과 기준 고점 사이의 범위이다. 범위를 정해야 equilibrium, premium, discount, OTE를 계산할 수 있다.

## ICT가 중요하게 보는 이유

같은 가격도 어느 range를 기준으로 보느냐에 따라 premium 또는 discount가 달라진다. 자막에서는 dealing range가 후반 Episode들에서 반복적으로 등장한다.

## 차트에서 발견 방법

최소 구현은 최근 확정 swing low와 swing high 쌍을 사용한다. 고급 구현은 HTF impulse leg, liquidity event, MSS 이후 leg를 별도 range 후보로 보존한다.

## 다른 개념과의 연결

Swing Points -> Dealing Range -> Premium/Discount/Equilibrium -> OTE. Daily Bias와 timeframe이 활성 range 선택에 영향을 준다.

## 실제 차트 예시

15분 OHLC에서 확정 swing low 이후 확정 swing high가 형성된 leg를 range로 저장하고 midpoint를 그린다. 동일 구간을 1시간 range와 비교한다.

## 실전 활용

scanner는 range 하나를 임의로 확정하지 않고 `range_id`, `timeframe`, `low`, `high`, `selection_reason`을 기록한다.

## 흔한 오해

- 화면에 보이는 전체 고저가 항상 활성 dealing range는 아니다.
- range 선택은 완전 객관화가 끝난 문제가 아니다. 최소 구현과 고급 구현을 분리한다.

