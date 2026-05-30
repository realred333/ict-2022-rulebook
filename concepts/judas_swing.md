# Judas Swing

대응 Rulebook: [judas_swing_rulebook.md](../rulebook/judas_swing_rulebook.md)

## 개념 정의

Judas Swing은 세션 초기에 opening price의 한쪽으로 움직여 유동성을 회수한 뒤 반대 방향으로 전환하는 후보이다.

## ICT가 중요하게 보는 이유

자막에서는 false rally, fake rally, opening price 주변 움직임과 연결된다. Power of Three의 manipulation leg를 intraday에서 관찰하는 방법이다.

## 차트에서 발견 방법

활성 killzone 안에서 opening price 한쪽의 BSL 또는 SSL을 관통하고, 제한 시간 안에 반대 방향 displacement와 MSS가 발생하는지 확인한다.

## 다른 개념과의 연결

Opening Price + Killzone + Sweep -> Judas Swing -> MSS/FVG retracement. Power of Three의 manipulation 단계와 연결된다.

## 실제 차트 예시

1분 OHLC에서 New York AM 창에 midnight open 위 BSL을 회수한 뒤 bearish MSS를 만드는 사례를 표시한다.

## 실전 활용

scanner는 `judas_candidate`와 `judas_confirmed`를 분리한다. sweep만 있고 MSS가 없으면 confirmed로 올리지 않는다.

## 흔한 오해

- opening price를 잠시 넘는 모든 움직임이 Judas Swing은 아니다.
- 시간 창 없는 sweep을 Judas Swing으로 분류하지 않는다.

