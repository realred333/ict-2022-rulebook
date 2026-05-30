# Institutional Order Flow

대응 Rulebook: [institutional_order_flow_rulebook.md](../rulebook/institutional_order_flow_rulebook.md)

## 개념 정의

Institutional Order Flow는 상위 방향과 일치하는 price delivery pattern을 관찰하는 프레임이다. 이 저장소에서는 자동화 가능한 proxy로 HTF FVG, MSS, OB, liquidity target의 정렬을 사용한다.

## ICT가 중요하게 보는 이유

Episode 6은 institutional order flow를 주제로 다루고 FVG의 3캔들 formation과 연결한다. Episode 8은 forex 적용을 다룬다.

## 차트에서 발견 방법

HTF에서 최근 confirmed MSS 방향, 활성 FVG/OB zone, draw on liquidity 방향을 계산한다. 모두 같은 방향이면 aligned, 충돌하면 mixed로 기록한다.

## 다른 개념과의 연결

HTF MSS/FVG/OB + Draw on Liquidity -> Institutional Order Flow -> Daily Bias filter.

## 실제 차트 예시

1시간 OHLC에서 bullish MSS, bullish FVG retracement, 상단 BSL 목표가 함께 있는 구간을 표시하고 5분 bullish 후보만 필터링한다.

## 실전 활용

scanner는 이를 객관적 사실처럼 단일 boolean으로 만들지 않는다. `aligned_bullish`, `aligned_bearish`, `mixed`, `manual_review`를 출력한다.

## 흔한 오해

- 기관 주문 자체를 OHLC만으로 관측한다고 주장하지 않는다.
- proxy 정렬과 실제 시장 참여자 주문은 동일하지 않다.

