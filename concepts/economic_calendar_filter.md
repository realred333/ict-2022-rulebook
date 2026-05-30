# Economic Calendar Filter

대응 Rulebook: [economic_calendar_filter_rulebook.md](../rulebook/economic_calendar_filter_rulebook.md)

## 개념 정의

Economic Calendar Filter는 구조화된 경제 이벤트 시각과 impact를 intraday 후보에 결합하는 외부 데이터 필터이다.

## ICT가 중요하게 보는 이유

Episode 10은 economic calendar events를 open과 함께 다루고, Episode 40은 killzone 안에서 high 또는 medium impact 이벤트를 확인하는 흐름을 설명한다.

## 차트에서 발견 방법

calendar adapter에서 timestamp, currency, impact, title을 수집한다. 후보 시각이 이벤트 전후 설정된 window 안에 있는지 join한다.

## 다른 개념과의 연결

Economic Calendar + Killzone -> volatility window metadata. Judas Swing, displacement, MSS/FVG 후보에 이벤트 문맥을 추가한다.

## 실제 차트 예시

1분 OHLC와 calendar event를 New York timezone으로 정렬한다. 08:30 이벤트 전후 sweep, displacement, MSS가 발생한 날과 발생하지 않은 날을 비교한다.

## 실전 활용

Pine Script 단독으로는 과거와 미래 calendar feed를 안정적으로 제공하기 어렵다. Python Scanner adapter를 우선 구현한다.

## 흔한 오해

- 이벤트가 있으면 방향이 자동 결정되는 것은 아니다.
- 외부 calendar source와 timezone을 기록하지 않으면 결과를 재현할 수 없다.

