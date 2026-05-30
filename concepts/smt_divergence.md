# SMT Divergence

대응 Rulebook: [smt_divergence_rulebook.md](../rulebook/smt_divergence_rulebook.md)

## 개념 정의

SMT Divergence는 상관관계가 있는 두 심볼이 비교 기준 swing을 동일하게 갱신하지 않는 사건이다. Episode 38은 한쪽 lower low와 다른 쪽 higher low를 비교한다.

## ICT가 중요하게 보는 이유

단일 심볼의 sweep을 상관 자산과 비교해 상대적 약세 또는 강세 metadata를 추가한다. 자막 후반의 index 사례에서 반복된다.

## 차트에서 발견 방법

bullish SMT 후보는 심볼 A가 이전 swing low 아래 lower low를 만들지만 심볼 B는 대응 swing low를 깨지 않는 경우이다. bearish SMT는 high 방향을 반대로 비교한다.

## 다른 개념과의 연결

Correlated symbols + Swing Points -> SMT Divergence -> Sweep/MSS 후보 confidence metadata.

## 실제 차트 예시

같은 timezone의 ES와 NQ 1분 OHLC를 정렬한다. NQ가 lower low를 만들 때 ES가 prior swing low 위에 남는 사례를 표시한다.

## 실전 활용

Python Scanner가 주 구현 대상이다. Pine Script는 제한된 외부 심볼 요청으로 overlay를 제공할 수 있다.

## 흔한 오해

- 임의의 두 종목을 비교하지 않는다.
- timestamp 정렬 없이 swing을 비교하지 않는다.

