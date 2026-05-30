# Liquidity

대응 Rulebook: [liquidity_rulebook.md](../rulebook/liquidity_rulebook.md)

## 개념 정의

### 일반적인 정의

일반적으로 liquidity는 주문을 큰 가격 왜곡 없이 체결할 수 있는 정도를 뜻한다. 그러나 OHLC 차트만으로 특정 가격의 실제 대기 주문 수량을 직접 관측할 수는 없다.

### ICT 정의

ICT 2022 Mentorship에서 liquidity는 주로 stop 주문이 대기할 가능성이 있는 가격대를 뜻한다.

- old high와 swing high 위: buy stops가 존재할 가능성이 있는 상단 reference
- old low와 swing low 아래: sell stops가 존재할 가능성이 있는 하단 reference
- relative equal highs 위: 복수 고점이 만든 상단 stop pool 후보
- relative equal lows 아래: 복수 저점이 만든 하단 stop pool 후보
- 이전 일자, 세션, overnight의 high/low: 시간 구간이 만든 stop pool 후보

Episode 2는 시장이 찾아가는 대상을 `stops`, 즉 liquidity와 `imbalance`로 구분한다. Episode 3은 old low 아래의 sell stops와 relative equal highs 위의 buy stops를 함께 설명한다. Episode 17과 25에서도 buy stops와 buy-side liquidity, sell stops와 sell-side liquidity가 같은 문맥에서 반복된다.

이 저장소는 실제 주문이 존재한다고 확정하지 않는다. OHLC에서 재현 가능한 가격 구조를 `liquidity reference candidate`로 기록한다.

### 차트에서 의미

Liquidity reference는 진입 신호가 아니라 가격 전달 과정에서 관찰할 목표 후보이다.

```text
confirmed price reference
-> active liquidity candidate
-> price trades through level
-> taken liquidity event
-> sweep 또는 breakout 분류
-> displacement와 MSS 확인
```

가격이 어느 liquidity를 향하는지에 대한 가설은 `draw on liquidity`이다. 자막에서 이 표현은 여러 에피소드에 반복되지만, OHLC만으로 단 하나의 draw를 확정할 수는 없다. scanner는 가능한 목표 목록과 근거를 출력해야 한다.

### 시장 심리

ICT의 설명 모델은 다음과 같다.

- short 포지션의 보호 stop은 고점 위에 놓일 수 있다.
- long 포지션의 보호 stop은 저점 아래에 놓일 수 있다.
- 가격이 고점 위로 거래되면 buy stop이 market buy order로 전환될 수 있다.
- 가격이 저점 아래로 거래되면 sell stop이 market sell order로 전환될 수 있다.

Episode 13은 buy stop이 시장가 주문으로 바뀌어 반대편 체결 유동성을 제공하는 과정을 설명한다. 이는 시장 심리를 이해하기 위한 모델이다. OHLC만 사용하는 코드는 주문 주체, 주문 수량, 실제 체결 의도를 판별할 수 없다.

## ICT 전체 체계에서의 위치

### 선행 개념

- 필수: [Swing Points](swing_points.md)
- 선택: 세션 경계, 일자 경계, ATR, tick size
- internal/external 분류 시 필수: [Dealing Range](dealing_range.md)
- imbalance를 delivery objective로 함께 다룰 때 필요: [Fair Value Gap](fair_value_gap.md)

### 후행 개념

- [BSL / SSL](bsl_ssl.md)
- [Liquidity Sweep](liquidity_sweep.md)
- [Daily Bias](daily_bias.md)
- [Market Structure Shift](market_structure_shift.md)
- [Judas Swing](judas_swing.md)

### 왜 중요한가

Liquidity는 진입 패턴보다 먼저 계산해야 하는 가격 지도이다. 동일한 FVG, OB, MSS가 여러 개 있어도 어느 active liquidity를 향하거나 어느 liquidity를 회수한 뒤 발생했는지에 따라 해석이 달라진다.

학습 맵의 기본 흐름은 다음과 같다.

```text
Swing Points
-> Liquidity
  -> BSL / SSL
    -> Liquidity Sweep
      -> Displacement
        -> MSS
```

## 실제 차트에서 발견하는 방법

### 찾는 순서

1. symbol과 timeframe별로 닫힌 OHLC를 시간순으로 정렬한다.
2. 확정 swing high와 swing low를 불러온다.
3. swing high를 상단 liquidity source, swing low를 하단 liquidity source 후보로 등록한다.
4. 같은 방향의 확정 source 중 가격 차이가 tolerance 이내인 복수 레벨을 relative equal highs/lows cluster 후보로 묶는다.
5. 완료된 이전 일자와 세션의 high/low를 별도 source로 등록한다.
6. 각 source의 생성 시각 `available_at` 이후에만 active target으로 사용한다.
7. 이후 봉이 상단 레벨 위 또는 하단 레벨 아래를 거래하면 `taken`으로 변경한다.
8. 선택된 dealing range가 있을 때만 internal/external 위치 tag를 계산한다.

### 유효 조건

- source가 look-ahead 없이 확정되었다.
- source에 `symbol`, `timeframe`, `side`, `price_low`, `price_high`, `available_at`이 있다.
- 상단과 하단 source를 섞지 않는다.
- equal-level cluster는 같은 방향의 확정 source가 최소 2개이다.
- equal-level cluster의 `max(level) - min(level)`이 configured tolerance 이하이다.
- 완료되지 않은 일자나 세션의 최종 high/low를 미리 사용하지 않는다.
- `taken` 판정은 source가 사용 가능해진 시점 이후의 봉으로만 수행한다.

### 무효 조건

- 미래 swing 확인 봉을 미리 읽었다.
- 아직 끝나지 않은 세션의 최종 high/low를 확정 레벨처럼 사용했다.
- 이미 `taken`인 레벨을 새 목표로 재사용했다.
- tolerance를 공개하지 않고 육안으로 비슷한 고점과 저점을 묶었다.
- 서로 다른 symbol 또는 timeframe의 레벨을 하나의 cluster로 합쳤다.
- 실제 주문 데이터가 없는데 stop 수량이나 주문 주체를 확정했다.

### 대표 예시

```text
confirmed swing highs: 100.00, 100.25
tick_size:             0.25
ATR(14):               3.00
tolerance:             max(0.25, 0.10 * 3.00) = 0.30

100.25 - 100.00 = 0.25 <= 0.30
-> relative equal highs cluster
-> upper liquidity candidate

later bar high: 100.50
100.50 > 100.25
-> active -> taken
```

`0.10 * ATR(14)`는 자막에 고정된 수치가 아니다. 사람이 보는 relative equal level을 백테스트하기 위한 `research default`이다.

### Internal Range Liquidity와 External Range Liquidity

Internal/external은 독립적인 source가 아니라 선택한 dealing range에 상대적인 위치 분류이다.

- External Range Liquidity(ERL): 선택한 range의 high 이상에 있는 상단 stop pool 또는 range의 low 이하에 있는 하단 stop pool
- Internal Range Liquidity(IRL): 선택한 range 내부의 short-term high/low 기반 stop pool 또는 내부 imbalance
- unclassified: dealing range를 선택하지 않았거나 source 위치가 자동 분류 규칙으로 결정되지 않은 상태

Episode 3은 되돌림 leg 내부의 short-term high/low와 imbalance를 IRL 문맥에서 설명한다. Episode 6은 range 중간의 FVG를 internal range liquidity, 하단 low 아래의 stops를 external range liquidity로 구분한다.

코드에서는 stop pool과 imbalance를 같은 객체로 합치지 않는다.

```text
delivery_objective
  -> stop_pool_reference
  -> imbalance_reference
```

두 유형은 모두 가격 전달 목표 후보가 될 수 있지만 생성 조건과 회수 조건이 다르다.

### 보조 설명: 시간 구간 기반 레벨

Episode 18은 최근 3일의 high와 low를 표시하여 공격 가능한 liquidity pool을 찾는 방법을 설명한다. Episode 19, 22, 25에는 London 또는 overnight high/low 문맥이 등장한다.

최소 구현은 swing source만 사용한다. 이전 일자, 세션, overnight source는 timezone과 세션 정의가 필요한 고급 구현으로 분리한다.

## 실전 활용

### 진입

Liquidity 자체로 진입하지 않는다. 최소 반전 후보 파이프라인은 다음과 같다.

```text
active liquidity
-> liquidity taken
-> reclaim 여부로 sweep 또는 breakout 분류
-> 반대 방향 displacement
-> MSS
-> FVG 또는 confirmed OB retracement
```

### 손절

Liquidity level은 손절 위치를 자동 결정하지 않는다.

- long 모델: SSL 회수 후 구조 기준 swing low와 configured buffer를 사용한다.
- short 모델: BSL 회수 후 구조 기준 swing high와 configured buffer를 사용한다.
- liquidity level 자체를 손절로 사용할지는 진입 모델 rulebook에서 정한다.

### 목표

- long 후보: 현재 가격 위의 active upper liquidity를 목표 후보로 유지한다.
- short 후보: 현재 가격 아래의 active lower liquidity를 목표 후보로 유지한다.
- 이미 `taken`인 level은 신규 목표 후보에서 제외한다.
- 복수 목표가 있으면 distance, timeframe, source 종류, internal/external 위치를 출력한다.
- 단일 최종 목표 선택은 전략별 ranking 규칙이 있을 때만 수행한다.

Episode 40은 initial draw on liquidity를 첫 partial로 사용하는 예시를 설명한다. 이는 보조적인 청산 정책이며 liquidity 정의 자체는 아니다.

### 시간대

기본 liquidity 레지스트리는 시간대와 무관하게 유지한다. 이전 일자, session, overnight 레벨을 사용할 때는 `America/New_York` timezone과 세션 구간을 명시한다. killzone은 level 생성 규칙이 아니라 후보 필터이다.

## 흔한 오해

### 초보자가 잘못 해석하는 부분

- liquidity를 거래량 지표 또는 order book 잔량과 동일하게 본다.
- BSL은 매수 신호, SSL은 매도 신호라고 해석한다.
- 모든 swing 위아래에 동일한 우선순위를 준다.
- level을 한 번 관통하면 반드시 반전한다고 가정한다.
- imbalance와 stop pool을 같은 데이터 타입으로 저장한다.
- relative equal highs/lows를 눈대중으로 선택하고 tolerance를 기록하지 않는다.
- 과거 차트를 보고 세션 최종 high/low를 세션 진행 중에도 알 수 있었다고 가정한다.

### ICT가 실제로 의미하는 것

- liquidity는 가격이 탐색할 가능성이 있는 stop 기반 reference이다.
- 가격 관통은 `taken` 사건일 뿐 반전을 확정하지 않는다.
- 반전 후보는 sweep, displacement, MSS 같은 후행 사건으로 별도 확인한다.
- draw on liquidity는 목표 가설이다. 코드가 자동으로 확정할 수 없는 경우 후보 목록과 `manual_review_reason`을 남긴다.
- imbalance는 liquidity와 함께 가격 전달 목표가 될 수 있지만 stop pool과 구분한다.

## 다른 개념과의 연결

### 관련 개념

| 연결 개념 | liquidity의 역할 |
| --- | --- |
| Swing Points | stop pool 후보를 만드는 최소 가격 reference |
| BSL / SSL | 상단과 하단 liquidity의 방향 분류 |
| Liquidity Sweep | active level 관통 후 reclaim 사건 |
| Dealing Range | IRL과 ERL 위치 분류의 좌표계 |
| Fair Value Gap | stop pool과 구분되는 imbalance objective |
| Daily Bias | HTF draw on liquidity 후보를 방향 필터에 전달 |
| Judas Swing | opening price와 시간 조건이 추가된 liquidity raid 응용 |
| SMT Divergence | 상관 심볼 사이의 liquidity 회수 불일치 비교 |

### 의존 관계

```text
confirmed swing
-> stop_pool_reference
  -> upper | lower
  -> active | taken
  -> BSL | SSL
    -> sweep | breakout

selected dealing range
+ stop_pool_reference or imbalance_reference
-> internal_range | external_range | unclassified
```

## 코드 구현 시 문제점

### 사람이 보는 기준

사람은 HTF 방향, 세션, 상대적으로 가까운 old high/low, equal level의 모양, FVG 위치를 함께 보고 우선 목표를 고른다. 같은 방향에 active liquidity가 여러 개 있어도 하나를 narrative상 주요 draw로 선택할 수 있다.

### 코드로 구현 가능한 부분

- 확정 swing 기반 upper/lower liquidity source 생성
- relative equal highs/lows cluster 생성
- 완료된 이전 일자와 세션 high/low 생성
- `active -> taken` 상태 변경
- selected dealing range 기준 IRL/ERL 위치 tag
- 목표까지 tick distance 계산
- source 생성, 관통, 만료 이력 저장

### 코드로 구현 어려운 부분

- 복수 HTF level 중 narrative상 주요 draw 하나를 선택하는 작업
- 실제 stop 주문 수량과 주문 주체 판별
- 사람이 차트 문맥으로 선택하는 dealing range
- 자막에서 수치로 고정하지 않은 equal-level tolerance
- 같은 level을 sweep, breakout, continuation 중 무엇으로 해석할지 liquidity만으로 판단하는 작업

코드는 어려운 부분을 숨기지 않고 `manual_review_reason`과 strategy-specific parameter로 분리해야 한다.

## 핵심 문장

1. OHLC 코드는 실제 주문을 관측하지 않고 liquidity reference candidate를 생성한다.
2. old high와 swing high 위는 상단 stop pool 후보이다.
3. old low와 swing low 아래는 하단 stop pool 후보이다.
4. relative equal highs/lows는 공개된 tolerance 규칙으로만 cluster화한다.
5. liquidity level 관통은 반전 확정이 아니라 `taken` 사건이다.
6. `taken` level은 과거 이력으로 남기되 신규 목표에서는 제외한다.
7. draw on liquidity는 단일 확정값이 아니라 근거가 붙은 목표 후보 목록이다.
8. internal/external은 선택된 dealing range에 상대적인 위치 분류이다.
9. imbalance는 delivery objective가 될 수 있지만 stop pool과 같은 객체가 아니다.
10. liquidity 단독 진입을 금지하고 sweep, displacement, MSS를 후행 확인한다.

## 이 개념의 본질

Liquidity는 ICT 운영체제에서 가격이 탐색하거나 회수할 수 있는 stop 기반 가격 reference를 look-ahead 없이 등록하고 추적하는 목표 레지스트리이다.

## Transcript 근거 분류

### 에피소드 전반에 반복되는 핵심 근거

- Episode 2: 시장이 draw하는 대상을 stops, 즉 liquidity와 imbalance로 분리하고 old high 위 buy stops, old low 아래 sell stops를 설명
- Episode 3: old low 아래 sell stops, relative equal highs 위 buy stops, IRL, stop hunt 이후 MSS 연결
- Episode 5, 6, 7, 9, 10: old high/low, relative equal highs/lows, buy stops/sell stops 회수 사례
- Episode 17, 25: buy stops와 BSL, sell stops와 SSL을 같은 문맥에서 반복 설명
- Episode 18: 최근 3일 high/low를 liquidity pool 후보로 표시하는 보조 절차
- Episode 40: old high/low 또는 imbalance를 향하는 draw와 initial partial 예시

### 특정 구간에 집중된 보조 근거

- Episode 6: selected range 내부 FVG와 외부 stops를 IRL/ERL로 구분
- Episode 13: buy stop이 market order로 전환되는 시장 심리 설명
- Episode 19, 22, 25: London, overnight, session high/low 문맥

### 검색 분포

정리본 `transcripts/*.en.txt` 전체 검색 기준으로 `liquidity`는 36개 에피소드 파일, `relative equal`은 33개, `old high|old low`는 25개, `draw on liquidity`는 15개에 등장한다. 자막 중복 라인이 있으므로 횟수는 정의의 근거가 아니라 반복 분포 확인용으로만 사용한다.
