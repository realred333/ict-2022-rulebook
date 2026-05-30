# Killzone

대응 Rulebook: [killzone_rulebook.md](../rulebook/killzone_rulebook.md)

## 개념 정의

Killzone은 특정 세션의 관찰 시간 창이다. Episode 8은 New York killzone을 New York session trades가 형성되는 time of day로 설명한다.

## ICT가 중요하게 보는 이유

같은 가격 패턴이라도 시간대를 제한하면 intraday 후보 수를 줄일 수 있다. Episode 40은 killzone과 경제 캘린더를 함께 확인하는 흐름을 설명한다.

## 차트에서 발견 방법

timestamp를 New York local time으로 변환하고 설정된 시간 창에 속하는지 확인한다. 시간 창 자체는 시장과 전략별 config로 둔다.

## 다른 개념과의 연결

Killzone + Economic Calendar Filter -> intraday time filter. Opening Price, Judas Swing, MSS/FVG entry pipeline과 결합한다.

## 실제 차트 예시

1분 OHLC 차트 배경에 London, New York AM, New York PM config window를 표시하고 sweep/MSS/FVG 후보가 어느 창에서 발생했는지 분류한다.

## 실전 활용

scanner는 시간 창 밖 후보를 삭제하지 않고 `outside_killzone` reason code로 보존할 수 있다.

## 흔한 오해

- killzone 자체가 진입 신호는 아니다.
- timezone과 DST 처리를 생략하면 재현할 수 없다.

