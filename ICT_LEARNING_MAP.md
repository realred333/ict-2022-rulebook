# ICT 2022 Mentorship Learning Map

## 목적

이 저장소의 문서는 자막 요약이 아니다. 목표는 자막에서 반복되는 설명을 다음 파이프라인으로 변환하는 것이다.

```text
transcripts -> ICT Wiki -> ICT Rulebook -> Pine Script -> Python Scanner
```

개념 문서는 의미와 연결 관계를 설명한다. Rulebook은 OHLC 데이터에서 판별할 수 있는 규칙, 판별할 수 없는 주관적 요소, 최소 구현과 고급 구현을 분리한다.

## 코퍼스 범위

- 기준 코퍼스: `transcripts/*.en.txt` 중 `-orig`가 없는 정리본 40개
- 대조 코퍼스: 표현이 불명확할 때 사용하는 대응 `*-orig.txt`
- 누락: 현재 저장소에는 Episode 28 파일이 없다.
- 주의: 정리본은 자막 문장이 반복되는 구간을 포함한다. 용어 횟수는 절대 빈도가 아니라 반복 등장 여부와 에피소드 분포를 확인하는 보조 자료로만 사용한다.

## 핵심 개념

| 그룹 | 개념 | 역할 |
| --- | --- | --- |
| 가격 구조 | [Swing Points](concepts/swing_points.md) | 구조와 유동성 후보를 만드는 최소 단위 |
| 유동성 | [Liquidity](concepts/liquidity.md), [BSL / SSL](concepts/bsl_ssl.md), [Liquidity Sweep](concepts/liquidity_sweep.md) | 가격이 탐색할 후보와 반전 전 사건 |
| 범위 | [Dealing Range](concepts/dealing_range.md), [Premium / Discount / Equilibrium](concepts/premium_discount_equilibrium.md) | 가격 위치를 정규화하는 좌표계 |
| 가격 전달 | [Displacement](concepts/displacement.md), [Fair Value Gap](concepts/fair_value_gap.md), [Market Structure Shift](concepts/market_structure_shift.md) | 방향 변화와 불균형을 확인하는 핵심 사건 |
| PD Array | [Order Block](concepts/order_block.md), [Breaker Block](concepts/breaker_block.md), [OTE](concepts/ote.md) | 되돌림 관찰 구간과 진입 후보 |
| 시간 | [Opening Price](concepts/opening_price.md), [Killzone](concepts/killzone.md), [Daily Bias](concepts/daily_bias.md), [Judas Swing](concepts/judas_swing.md), [Power of Three](concepts/power_of_three.md) | 가격 사건을 시간 문맥에 배치 |
| 고급 필터 | [Institutional Order Flow](concepts/institutional_order_flow.md), [SMT Divergence](concepts/smt_divergence.md), [Economic Calendar Filter](concepts/economic_calendar_filter.md) | 후보를 줄이고 별도 검토 대상으로 분류 |

## 의존 관계

아래 화살표는 항상 발생한다는 뜻이 아니다. scanner에서 후행 조건을 평가하려면 선행 데이터가 필요하다는 뜻이다.

```text
Swing Points
  -> Liquidity
    -> BSL / SSL
      -> Liquidity Sweep
      -> Market Structure Shift context

Swing Points
  -> Dealing Range
    -> Premium / Discount / Equilibrium
      -> OTE

Liquidity Sweep
  -> Displacement
    -> Market Structure Shift confluence metadata
    -> Fair Value Gap
      -> Order Block confluence metadata

BSL / SSL taken
  -> Breaker Block

Order Block overrun history
  -> Breaker Block optional provenance link

Opening Price
  -> Judas Swing
  -> Power of Three

Killzone + Economic Calendar Filter
  -> intraday candidate time filter

Daily Bias + Institutional Order Flow
  -> directional filter

correlated symbols + Swing Points
  -> SMT Divergence
    -> candidate confidence metadata
```

MSS 최소 event의 hard dependency는 사전 확정 swing point와 반대편 BSL/SSL `taken` event이다. confirmed sweep과 displacement는 거래 후보를 강화하는 confluence metadata다. Episode 3의 최소 MSS는 swing level strict traversal을 사용하며 level 바깥 종가 마감을 강제하지 않는다. 종가 이탈과 energetic displacement를 함께 요구하는 Episode 6 모델은 강화된 전략 profile로 분리한다.

FVG 최소 event의 hard dependency는 같은 symbol과 timeframe의 연속된 닫힌 OHLC 3개뿐이다. displacement, MSS, liquidity event는 raw FVG zone을 생성하는 조건이 아니라 retracement setup을 강화하는 confluence metadata다. raw zone 이력과 setup eligibility를 분리한다.

OB 최소 event의 hard dependency는 같은 symbol과 timeframe의 연속 directional close series와 series 시작 open의 반대 방향 strict traversal이다. 마지막 반대색 봉 하나로 고정하지 않는다. FVG, liquidity event, MSS, displacement는 raw OB를 생성하는 조건이 아니라 retracement setup을 강화하는 confluence metadata다.

Breaker Block 최소 event의 hard dependency는 확정 BSL 또는 SSL `taken` event, 반대편 확정 short-term swing의 strict traversal, source candle zone이다. bearish source는 linked high를 제거한 상승 이동 직전 source window의 lowest down-close candle이다. bullish source는 방향 대칭 구현 규칙에 따른 highest up-close candle이며 `symmetry_inference`로 기록한다. OB overrun 이력은 nullable provenance link이고 hard dependency가 아니다. narrative, displacement, MSS, FVG는 setup eligibility와 ranking metadata다.

OTE 최소 zone의 hard dependency는 선택 시각과 endpoint provenance가 저장된 유효한 dealing range, range 방향, anchor policy이다. 반복 자막 근거가 있는 corpus default는 `0.62~0.79` retracement이고 `0.705`는 두 경계의 산술 중간값인 derived metadata다. Episode 22의 open-close body anchor는 별도 refinement policy다. FVG, OB, MSS, liquidity event는 raw OTE zone 생성 조건이 아니라 setup eligibility와 ranking metadata다. Episode 41의 `62 to 70` 전사는 영상 수동 확인이 필요한 `transcript_conflict`로 보존한다.

실전 후보 파이프라인은 다음처럼 구성한다.

```text
HTF draw on liquidity
  -> active session and calendar window
  -> BSL or SSL sweep
  -> opposite-direction displacement
  -> MSS confirmation
  -> FVG or confirmed OB retracement
  -> target liquidity
```

위 실전 후보 파이프라인은 강화된 setup profile이다. 최소 MSS 탐지 규칙 자체를 뜻하지 않는다.

## 초보자 학습 순서

1. [Swing Points](concepts/swing_points.md): 고점과 저점을 동일한 알고리즘으로 표시한다.
2. [Liquidity](concepts/liquidity.md), [BSL / SSL](concepts/bsl_ssl.md): swing, relative equal highs/lows, old highs/lows를 유동성 후보로 분류한다.
3. [Dealing Range](concepts/dealing_range.md), [Premium / Discount / Equilibrium](concepts/premium_discount_equilibrium.md): 현재 가격의 상대 위치를 계산한다.
4. [Displacement](concepts/displacement.md), [Fair Value Gap](concepts/fair_value_gap.md): 큰 방향성 이동과 3캔들 불균형을 탐지한다.
5. [Liquidity Sweep](concepts/liquidity_sweep.md), [Market Structure Shift](concepts/market_structure_shift.md): 단순 돌파와 유동성 회수 후 방향 변화를 분리한다.
6. [Order Block](concepts/order_block.md), [Breaker Block](concepts/breaker_block.md), [OTE](concepts/ote.md): 되돌림 후보를 구조 조건과 결합한다.
7. [Opening Price](concepts/opening_price.md), [Killzone](concepts/killzone.md), [Judas Swing](concepts/judas_swing.md), [Power of Three](concepts/power_of_three.md): 동일한 가격 패턴을 시간 문맥에서 필터링한다.
8. [Daily Bias](concepts/daily_bias.md), [Institutional Order Flow](concepts/institutional_order_flow.md): 상위 시간 프레임 필터를 적용한다.
9. [SMT Divergence](concepts/smt_divergence.md), [Economic Calendar Filter](concepts/economic_calendar_filter.md): 상관 자산과 외부 이벤트 데이터를 추가한다.

## 구현 순서

1. OHLC 정규화, New York timezone 변환, tick size 설정
2. swing high/low와 relative equal highs/lows 탐지
3. BSL/SSL 후보 레지스트리와 sweep 상태 머신
4. dealing range, equilibrium, premium, discount 계산
5. displacement, FVG, MSS 탐지
6. OB, breaker, OTE 후보 생성
7. opening price, killzone, Judas swing, Power of Three 메타데이터
8. HTF daily bias와 institutional order flow 필터
9. 다중 심볼 SMT와 경제 캘린더 adapter
10. 후보별 근거를 JSON으로 출력하고 차트 overlay로 검증

## 객관화 원칙

- scanner의 결과는 `true/false` 하나로 축약하지 않는다. `candidate`, `confirmed`, `invalidated`, `expired`, `manual_review` 상태를 사용한다.
- 자막이 수치 임계값을 고정하지 않은 경우 Rulebook 기본값은 `research default`이다. 백테스트 가능한 파라미터로 노출한다.
- `명확한`, `강한`, `이상적인` 같은 형용사는 판별 규칙이 아니다. body ratio, ATR, median body, tick distance, 시간 창으로 바꾼다.
- 자막만으로 영상 차트의 종목, 날짜, 정확한 가격 좌표를 항상 복원할 수 없다. 각 개념 문서의 차트 예시는 OHLC에서 재현 가능한 검증 템플릿으로 제공한다.

## 주요 자막 근거

- 상세 색인: [TRANSCRIPT_SOURCE_INDEX.md](TRANSCRIPT_SOURCE_INDEX.md)
- Episode 3: internal range liquidity, MSS, FVG, OB의 연결
- Episode 5: displacement 이후 FVG 되돌림
- Episode 6: institutional order flow와 3캔들 FVG
- Episode 8: New York killzone
- Episode 9~10: Power of Three, opening price, economic calendar
- Episode 12: short-term low break와 MSS 이후 imbalance 되돌림
- Episode 35, 38: SMT divergence와 correlated index 비교
- Episode 39: New York midnight opening price와 bearish day 예시
- Episode 40: daily bias 단순화 규칙, killzone과 경제 캘린더 결합
