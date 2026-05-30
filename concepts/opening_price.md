# Opening Price

대응 Rulebook: [opening_price_rulebook.md](../rulebook/opening_price_rulebook.md)

## 개념 정의

Opening Price는 시간 문맥의 기준 가격이다. 이 저장소의 최소 구현은 New York local time midnight open을 우선 지원하고, 일봉 open과 08:30 open은 별도 레벨 유형으로 보존한다.

## ICT가 중요하게 보는 이유

opening price는 자막에서 매우 반복적으로 등장한다. Episode 39는 New York midnight opening price 위에서 bearish day short 후보를 설명한다.

## 차트에서 발견 방법

OHLC timestamp를 `America/New_York`으로 변환한다. 거래일별 midnight를 포함하는 첫 봉의 open을 저장하고 세션 동안 수평선으로 유지한다.

## 다른 개념과의 연결

Opening Price -> Judas Swing -> Power of Three. Daily Bias와 premium/discount 위치를 함께 비교한다.

## 실제 차트 예시

1분 OHLC에서 New York midnight open을 표시한다. bearish context에서 08:30 전후 가격이 open 위를 거래한 뒤 하락하는 사례를 수집한다.

## 실전 활용

scanner는 `level_type = ny_midnight | daily_open | event_0830`을 분리하고 timezone을 출력한다.

## 흔한 오해

- 서로 다른 opening price를 하나의 레벨처럼 섞지 않는다.
- 고정 UTC offset을 사용하면 DST 기간에 잘못된 봉을 선택한다.

