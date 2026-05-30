# Liquidity Sweep

대응 Rulebook: [liquidity_sweep_rulebook.md](../rulebook/liquidity_sweep_rulebook.md)

## 개념 정의

### 일반적인 정의

Liquidity sweep은 기존 high 또는 low 바깥의 stop 주문을 회수한 뒤 가격이 다시 기존 range 안쪽으로 돌아오는 사건이다. ICT 자막에서는 `sweep`, `stop hunt`, `raid on liquidity`가 관련 문맥에서 사용된다.

### ICT 정의

Episode 23은 sweep과 run을 명확히 구분한다.

- sweep: high 위 또는 low 아래로 얕게 거래한 뒤 range 안쪽으로 되돌아오는 반전형 사건
- run 또는 expansion: old high, old low, relative equal highs/lows를 지나 계속 확장하는 지속형 사건

따라서 liquidity level 관통만으로 sweep이 확정되지는 않는다.

```text
active BSL 또는 SSL
-> strict traversal
-> taken event
-> range 안쪽 reclaim
-> confirmed sweep
```

- BSL sweep: BSL trigger 위로 거래한 뒤 trigger 아래로 종가가 돌아온 사건
- SSL sweep: SSL trigger 아래로 거래한 뒤 trigger 위로 종가가 돌아온 사건

Episode 3은 old high 위 또는 old low 아래의 stop hunt 이후 MSS를 찾는 흐름을 설명한다. Episode 5는 stop hunt와 raid on liquidity를 함께 언급한다.

### 차트에서 의미

Sweep은 liquidity가 회수되었고 해당 방향의 확장이 즉시 유지되지 않았음을 뜻한다.

```text
BSL sweep
-> bearish reversal candidate
-> bearish displacement와 MSS 확인

SSL sweep
-> bullish reversal candidate
-> bullish displacement와 MSS 확인
```

`candidate`라는 표현이 중요하다. sweep 자체는 진입 신호도, 반전 확정도 아니다. Episode 23의 설명처럼 reclaim 없이 계속 확장하면 sweep이 아니라 run 또는 breakout 후보이다.

### 시장 심리

ICT의 설명 모델은 다음과 같다.

- high 위 buy stops 또는 low 아래 sell stops가 체결된다.
- stop 체결 이후 가격이 range 안쪽으로 돌아오면 관통 방향의 지속이 실패한 것으로 본다.
- 이후 반대 방향 displacement와 MSS가 나타나는지 확인한다.

OHLC 코드는 실제 stop 수량, 주문 주체, 체결 의도를 관측할 수 없다. sweep은 stop hunt를 직접 증명하는 데이터가 아니라 `traversal + reclaim` 가격 프록시이다.

## ICT 전체 체계에서의 위치

### 선행 개념

- 필수: [Swing Points](swing_points.md)
- 필수: [Liquidity](liquidity.md)
- 필수: [BSL / SSL](bsl_ssl.md)
- 선택: 완료된 세션 high/low와 HTF liquidity

### 후행 개념

- [Displacement](displacement.md)
- [Market Structure Shift](market_structure_shift.md)
- [Fair Value Gap](fair_value_gap.md)
- [Judas Swing](judas_swing.md)
- [SMT Divergence](smt_divergence.md)

### 왜 중요한가

BSL/SSL 모듈은 관통된 liquidity를 `taken`으로 기록한다. Sweep 모듈은 그 사건이 range 안쪽으로 복귀한 반전형 사건인지, 관통 방향으로 이어진 run인지 분리한다.

```text
Liquidity
-> BSL / SSL
  -> taken event
    -> sweep candidate
      -> confirmed sweep | run_or_breakout
```

이 분류가 없으면 단순 breakout과 liquidity raid를 같은 신호로 처리하게 된다.

## 실제 차트에서 발견하는 방법

### 찾는 순서

1. `active` 상태였던 BSL/SSL reference의 최초 strict traversal event를 받는다.
2. BSL이면 `bar.high > trigger_price`, SSL이면 `bar.low < trigger_price`였는지 확인한다.
3. traversal 시점부터 reclaim window를 연다.
4. BSL은 `close < trigger_price`, SSL은 `close > trigger_price`인지 닫힌 봉마다 검사한다.
5. window 안에 reclaim하면 sweep을 확정한다.
6. reclaim 없이 window가 끝나면 `expired`와 `run_or_breakout_candidate` reason을 기록한다.
7. sweep 폭, reclaim까지 걸린 봉 수, ATR 비율을 저장한다.
8. confirmed sweep을 displacement와 MSS 모듈에 전달한다.

### 유효 조건

- 대상 reference가 traversal 직전에 `active`였다.
- 대상 reference는 look-ahead 없이 확정된 BSL 또는 SSL이다.
- traversal은 동일 가격 touch가 아니라 strict inequality이다.
- reclaim은 닫힌 봉의 종가로 판정한다.
- traversal bar 자체가 range 안쪽으로 마감하면 same-bar reclaim으로 허용한다.
- BSL cluster trigger는 zone 상단 `price_high`, SSL cluster trigger는 zone 하단 `price_low`이다.
- 허용 reclaim 봉 수와 계산 기준을 parameters version에 기록한다.

### 무효 조건

- 이미 `taken`인 reference의 반복 관통을 신규 sweep으로 생성한다.
- wick touch만으로 taken 또는 sweep을 확정한다.
- reclaim 없이 관통 방향으로 계속 확장했는데 sweep으로 표시한다.
- 미래 봉의 reclaim을 미리 읽어 traversal 시점에 sweep을 확정한다.
- BSL과 SSL trigger 경계를 뒤집는다.
- sweep만으로 reversal, entry, 손절을 확정한다.

### 대표 예시

```text
active BSL trigger: 105.00
sweep_reclaim_bars: 3  # research default

bar t:
  high  = 105.50
  close = 105.25
  -> BSL taken
  -> sweep candidate

bar t + 1:
  close = 104.75
  -> close < 105.00
  -> BSL sweep confirmed
  -> bars_to_reclaim = 1
  -> bearish displacement 확인 대기
```

반대로 `t + 3`까지 종가가 `105.00` 아래로 돌아오지 않으면 최소 구현에서는 confirmed sweep이 아니다.

### Sweep과 Run 구분

| 사건 | 관통 | range 안쪽 reclaim | 해석 |
| --- | --- | --- | --- |
| touch | 없음 | 해당 없음 | 관찰만 기록 |
| taken | strict traversal | 아직 미정 | sweep/run 분류 대기 |
| sweep | strict traversal | 허용 window 안에 있음 | 반전 후보 |
| run 또는 breakout 후보 | strict traversal | 허용 window 안에 없음 | 지속 가능성 후보 |

Episode 23은 sweep을 shallow move 뒤 reverse back into the range로, run을 level을 지나 continue하는 움직임으로 설명한다.

### 보조 설명: 세션 high/low

Episode 3은 London, New York, Asia session의 high와 low를 관찰하고 해당 레벨 위아래 sweep 가능성을 연구하도록 설명한다. Episode 18은 previous day low sweep 뒤 rally 예시를 제공한다.

세션 source를 쓰려면 timezone과 세션 종료 시각이 필요하다. 최소 구현은 swing 기반 BSL/SSL만 사용하고, 세션 source는 고급 구현으로 분리한다.

## 실전 활용

### 진입

Sweep 단독 진입은 금지한다.

```text
BSL sweep
-> bearish displacement
-> bearish MSS
-> bearish FVG 또는 confirmed OB retracement
-> short 후보

SSL sweep
-> bullish displacement
-> bullish MSS
-> bullish FVG 또는 confirmed OB retracement
-> long 후보
```

### 손절

Sweep level은 손절 후보를 제공하지만 손절을 자동 확정하지 않는다.

- short 후보: BSL sweep extreme 위에 configured buffer를 더한 가격 검토
- long 후보: SSL sweep extreme 아래에 configured buffer를 뺀 가격 검토
- buffer는 `max(1 * tick_size, configured_buffer_ticks * tick_size)`처럼 수치화한다.

### 목표

- BSL sweep 뒤 short 후보: 현재 가격 아래 active SSL 목록
- SSL sweep 뒤 long 후보: 현재 가격 위 active BSL 목록
- 복수 목표가 있으면 distance, timeframe, source kind, IRL/ERL tag를 출력한다.
- 반대편 liquidity까지 반드시 도달한다고 가정하지 않는다.

### 시간대

기본 sweep 탐지는 시간대와 무관하다. 세션과 경제지표는 metadata 및 필터이다.

- Episode 3: session high/low sweep 연구
- Episode 5: 08:30 전후 stop hunt 예시
- Episode 38: lunch hour 이후 sell stops sweep 예시
- Episode 40: news driver와 killzone 안 stop run 예시

## 흔한 오해

### 초보자가 잘못 해석하는 부분

- liquidity level을 넘은 모든 봉을 sweep이라고 부른다.
- wick touch와 strict traversal을 구분하지 않는다.
- 관통 방향으로 계속 확장하는 run을 sweep으로 분류한다.
- same-bar wick sweep만 허용하고 다음 봉 reclaim은 배제한다.
- reclaim window를 자막의 고정 진리처럼 취급한다.
- sweep만 보고 바로 진입한다.
- 이미 taken인 level의 반복 관통을 매번 새로운 sweep으로 센다.

### ICT가 실제로 의미하는 것

- sweep은 level 바깥으로 나갔다가 range 안쪽으로 돌아오는 반전형 사건이다.
- run은 level을 지나 계속 확장하는 지속형 사건이다.
- 자막은 모든 시장과 timeframe에 적용할 고정 reclaim 봉 수를 제시하지 않는다.
- 코드는 reclaim window를 `research default`로 노출하고 백테스트해야 한다.
- sweep 이후 displacement와 MSS를 독립적으로 확인해야 한다.

## 다른 개념과의 연결

### 관련 개념

| 연결 개념 | sweep의 역할 |
| --- | --- |
| Liquidity | stop pool reference 생성 |
| BSL / SSL | sweep 대상과 trigger 제공 |
| Displacement | sweep 이후 반대 방향 repricing 확인 |
| MSS | sweep 이후 구조 변화 확인 |
| FVG | displacement leg 안 되돌림 후보 |
| Judas Swing | opening price와 시간 조건이 추가된 sweep 응용 |
| SMT Divergence | 상관 심볼의 sweep 불일치 비교 |
| Economic Calendar Filter | event window 안 sweep metadata |

### 의존 관계

```text
active BSL
-> taken BSL
  -> reclaim below trigger
    -> BSL sweep
      -> bearish displacement와 MSS 후보

active SSL
-> taken SSL
  -> reclaim above trigger
    -> SSL sweep
      -> bullish displacement와 MSS 후보
```

## 코드 구현 시 문제점

### 사람이 보는 기준

사람은 level을 얼마나 얕게 넘었는지, 얼마나 빨리 range 안으로 돌아왔는지, 이후 가격이 energetic하게 반대 방향으로 움직이는지 함께 본다. 같은 traversal도 HTF level, session, news event에 따라 중요도를 다르게 평가한다.

### 코드로 구현 가능한 부분

- active BSL/SSL 최초 strict traversal 수신
- same-bar 및 multi-bar reclaim 판정
- sweep 폭을 tick과 ATR 비율로 계산
- reclaim까지 걸린 봉 수 계산
- sweep과 run_or_breakout_candidate 분리
- sweep extreme과 reference id 저장
- 후행 displacement/MSS 연결

### 코드로 구현 어려운 부분

- 모든 symbol과 timeframe에 공통인 최적 reclaim window
- `shallow`의 보편적 수치 임계값
- 복수 sweep 중 narrative상 핵심 사건 선택
- OHLC 한 봉 안에서 traversal과 reclaim의 실제 발생 순서
- 실제 stop 체결 수량과 주문 주체 확인

어려운 부분은 `research default`, `manual_review_reason`, strategy-specific ranking으로 분리한다.

## 핵심 문장

1. Sweep은 단순 touch가 아니라 active liquidity의 strict traversal 이후 reclaim이다.
2. BSL sweep은 BSL trigger 위 관통 뒤 trigger 아래 종가 복귀이다.
3. SSL sweep은 SSL trigger 아래 관통 뒤 trigger 위 종가 복귀이다.
4. Sweep과 run 또는 breakout은 같은 사건이 아니다.
5. 관통 방향으로 계속 확장하면 confirmed sweep으로 처리하지 않는다.
6. Reclaim window는 자막의 고정값이 아니라 백테스트할 research default이다.
7. Same-bar reclaim은 허용하되 봉 내부 순서 불확실성을 기록한다.
8. Sweep은 반전 후보이지 진입 신호가 아니다.
9. Sweep 이후 반대 방향 displacement와 MSS를 독립적으로 확인한다.
10. 이미 taken인 level의 반복 관통은 신규 sweep으로 중복 집계하지 않는다.

## 이 개념의 본질

Liquidity sweep은 active BSL 또는 SSL을 strict traversal한 뒤 range 안쪽으로 복귀했는지를 추적하여 stop 회수형 반전 후보와 지속형 run을 분리하는 상태 머신이다.

## Transcript 근거 분류

### 에피소드 전반에 반복되는 핵심 근거

- Episode 3: session high/low 위아래 sweep, old high/low stop hunt 이후 MSS 탐색
- Episode 5: stop hunt와 raid on liquidity, low 관통 뒤 후속 구조 관찰
- Episode 12: short-term lows 아래 SSL sweep 가능성
- Episode 18: previous day low sweep 뒤 rally 예시
- Episode 23: sweep과 run/expansion의 용어 차이, shallow traversal 뒤 range 안쪽 reversal
- Episode 38: lunch hour low 아래 sell stops sweep 예시
- Episode 40: short-term high stop run 뒤 displacement 예시

### 특정 구간에 집중된 보조 근거

- Episode 3: London, New York, Asia session high/low 관찰 시간
- Episode 5: 08:30 전후 stop hunt 사례
- Episode 18: stop hunt가 잦은 환경에서 단기 거래 주의

### 검색 분포

정리본 `transcripts/*.en.txt` 전체 검색 기준으로 `sweep|swept`는 13개 에피소드 파일, `stop hunt|stop hunts`는 4개, `run on liquidity`는 3개에 등장한다. 표현 횟수는 자막 중복을 포함하므로 정의의 근거가 아니라 분포 확인용이다.
