# Market Structure Shift

대응 Rulebook: [market_structure_shift_rulebook.md](../rulebook/market_structure_shift_rulebook.md)

## 개념 정의

### 일반적인 정의

Market Structure Shift(MSS)는 liquidity run 이후 가격이 반대 방향의 단기 구조를 관통하는 사건이다. ICT 2022 Mentorship에서는 `market structure shift`, `shift in market structure`, `market structure shifted`가 함께 사용된다.

MSS는 단순히 아무 swing이나 넘는 사건이 아니다. 어떤 방향의 liquidity가 먼저 회수되었고, 그 뒤 반대 방향 short-term swing이 관통되었는지를 함께 본다.

### ICT 정의

Episode 3의 기본 모델은 다음과 같다.

```text
buy stops 또는 BSL taken
-> short-term low 아래 거래
-> bearish MSS

sell stops 또는 SSL taken
-> short-term high 위 거래
-> bullish MSS
```

Episode 3은 bullish 예시에서 short-term high 위로 거래했으면 해당 high 위 종가 마감이 필수는 아니라고 명시한다. 따라서 자막 근거에 가장 가까운 최소 판정은 `strict traversal`이다.

```text
bullish MSS traversal = high > confirmed short-term swing high
bearish MSS traversal = low < confirmed short-term swing low
```

동일 가격 touch는 traversal이 아니다. scanner는 traversal 봉이 닫힌 뒤 event를 확정한다. 봉 내부에서 처음 level을 넘은 시각은 OHLC만으로 복원할 수 없다.

Episode 6은 더 강한 거래 후보 프로필을 설명한다.

```text
sell stops taken
-> short-term high 위 종가 마감
-> energetic displacement higher
-> displacement leg 내부 FVG 탐색
```

즉, MSS 최소 정의와 강화된 거래 필터는 분리해야 한다.

| 구분 | 판정 | 역할 |
| --- | --- | --- |
| `traversal_confirmed` | swing level을 strict traversal | 자막 기반 최소 MSS |
| `close_confirmed` | 종가가 swing level 바깥에서 마감 | 강화된 구조 확인 tag |
| `displacement_confirmed` | 같은 방향 displacement가 연결됨 | 거래 후보 강화 tag |

`close_confirmed`와 `displacement_confirmed`는 유용하지만 모든 MSS의 필수 조건으로 강제하지 않는다.

### 차트에서 의미

MSS는 liquidity 회수 뒤 가격 전달 방향이 바뀌는지 확인한다.

```text
BSL taken
-> bearish MSS traversal
-> bearish displacement confluence 확인
-> bearish FVG 또는 confirmed OB retracement 관찰

SSL taken
-> bullish MSS traversal
-> bullish displacement confluence 확인
-> bullish FVG 또는 confirmed OB retracement 관찰
```

MSS는 반전 후보를 구조적으로 좁힌다. 그러나 MSS 자체는 진입도 아니고 목표가 도달 보장도 아니다.

### 시장 심리

ICT의 설명 모델은 다음과 같다.

- old high, old low, relative equal highs/lows 바깥의 stops가 먼저 회수된다.
- 이후 반대 방향 short-term swing이 관통되면 기존 intraday delivery가 바뀌는 후보로 본다.
- 같은 leg 안의 imbalance 또는 FVG 되돌림을 관찰한다.
- 선행 liquidity run이 없으면 같은 swing break라도 의미가 약하다.

OHLC 코드는 실제 stop 주문 수량, 알고리즘 주문 주체, 기관 주문을 확인할 수 없다. MSS는 `liquidity taken + opposite swing traversal`을 사용한 가격 행동 프록시이다.

## ICT 전체 체계에서의 위치

### 선행 개념

- 필수: [Swing Points](swing_points.md)
- 필수 문맥: [BSL / SSL](bsl_ssl.md)의 opposing-side `taken` event
- 권장 confluence: [Liquidity Sweep](liquidity_sweep.md)
- 권장 confluence: [Displacement](displacement.md)
- 고급 문맥: HTF draw on liquidity, session, economic calendar

MSS 최소 판정은 confirmed sweep을 필수로 요구하지 않는다. 자막은 stop run 또는 liquidity taken 뒤 구조 변화를 설명한다. Sweep 모듈의 reclaim window는 별도의 `research default`이므로 MSS 최소 정의에 강제로 섞지 않는다.

### 후행 개념

- [Fair Value Gap](fair_value_gap.md)
- [Order Block](order_block.md)
- [Breaker Block](breaker_block.md)
- [Dealing Range](dealing_range.md)
- retracement entry model

### 왜 중요한가

단순 swing break는 trend continuation, liquidity run, noise일 수 있다. MSS는 opposing-side liquidity가 먼저 회수되었다는 문맥을 추가한다.

```text
raw swing traversal
!= contextual MSS

contextual MSS
= opposing liquidity taken
  + opposite-direction confirmed swing traversal
```

Episode 3은 market structure `break`보다 intraday `shift`라는 용어를 사용하는 이유를 설명한다. 해당 사건이 하루 전체 추세 전환을 반드시 의미하지 않고, intraday price leg를 만들 수 있기 때문이다.

## 실제 차트에서 발견하는 방법

### 찾는 순서

1. symbol과 timeframe별 confirmed short-term swing high/low를 읽는다.
2. BSL 또는 SSL의 최초 strict traversal인 `taken` event를 받는다.
3. BSL taken이면 bearish MSS 후보를, SSL taken이면 bullish MSS 후보를 연다.
4. 후보 window 안에서 반대 방향 confirmed short-term swing reference를 추적한다.
5. bullish 후보는 `high > swing_high.level`, bearish 후보는 `low < swing_low.level`인지 닫힌 봉마다 검사한다.
6. traversal이 발생하면 MSS를 `traversal_confirmed`로 확정한다.
7. 같은 봉 종가가 level 바깥에 있으면 `close_confirmed = true`를 추가한다.
8. 같은 방향 displacement event가 연결되면 `displacement_confirmed = true`를 추가한다.
9. 같은 displacement leg 내부 FVG와 retracement 후보를 별도 모듈에서 연결한다.

### 유효 조건

- broken swing은 traversal 전에 확정되어 있어야 한다.
- swing의 `pivot_at`과 `confirmed_at`을 분리한다.
- bullish MSS는 SSL 또는 sell-side liquidity taken 뒤 short-term swing high 위 strict traversal이다.
- bearish MSS는 BSL 또는 buy-side liquidity taken 뒤 short-term swing low 아래 strict traversal이다.
- 동일 가격 touch는 MSS가 아니다.
- event는 traversal 봉 close 이후 확정한다.
- reference 선택 정책과 후보 expiry window를 parameters version에 저장한다.
- close 이탈 여부와 displacement 연결 여부를 별도 필드로 저장한다.

### 무효 조건

- 미래 봉을 사용해 swing이 확정되기 전에 MSS를 표시한다.
- 동일 가격 touch를 strict traversal로 처리한다.
- opposing-side liquidity taken 문맥 없이 모든 swing break를 MSS로 부른다.
- 종가 이탈을 모든 MSS의 필수 조건이라고 설명한다.
- MSS traversal만으로 진입을 확정한다.
- MSS와 displacement를 같은 event로 합친다.
- FVG 존재만으로 MSS를 추론한다.
- HTF와 LTF 구조 사건을 같은 timeframe event로 합친다.

### 대표 예시: Bullish MSS

```text
active SSL trigger:                 100.00
SSL taken at t:                      99.50
confirmed short-term swing high:    102.00

bar t + 2:
  high  = 102.25
  close = 101.90

high > 102.00
close <= 102.00
-> bullish MSS traversal_confirmed
-> close_confirmed = false
```

Episode 3의 최소 정의에 맞는 사례다. level 위 종가 마감은 필수가 아니다.

### 대표 예시: 강화된 Bullish MSS

```text
active SSL trigger:                 100.00
SSL taken at t:                      99.50
confirmed short-term swing high:    102.00

bar t + 2:
  open  = 101.00
  high  = 103.25
  low   = 100.75
  close = 103.00

high > 102.00
close > 102.00
same-direction displacement event linked
-> bullish MSS traversal_confirmed
-> close_confirmed = true
-> displacement_confirmed = true
```

Episode 6의 강화된 거래 후보 프로필에 가깝다.

### 대표 예시: Bearish MSS

```text
active BSL trigger:                 105.00
BSL taken at t:                     105.50
confirmed short-term swing low:     103.00

bar t + 1:
  low   = 102.75
  close = 103.10

low < 103.00
close >= 103.00
-> bearish MSS traversal_confirmed
-> close_confirmed = false
```

### 단순 Swing Break와 MSS 구분

| 사건 | opposing liquidity taken | 반대 swing strict traversal | 분류 |
| --- | --- | --- | --- |
| touch | 무관 | 없음 | 관찰만 기록 |
| raw structure traversal | 없음 또는 미확인 | 있음 | 구조 관통 event |
| contextual MSS | 있음 | 있음 | MSS confirmed |
| strengthened MSS profile | 있음 | 있음, 종가 이탈 및 displacement 연결 | 전략 ranking 상향 |

raw structure traversal을 버리지 않는다. 별도 table에 저장하면 context 정책을 바꿔 다시 연구할 수 있다.

### Broken Swing 선택

자막은 모든 상황에 공통인 단일 reference 선택 알고리즘을 고정하지 않는다.

- Episode 3 bearish 예시: buy stops taken 뒤 형성된 short-term low를 관찰한다.
- Episode 6 bearish 예시: old high run 이전 short-term low break를 사용한다.
- Episode 12: short-term, intermediate-term 구조 계층을 함께 설명한다.

따라서 코드는 모든 eligible reference id를 저장하고, primary reference 선택 정책을 versioned parameter로 관리한다.

최소 구현의 초기 정책:

```text
reference_strength = short_term
primary_reference = most recent eligible confirmed swing
```

이 정책은 ICT가 직접 고정한 값이 아니라 `research default`이다.

## 실전 활용

### 진입

MSS traversal에서 즉시 추격 진입하지 않는다.

```text
BSL taken 또는 BSL sweep
-> bearish MSS
-> bearish displacement confluence
-> bearish FVG 또는 confirmed OB retracement
-> short 후보

SSL taken 또는 SSL sweep
-> bullish MSS
-> bullish displacement confluence
-> bullish FVG 또는 confirmed OB retracement
-> long 후보
```

Episode 3, 6, 7, 12는 MSS 뒤 FVG 또는 imbalance 되돌림을 관찰하는 흐름을 반복한다.

### 손절

MSS는 손절을 직접 정하지 않는다.

- long 후보: SSL sweep extreme 또는 구조 low 아래 buffer 검토
- short 후보: BSL sweep extreme 또는 구조 high 위 buffer 검토
- broken swing 자체를 손절 가격으로 자동 사용하지 않는다.

### 목표

- bullish MSS 이후: 현재 가격 위 active BSL
- bearish MSS 이후: 현재 가격 아래 active SSL
- 목표 후보마다 source kind, timeframe, distance, IRL/ERL tag를 기록한다.

### 시간대

MSS 기본 탐지는 timeframe과 시간대에 독립적이다. Episode 6은 모든 timeframe에서 적용된다고 설명하면서 intraday 모델에 집중한다.

- Episode 3: 1분, 2분, 3분 intraday chart와 08:30~12:00 New York local time 연구 과제
- Episode 22: 5분 chart MSS 예시
- Episode 41: 5분 chart와 1:30 시간 문맥 예시

세션과 경제지표는 MSS 정의가 아니라 metadata 및 전략 필터이다.

## 흔한 오해

### 초보자가 잘못 해석하는 부분

- 모든 swing break를 MSS라고 부른다.
- wick traversal은 항상 무효이고 종가 이탈만 MSS라고 생각한다.
- 반대로 wick traversal만 있으면 무조건 진입한다.
- 선행 BSL/SSL taken과 반대 방향 구조 관통을 연결하지 않는다.
- liquidity sweep, displacement, MSS를 하나의 event로 합친다.
- close-confirmed 강화 프로필을 ICT 최소 정의와 혼동한다.
- MSS가 있으면 FVG 또는 OB도 자동으로 있다고 가정한다.
- HTF structure shift와 LTF entry trigger를 같은 event로 합친다.

### ICT가 실제로 의미하는 것

- MSS는 liquidity run 이후 반대 방향 short-term swing 관통을 보는 문맥 기반 사건이다.
- Episode 3의 최소 예시는 broken high 위 종가 마감을 요구하지 않는다.
- Episode 6의 강화된 모델은 energetic displacement와 종가 이탈을 함께 본다.
- MSS 이후 같은 price leg의 FVG 또는 imbalance 되돌림을 관찰한다.
- intraday shift는 반드시 장기 추세 반전을 의미하지 않는다.

## 다른 개념과의 연결

### 관련 개념

| 연결 개념 | MSS의 역할 |
| --- | --- |
| Swing Points | 반대 방향 structure reference 제공 |
| Liquidity | 선행 stop pool 문맥 제공 |
| BSL / SSL | opposing-side taken event 제공 |
| Liquidity Sweep | reclaim이 확인된 강화 문맥 제공 |
| Displacement | 같은 방향 energetic repricing confluence 제공 |
| FVG | MSS leg 내부 retracement 후보 제공 |
| Order Block | delivery 변화와 retracement 구간 후보 제공 |
| Dealing Range | MSS leg high/low를 range 후보로 전달 |
| SMT Divergence | correlated symbol 사이 구조 변화 차이 metadata |
| Economic Calendar Filter | event window metadata |

### 의존 관계

```text
confirmed swing points
+ opposing BSL 또는 SSL taken event
-> contextual MSS traversal

contextual MSS traversal
+ optional close confirmation
+ optional displacement confluence
+ optional confirmed sweep link
-> ranked MSS setup context

ranked MSS setup context
-> FVG 또는 confirmed OB retracement 후보
```

## 코드 구현 시 문제점

### 사람이 보는 기준

사람은 어떤 swing이 narrative상 중요한지, stop run 뒤 움직임이 얼마나 energetic한지, 어느 timeframe의 structure를 우선할지 함께 본다. `significant`, `meaningful`, `proper context`는 유용한 설명이지만 그대로는 코드 규칙이 아니다.

### 코드로 구현 가능한 부분

- pre-confirmed short-term swing registry 조회
- BSL/SSL taken event와 방향 매핑
- strict traversal 판정
- same-level touch 제외
- traversal 봉 close 이후 확정
- close 이탈 여부 tag
- same-direction displacement link
- confirmed sweep, FVG, OB link
- reference strength와 timeframe 저장
- 후보 window expiry

### 코드로 구현 어려운 부분

- 복수 swing 중 narrative상 가장 중요한 reference 선택
- 모든 symbol과 timeframe에 공통인 MSS 후보 expiry window
- wick-only traversal과 close-confirmed traversal의 전략별 우위
- HTF shift와 LTF trigger의 최적 결합 방식
- OHLC 한 봉 안에서 liquidity taken과 structure traversal의 실제 순서

어려운 부분은 `research default`, `manual_review_reason`, 백테스트 cohort로 관리한다.

## 핵심 문장

1. MSS는 opposing liquidity taken 뒤 반대 방향 short-term swing을 관통하는 문맥 기반 사건이다.
2. Bullish MSS는 sell-side liquidity taken 뒤 confirmed swing high 위 strict traversal이다.
3. Bearish MSS는 buy-side liquidity taken 뒤 confirmed swing low 아래 strict traversal이다.
4. Broken swing은 traversal 전에 확정되어 있어야 한다.
5. Episode 3의 최소 MSS는 swing level 바깥 종가 마감을 필수로 요구하지 않는다.
6. 종가 이탈은 `close_confirmed` 강화 tag로 별도 저장한다.
7. Energetic displacement 연결은 `displacement_confirmed` 강화 tag로 별도 저장한다.
8. Confirmed sweep은 유용한 confluence지만 MSS 최소 정의의 필수 조건은 아니다.
9. MSS, displacement, FVG는 연결되지만 서로 다른 event이다.
10. MSS 뒤 retracement model을 기다리고 즉시 추격 진입하지 않는다.

## 이 개념의 본질

Market Structure Shift는 opposing-side liquidity taken 뒤 반대 방향의 사전 확정 short-term swing이 strict traversal되었는지를 기록하고, 종가 이탈과 displacement를 별도 강화 tag로 연결하는 문맥 기반 구조 변화 event이다.

## Transcript 근거 분류

### 에피소드 전반에 반복되는 핵심 근거

- Episode 3: market structure `break`보다 intraday `shift` 용어를 사용하는 이유
- Episode 3: relative equal highs run 뒤 MSS를 기다리는 흐름
- Episode 3: bullish 예시에서 short-term high 위 거래는 필요하지만 종가 마감은 필수가 아니라는 설명
- Episode 3: buy stops taken 뒤 형성된 short-term low 아래 거래를 bearish MSS로 분류
- Episode 3: stop hunt 뒤 1분, 2분, 3분 chart MSS signature를 연구하는 과제
- Episode 6: bearish MSS, displacement high/low, 내부 FVG 연결
- Episode 6: bullish 강화 모델에서 short-term high 위 종가와 energetic displacement 확인
- Episode 7: sell-side liquidity taken 뒤 swing high break와 bullish MSS
- Episode 12: short-term low break 뒤 MSS와 imbalance retracement
- Episode 40: displacement 뒤 short-term low taken과 MSS, imbalance 연결

### 특정 구간에 집중된 보조 근거

- Episode 16: swing low break, MSS, FVG 연결 사례
- Episode 22: 5분 chart MSS와 FVG 연결 사례
- Episode 29: short-term MSS 뒤 FVG 되돌림 사례
- Episode 36: sell-side run 뒤 MSS와 FVG 되돌림 사례
- Episode 41: 5분 chart 및 1:30 시간 문맥 MSS 사례

### 검색 분포

정리본 `transcripts/*.en.txt` 전체 검색 기준:

| 표현 | 검색 라인 수 | 등장 에피소드 파일 수 |
| --- | ---: | ---: |
| `market structure shift` | 61 | 7 |
| `shift in market structure` | 66 | 13 |
| `market structure shifted` | 3 | 1 |
| `market structure break` | 15 | 3 |
| `break in market structure` | 6 | 2 |

위 표현의 합집합은 18개 에피소드 파일에 등장한다. 수치는 자막 중복을 포함하므로 절대 빈도가 아니라 반복 분포 확인용이다.
