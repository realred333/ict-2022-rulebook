# Premium / Discount / Equilibrium

대응 Rulebook: [premium_discount_equilibrium_rulebook.md](../rulebook/premium_discount_equilibrium_rulebook.md)

## 개념 정의

Equilibrium은 dealing range의 50% 지점이다. midpoint보다 위는 premium, 아래는 discount이다.

## ICT가 중요하게 보는 이유

자막에서 premium, discount, equilibrium은 반복적으로 가격 위치를 설명한다. 방향성 후보와 되돌림 후보의 위치를 숫자로 표현할 수 있다.

## 차트에서 발견 방법

`equilibrium = (range_high + range_low) / 2`로 계산한다. 현재 가격의 정규 위치는 `(price - range_low) / (range_high - range_low)`이다.

## 다른 개념과의 연결

Dealing Range -> Premium/Discount/Equilibrium -> OTE. bearish 후보는 premium, bullish 후보는 discount 조건과 결합할 수 있다.

## 실제 차트 예시

15분 dealing range에 midpoint를 표시한다. bearish FVG retracement가 midpoint 위에서 발생하는지, bullish FVG retracement가 midpoint 아래에서 발생하는지 분류한다.

## 실전 활용

scanner는 이 개념을 단독 진입 신호가 아닌 위치 metadata와 필터로 사용한다.

## 흔한 오해

- discount라고 자동 매수하고 premium이라고 자동 매도하지 않는다.
- dealing range가 바뀌면 구분도 다시 계산해야 한다.

