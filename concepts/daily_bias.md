# Daily Bias

대응 Rulebook: [daily_bias_rulebook.md](../rulebook/daily_bias_rulebook.md)

## 개념 정의

Daily Bias는 당일 우선 탐색할 방향과 draw on liquidity 후보이다. `bullish`, `bearish`, `neutral`, `manual_review` 상태를 사용한다.

## ICT가 중요하게 보는 이유

Episode 7은 daily bias가 매일 유리하게 작동하는 bias가 아니라고 설명한다. Episode 40도 매일 predetermined bias를 강제하지 않는 흐름을 강조한다.

## 차트에서 발견 방법

최소 구현은 HTF active liquidity와 전일 range 위치를 사용한다. 상단 liquidity 목표만 남고 하단 liquidity가 sweep된 경우 bullish 후보, 반대는 bearish 후보, 충돌하면 neutral이다.

## 다른 개념과의 연결

HTF Liquidity + Dealing Range + Institutional Order Flow -> Daily Bias. intraday Sweep/MSS/FVG pipeline의 방향 필터로 사용한다.

## 실제 차트 예시

일봉과 1시간 OHLC에서 활성 BSL/SSL, premium/discount 위치를 표시한다. 양쪽 목표가 모두 가까운 날은 neutral로 분류하고 intraday scanner 결과와 비교한다.

## 실전 활용

scanner는 방향을 억지로 선택하지 않는다. 근거 코드와 충돌 근거를 함께 출력한다.

## 흔한 오해

- 매 거래일 반드시 bullish 또는 bearish를 선택하지 않는다.
- daily bias만으로 entry price가 결정되지 않는다.

