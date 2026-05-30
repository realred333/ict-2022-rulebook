# BSL / SSL

대응 Rulebook: [bsl_ssl_rulebook.md](../rulebook/bsl_ssl_rulebook.md)

## 개념 정의

### 일반적인 정의

BSL은 `Buy-Side Liquidity`, SSL은 `Sell-Side Liquidity`의 약어이다. ICT 문맥에서는 실제 order book 잔량을 뜻하지 않는다. 고점 위 또는 저점 아래에 stop 주문이 집중될 가능성을 나타내는 방향성 가격 reference이다.

### ICT 정의

- BSL: old high, swing high, relative equal highs, 완료된 기간 high 위에 buy stops가 존재할 가능성이 있는 상단 liquidity
- SSL: old low, swing low, relative equal lows, 완료된 기간 low 아래에 sell stops가 존재할 가능성이 있는 하단 liquidity

Episode 2는 old high 위의 buy stops와 old low 아래의 sell stops를 설명한다. Episode 3은 old low 아래의 sell stops와 relative equal highs 위의 buy stops를 한 차트에서 함께 표시한다. Episode 17은 buy side와 buy stops, sell stops와 sell-side liquidity를 같은 문맥에서 직접 연결한다.

이 저장소에서는 실제 주문 존재를 확정하지 않는다.

```text
upper liquidity reference -> BSL candidate
lower liquidity reference -> SSL candidate
```

BSL과 SSL은 새로운 source를 만드는 탐지기가 아니다. [Liquidity](liquidity.md) 모듈이 만든 stop pool reference에 의미 label을 붙이는 분류 계층이다.

### 차트에서 의미

BSL과 SSL은 현재 가격을 기준으로 위아래 목표 후보를 구분한다.

```text
active BSL above current price
-> upside target candidate

active SSL below current price
-> downside target candidate
```

고점 또는 저점이 관통되면 label을 삭제하지 않는다. 해당 reference의 availability를 `taken`으로 바꾼다.

```text
active BSL + later high > BSL trigger -> BSL taken
active SSL + later low < SSL trigger  -> SSL taken
```

`taken`은 반전을 뜻하지 않는다. 관통 후 reclaim은 [Liquidity Sweep](liquidity_sweep.md), 후속 방향 변화는 [Displacement](displacement.md)와 [Market Structure Shift](market_structure_shift.md)에서 판정한다.

### 시장 심리

ICT의 설명 모델은 다음과 같다.

- short 포지션의 보호 buy stop은 고점 위에 놓일 수 있다.
- 가격이 고점 위로 거래되면 buy stop이 market buy order로 전환될 수 있다.
- long 포지션의 보호 sell stop은 저점 아래에 놓일 수 있다.
- 가격이 저점 아래로 거래되면 sell stop이 market sell order로 전환될 수 있다.

Episode 13은 고점 위 buy stop이 시장가 매수 주문으로 바뀌어 반대편 매도 체결에 유동성을 제공하는 과정을 설명한다. Episode 25는 relative equal highs 위 buy stops 회수와 이후 sell-side liquidity 공격을 연결한다.

OHLC 코드는 실제 stop 수량, 주문 주체, 체결 의도를 볼 수 없다. 따라서 BSL과 SSL은 `assumed_stop_side`가 붙은 가격 프록시이다.

## ICT 전체 체계에서의 위치

### 선행 개념

- 필수: [Swing Points](swing_points.md)
- 필수: [Liquidity](liquidity.md)
- 선택: relative equal highs/lows cluster
- 선택: 완료된 이전 일자, 세션, overnight high/low

### 후행 개념

- [Liquidity Sweep](liquidity_sweep.md)
- [Displacement](displacement.md)
- [Market Structure Shift](market_structure_shift.md)
- [Daily Bias](daily_bias.md)
- [SMT Divergence](smt_divergence.md)

### 왜 중요한가

BSL/SSL은 추상적인 liquidity를 코드가 처리할 수 있는 방향 label로 바꾼다. 이후 모듈은 다음 질문을 명시적으로 구분할 수 있다.

- 어느 방향의 liquidity가 active인가?
- 가격이 BSL을 관통했는가, SSL을 관통했는가?
- 회수된 쪽과 반대 방향의 active target은 무엇인가?
- sweep 이후 확인해야 할 displacement와 MSS 방향은 무엇인가?

학습 맵의 위치는 다음과 같다.

```text
Swing Points
-> Liquidity
  -> BSL / SSL
    -> Liquidity Sweep
```

## 실제 차트에서 발견하는 방법

### 찾는 순서

1. liquidity registry에서 lifecycle이 `confirmed`이고 availability가 `active`인 stop pool reference를 불러온다.
2. `side = upper`이면 `liquidity_side = BSL`로 분류한다.
3. `side = lower`이면 `liquidity_side = SSL`로 분류한다.
4. BSL의 관통 기준은 zone 상단 `price_high`, SSL의 관통 기준은 zone 하단 `price_low`로 둔다.
5. 현재 가격보다 위에 있는 active BSL과 현재 가격보다 아래에 있는 active SSL을 directional target 목록에 둔다.
6. 현재 가격 반대편에 있거나 이미 `taken`인 reference는 이력으로 남기고 신규 target 목록에서는 제외한다.
7. target까지 거리를 tick과 ATR 비율로 계산한다.
8. 관통이 발생하면 `taken` event를 sweep classifier에 전달한다.

### 유효 조건

- upstream liquidity reference가 look-ahead 없이 확정되었다.
- upstream reference의 `side`가 `upper | lower` 중 하나이다.
- reference에 `price_low`, `price_high`, `available_at`, `availability`가 있다.
- `price_low <= price_high`이다.
- BSL은 upper source에서만 생성한다.
- SSL은 lower source에서만 생성한다.
- 신규 target 목록에는 `availability = active`인 reference만 넣는다.
- BSL target은 현재 가격 위, SSL target은 현재 가격 아래에 있을 때만 directional target으로 사용한다.

### 무효 조건

- swing 확인 전 source를 BSL 또는 SSL로 확정한다.
- high 기반 source를 SSL, low 기반 source를 BSL로 뒤집는다.
- 이미 `taken`인 reference를 새 목표처럼 사용한다.
- 현재 가격 아래에 남은 BSL 또는 현재 가격 위에 남은 SSL을 관통 처리 누락 없이 active directional target으로 사용한다.
- BSL label을 long 진입 신호, SSL label을 short 진입 신호로 사용한다.
- BSL/SSL만 보고 sweep, breakout, reversal을 확정한다.
- 실제 주문 데이터 없이 stop 수량을 출력한다.

### 대표 예시

```text
current close: 100.00

active upper reference:
  source_kind = equal_cluster
  price_low   = 104.75
  price_high  = 105.00
  -> BSL
  -> upside target candidate

active lower reference:
  source_kind = swing
  price_low   = 97.50
  price_high  = 97.50
  -> SSL
  -> downside target candidate

later bar:
  high = 105.25
  105.25 > 105.00
  -> BSL availability: active -> taken
  -> sweep 또는 breakout 분류 대기
```

### Source 종류

| upstream source | BSL 분류 | SSL 분류 |
| --- | --- | --- |
| confirmed swing | swing high | swing low |
| equal-level cluster | relative equal highs | relative equal lows |
| 완료된 이전 일자 | prior day high | prior day low |
| 완료된 세션 | session high | session low |
| 완료된 overnight 구간 | overnight high | overnight low |
| 수동 HTF reference | manual upper level | manual lower level |

최소 구현은 confirmed swing만 사용한다. 나머지는 liquidity registry의 고급 source가 준비된 뒤 추가한다.

### 보조 설명: 한쪽 회수 후 반대편 목표

Episode 17은 relative equal highs 위 buy side가 회수된 뒤 아래 sell stops 또는 SSL을 다음 후보로 설명한다. Episode 26은 relative equal lows 아래 SSL이 회수된 뒤 short-term high의 BSL을 다음 draw로 사용하는 tape reading 예시를 제공한다.

이 연결은 유용하지만 항상 발생하는 규칙은 아니다.

```text
taken BSL -> active SSL 후보 탐색
taken SSL -> active BSL 후보 탐색
```

scanner는 반대편 후보 목록을 출력한다. 단일 목표 선택이나 반드시 반대편까지 이동한다는 가정은 strategy-specific ranking으로 분리한다.

## 실전 활용

### 진입

BSL과 SSL 자체로 진입하지 않는다.

반전 모델 후보:

```text
BSL taken
-> bearish reclaim 여부 확인
-> bearish displacement
-> bearish MSS
-> short retracement model

SSL taken
-> bullish reclaim 여부 확인
-> bullish displacement
-> bullish MSS
-> long retracement model
```

추세 지속 모델에서는 active BSL 또는 SSL이 청산 목표가 될 수 있다. 같은 label이 목표와 회수 사건 양쪽에 쓰이므로 `target candidate`와 `taken event`를 분리한다.

### 손절

BSL/SSL label은 손절 가격을 직접 결정하지 않는다.

- BSL 회수 후 short 모델: 구조 기준 swing high와 configured buffer를 검토한다.
- SSL 회수 후 long 모델: 구조 기준 swing low와 configured buffer를 검토한다.
- 손절 규칙은 sweep, MSS, entry model rulebook에서 확정한다.

### 목표

- long 후보의 상단 목표: 현재 가격 위 active BSL
- short 후보의 하단 목표: 현재 가격 아래 active SSL
- 복수 목표가 있으면 `distance_ticks`, `distance_atr`, `timeframe`, `source_kind`, `confluence_count`를 출력한다.
- 첫 partial과 최종 목표는 전략별 청산 정책에서 선택한다.

Episode 40은 long 문맥에서 상단 BSL을 initial draw와 첫 partial로, short 문맥에서 하단 SSL을 목표로 사용하는 예시를 설명한다.

### 시간대

기본 분류는 시간대와 무관하다. 다만 source가 prior day, session, overnight high/low이면 `America/New_York` timezone과 기간 종료 시각을 명시한다. killzone과 경제지표는 BSL/SSL 정의가 아니라 후보 평가 metadata이다.

## 흔한 오해

### 초보자가 잘못 해석하는 부분

- BSL을 bullish signal, SSL을 bearish signal이라고 본다.
- BSL은 매수해야 하는 곳, SSL은 매도해야 하는 곳이라고 해석한다.
- 고점 위 또는 저점 아래를 한 번 거래하면 반드시 반전한다고 가정한다.
- wick touch와 strict traversal을 구분하지 않는다.
- taken level을 차트에서 지우고 과거 구조 이력을 잃는다.
- 현재 가격과 source 위치를 확인하지 않고 모든 BSL/SSL을 동일한 목표로 사용한다.
- 가장 가까운 BSL/SSL이 항상 최종 draw라고 가정한다.
- 자막의 설명 모델을 실제 order flow 데이터처럼 취급한다.

### ICT가 실제로 의미하는 것

- BSL과 SSL은 stop 기반 liquidity의 위치와 assumed stop side를 표현한다.
- BSL은 상승 목표가 될 수 있고, 회수 후 bearish 반전 후보의 입력이 될 수도 있다.
- SSL은 하락 목표가 될 수 있고, 회수 후 bullish 반전 후보의 입력이 될 수도 있다.
- label은 trade direction이 아니다.
- 관통은 `taken` 사건이다. 반전은 후행 구조로 확인한다.
- 복수 목표 중 우선순위는 HTF draw, 세션, 거리, source 종류를 포함한 별도 정책이다.

## 다른 개념과의 연결

### 관련 개념

| 연결 개념 | BSL/SSL의 역할 |
| --- | --- |
| Swing Points | high와 low source를 제공 |
| Liquidity | upper/lower stop pool registry를 제공 |
| Liquidity Sweep | BSL/SSL 관통 후 reclaim을 분류 |
| Displacement | taken event 이후 반대 방향 가격 전달 강도를 확인 |
| MSS | taken event 이후 구조 변화 여부를 확인 |
| Daily Bias | HTF directional target 후보를 전달 |
| SMT Divergence | 상관 심볼의 BSL/SSL 회수 불일치를 비교 |
| Judas Swing | 시간과 opening price 조건이 추가된 liquidity raid 응용 |

### 의존 관계

```text
confirmed upper liquidity reference
-> BSL
  -> active upside target
  -> taken BSL event
    -> bearish sweep 또는 upside breakout 분류

confirmed lower liquidity reference
-> SSL
  -> active downside target
  -> taken SSL event
    -> bullish sweep 또는 downside breakout 분류
```

## 코드 구현 시 문제점

### 사람이 보는 기준

사람은 active BSL과 SSL 중 HTF narrative, source 계층, 세션, 거리, 이미 회수된 liquidity를 함께 보고 주요 draw를 선택한다. BSL을 회수한 뒤 즉시 반전할지 계속 확장할지도 차트 문맥으로 판단한다.

### 코드로 구현 가능한 부분

- upper reference를 BSL, lower reference를 SSL로 분류
- active/taken availability 동기화
- 현재 가격 기준 directional target eligibility 계산
- BSL/SSL trigger 가격 계산
- 목표까지 tick 거리와 ATR 거리 계산
- 반대편 active target 목록 생성
- HTF, source 종류, confluence metadata 전달
- taken event를 sweep classifier에 전달

### 코드로 구현 어려운 부분

- 복수 BSL/SSL 중 narrative상 주요 draw 하나 선택
- 관통 직후 sweep과 continuation 중 어느 쪽이 우세한지 BSL/SSL만으로 판정
- 실제 stop 수량과 주문 주체 확인
- HTF 수동 reference와 자동 source가 충돌할 때 우선순위 선택
- 첫 partial과 최종 목표를 전략 없이 자동 결정

어려운 부분은 `manual_review_reason` 또는 strategy-specific ranking으로 분리한다.

## 핵심 문장

1. BSL은 고점 위 buy stops를 가정한 상단 liquidity reference이다.
2. SSL은 저점 아래 sell stops를 가정한 하단 liquidity reference이다.
3. BSL과 SSL은 trade direction 신호가 아니다.
4. upper liquidity source만 BSL로 분류한다.
5. lower liquidity source만 SSL로 분류한다.
6. BSL 관통은 `taken BSL`, SSL 관통은 `taken SSL` 사건이다.
7. `taken`은 반전 확정이 아니므로 sweep과 breakout을 후행 분류한다.
8. active BSL은 long 목표가 될 수 있고, taken BSL은 short 반전 후보의 입력이 될 수 있다.
9. active SSL은 short 목표가 될 수 있고, taken SSL은 long 반전 후보의 입력이 될 수 있다.
10. 복수 BSL/SSL 중 주요 draw 선택은 registry 분류와 분리된 ranking 문제이다.

## 이 개념의 본질

BSL/SSL은 stop 기반 liquidity registry를 상단 buy-stop 후보와 하단 sell-stop 후보로 분류하여 목표 방향과 회수 사건을 코드가 혼동 없이 처리하게 만드는 의미 label이다.

## Transcript 근거 분류

### 에피소드 전반에 반복되는 핵심 근거

- Episode 2: short-term low 아래 sell stops, old high 위 buy stops, old low 아래 sell stops 설명
- Episode 3: old low 아래 sell stops와 relative equal highs 위 buy stops를 동시에 표시
- Episode 6: highs 위 buy stops를 BSL로, old low 또는 복수 lows 아래를 SSL로 설명
- Episode 12: 위쪽 BSL 또는 imbalance, 아래쪽 SSL 또는 imbalance를 가격 전달 후보로 구분
- Episode 17: buy side와 buy stops, sell stops와 SSL을 직접 연결하고 한쪽 회수 후 반대편 후보를 설명
- Episode 25: relative equal highs 위 BSL 회수 후 하단 SSL 공격 사례
- Episode 26: SSL 회수 후 short-term high BSL을 다음 draw로 사용하는 tape reading 사례
- Episode 39, 40: BSL/SSL을 draw, partial, bias 문맥에서 사용

### 특정 구간에 집중된 보조 근거

- Episode 3: `offering buy side liquidity`, `offering sell side liquidity`와 state of delivery 문맥
- Episode 13: 고점 위 buy stop이 market buy order로 바뀌는 시장 심리 설명
- Episode 40: BSL initial draw를 첫 partial로 사용하는 청산 예시

### 검색 분포

정리본 `transcripts/*.en.txt` 전체 검색 기준으로 BSL 표기 계열은 16개 에피소드 파일, SSL 표기 계열은 15개, `buy stops`는 20개, `sell stops`는 16개에 등장한다. 자동 자막의 `bicep liquidity`, `cell side liquidity` 같은 오인식은 문맥을 대조해 BSL/SSL로 정규화한다.
