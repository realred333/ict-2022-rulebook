# Displacement

대응 Rulebook: [displacement_rulebook.md](../rulebook/displacement_rulebook.md)

## 개념 정의

### 일반적인 정의

Displacement는 가격이 짧은 시간 안에 한 방향으로 빠르게 재평가되는 움직임이다. ICT 자막에서는 `energetic`, `quick`, `sudden`, `obvious`, `animated` 같은 표현과 함께 반복된다.

### ICT 정의

ICT 2022 Mentorship에서 displacement는 작고 느린 움직임이 아니라 방향 의지가 드러나는 energetic repricing이다.

- bullish displacement: 가격이 위로 빠르게 이동하고, 구조 문맥에서는 이전 short-term high 위 종가 돌파를 동반할 수 있다.
- bearish displacement: 가격이 아래로 빠르게 이동하고, 구조 문맥에서는 이전 short-term low 아래 종가 돌파를 동반할 수 있다.
- displacement leg: liquidity raid extreme과 구조 돌파 사이에 형성된 방향성 가격 구간

Episode 5는 old high 위 거래 후 아래로 내려올 때 lethargic move가 아니라 obvious displacement를 찾아야 한다고 설명한다. Episode 6은 quick shift, energetic movement, swing 종가 돌파, displacement high/low, 그 내부 FVG를 연결한다. Episode 41은 FVG가 있어도 low 아래 의미 있는 displacement가 없으면 참여하지 않는 예시를 제공한다.

자막은 모든 symbol과 timeframe에 공통인 body 비율이나 ATR 임계값을 고정하지 않는다. 코드는 정성 표현을 백테스트 가능한 `research default`로 변환해야 한다.

### 차트에서 의미

Displacement는 가격이 단순히 liquidity를 건드린 것이 아니라 반대편으로 재평가되고 있는지 확인하는 필터이다.

```text
liquidity sweep
-> opposite-direction displacement
-> MSS
-> displacement leg 내부 FVG 탐색
-> retracement 후보
```

Displacement는 단일 봉으로 나타날 수도 있고 연속 봉 impulse leg로 나타날 수도 있다. 최소 구현은 단일 봉 후보를 탐지하고, 고급 구현은 여러 봉을 하나의 leg로 묶는다.

### 시장 심리

ICT의 설명 모델은 다음과 같다.

- liquidity가 회수된 뒤 가격이 빠르게 반대 방향으로 움직이면 해당 가격대에서 적극적인 repricing이 발생한 것으로 본다.
- 작은 wick 또는 느린 drift만 있으면 방향 의지가 충분히 확인되지 않은 것으로 본다.
- displacement leg 안의 FVG는 빠른 전달 과정에서 남은 imbalance 후보이다.

OHLC 코드는 알고리즘 주문, 기관 주문, 실제 체결 주체를 확인할 수 없다. displacement는 속도와 크기를 OHLC로 근사한 가격 행동 프록시이다.

## ICT 전체 체계에서의 위치

### 선행 개념

- 필수 입력: 닫힌 OHLC
- 권장: [Liquidity Sweep](liquidity_sweep.md)
- 구조 확인 시 필요: [Swing Points](swing_points.md)
- 상대 크기 계산 시 필요: ATR, 이전 body history, tick size

### 후행 개념

- [Market Structure Shift](market_structure_shift.md)
- [Fair Value Gap](fair_value_gap.md)
- [Order Block](order_block.md)
- [Dealing Range](dealing_range.md)

### 왜 중요한가

Displacement는 `관통했다`와 `의미 있는 repricing이 발생했다`를 분리한다. Episode 24는 old low 아래 sell stops를 회수한 뒤 displacement가 없으면 FVG long 모델을 찾지 않는 예시를 설명한다.

```text
taken liquidity
-> sweep
-> displacement candidate
-> structure effect와 FVG metadata
```

## 실제 차트에서 발견하는 방법

### 찾는 순서

1. symbol과 timeframe별 닫힌 OHLC를 시간순으로 정렬한다.
2. 각 봉의 `body`, `range`, `body_ratio`를 계산한다.
3. 현재 봉을 제외한 이전 `20`개 body의 median을 계산한다.
4. 현재 body가 baseline 대비 충분히 크고 range 안에서 body 비율이 충분히 높으면 displacement bar candidate를 만든다.
5. 종가 방향으로 `bullish | bearish`를 정한다.
6. 선행 sweep id, broken swing id, FVG 생성 참여 여부를 metadata로 연결한다.
7. 고급 구현에서는 인접한 같은 방향 봉을 impulse leg로 묶고 leg high/low를 저장한다.
8. 구조 돌파가 있다면 `structure_confirmed` tag를 추가한다.

### 유효 조건

최소 구현의 `research default`:

```text
body = abs(close - open)
range = high - low
body_ratio = body / range
baseline_body = median(previous 20 closed-bar bodies)

displacement_bar_candidate =
  body >= 1.50 * baseline_body
  and body_ratio >= 0.60
```

- 현재 봉을 baseline 계산에서 제외한다.
- `range > 0`이다.
- 이전 body history가 충분하다.
- bullish는 `close > open`, bearish는 `close < open`이다.
- body factor와 body ratio threshold를 parameters version에 기록한다.
- 구조 확인형 bearish displacement는 사전 확정 swing low 아래 종가, bullish displacement는 사전 확정 swing high 위 종가를 별도 tag로 기록한다.

### 무효 조건

- range가 `0`이다.
- baseline 계산에 현재 봉 또는 미래 봉을 포함한다.
- 결측 또는 데이터 gap 직후 봉을 정상 baseline으로 비교한다.
- wick만 길고 body 비율이 낮은 spike를 displacement로 확정한다.
- 작은 lethargic move를 수치 근거 없이 displacement라고 부른다.
- displacement candidate만으로 MSS, FVG, 진입을 자동 확정한다.
- `1.50`, `0.60`, `20`을 ICT가 직접 고정한 수치라고 설명한다.

### 대표 예시

```text
median(previous 20 bodies): 4.00
current open:               100.00
current high:               107.00
current low:                 99.50
current close:              106.50

body       = 6.50
range      = 7.50
body_ratio = 0.867
size_ratio = 6.50 / 4.00 = 1.625

1.625 >= 1.50
0.867 >= 0.60
close > open
-> bullish displacement bar candidate
```

### 단일 봉과 Displacement Leg

자막은 displacement를 단일 캔들뿐 아니라 가격 leg로도 설명한다.

- 단일 봉 후보: 빠른 움직임을 자동 탐지하기 위한 최소 프록시
- impulse leg: 연속된 방향성 이동을 묶은 고급 객체
- structure-confirming leg: liquidity raid extreme부터 swing 종가 돌파까지의 구간

Episode 6은 bearish 문맥에서 old high 위 run 이후 short-term low 아래 종가 돌파까지를 displacement range로 보고, 그 high와 low 사이에서 FVG를 찾도록 설명한다.

```text
displacement_leg = {
  direction,
  start_at,
  end_at,
  price_low,
  price_high,
  linked_sweep_id,
  broken_swing_id,
  fvg_ids[]
}
```

leg 결합 규칙은 자막에 고정된 숫자가 없다. 고급 구현의 configurable parameter로 둔다.

### 보조 설명: FVG가 있어도 Displacement가 없는 경우

Episode 20과 41은 FVG가 존재하더라도 energetic하게 멀어지는 움직임 또는 low 아래 meaningful displacement가 없으면 참여하지 않는 사례를 설명한다.

따라서:

```text
FVG existence != qualified trade setup
```

FVG 탐지 자체와 displacement confluence를 분리해 저장해야 한다.

## 실전 활용

### 진입

Displacement 자체로 추격 진입하지 않는다.

```text
SSL sweep
-> bullish displacement
-> bullish MSS
-> bullish FVG retracement
-> long 후보

BSL sweep
-> bearish displacement
-> bearish MSS
-> bearish FVG retracement
-> short 후보
```

Episode 5와 24는 displacement 이후 FVG 되돌림을 관찰하는 흐름을 설명한다.

### 손절

Displacement는 손절을 직접 정하지 않는다.

- long 후보: sweep extreme 또는 구조 swing low 아래 buffer 검토
- short 후보: sweep extreme 또는 구조 swing high 위 buffer 검토
- 실제 손절 규칙은 retracement entry model에서 확정한다.

### 목표

- bullish displacement 이후: 현재 가격 위 active BSL
- bearish displacement 이후: 현재 가격 아래 active SSL
- displacement leg의 range high/low와 linked liquidity target을 scanner에 남긴다.

### 시간대

기본 displacement 탐지는 시간대와 무관하다. 세션과 경제지표는 필터 metadata이다.

- Episode 3, 5: intraday stop hunt와 MSS 문맥
- Episode 24: 09:30 Judas swing 뒤 displacement 예시
- Episode 40: 08:30 news driver와 energetic displacement 예시

## 흔한 오해

### 초보자가 잘못 해석하는 부분

- 큰 wick 하나를 displacement라고 부른다.
- body가 큰 모든 봉을 매매 신호로 사용한다.
- 단일 봉 프록시를 ICT의 완전한 정의라고 생각한다.
- FVG가 있으면 displacement도 자동으로 있다고 가정한다.
- displacement와 MSS를 같은 사건으로 취급한다.
- 고정 threshold를 모든 symbol과 timeframe에 검증 없이 사용한다.
- displacement 봉을 본 뒤 되돌림 없이 추격 진입한다.

### ICT가 실제로 의미하는 것

- displacement는 빠르고 energetic하며 명확한 repricing이다.
- lethargic movement는 충분하지 않다.
- 구조 문맥에서는 반대 방향 short-term swing 종가 돌파가 중요하다.
- displacement leg 안에서 FVG를 찾는다.
- 수치 threshold는 자막의 고정값이 아니라 연구용 프록시이다.
- raw displacement, structure effect, FVG confluence를 별도 필드로 기록한다.

## 다른 개념과의 연결

### 관련 개념

| 연결 개념 | displacement의 역할 |
| --- | --- |
| Liquidity Sweep | repricing을 관찰할 선행 사건 |
| Swing Points | structure break reference 제공 |
| MSS | displacement 종가가 반대 swing을 이탈했는지 확인 |
| FVG | displacement leg 안 imbalance 후보 |
| Order Block | displacement와 delivery state 변화 metadata |
| Dealing Range | displacement leg high/low를 range 후보로 전달 |
| Economic Calendar Filter | news driver window metadata |

### 의존 관계

```text
closed OHLC
-> displacement bar candidate
  -> optional impulse leg
  -> optional linked sweep
  -> optional broken swing
  -> optional FVG ids

sweep + opposite displacement + broken swing close
-> MSS confluence
```

## 코드 구현 시 문제점

### 사람이 보는 기준

사람은 움직임이 이전 가격 행동보다 얼마나 빠르고 큰지, swing을 종가로 돌파했는지, FVG를 남겼는지, sweep 직후인지 함께 본다. `energetic`, `obvious`, `meaningful`은 사람에게 유용하지만 그대로는 코드 규칙이 아니다.

### 코드로 구현 가능한 부분

- body, range, body ratio 계산
- 이전 body median 대비 size ratio 계산
- bullish/bearish 방향 분류
- raw displacement bar candidate 탐지
- 사전 확정 swing 종가 돌파 tag
- sweep event와 시간 거리 계산
- FVG 참여 여부 연결
- leg high/low, duration, cumulative move 계산

### 코드로 구현 어려운 부분

- 모든 시장에 공통인 energetic movement threshold
- 복수 봉을 하나의 impulse leg로 묶는 최적 규칙
- 사람이 시각적으로 보는 obvious repricing
- news spike와 유효 displacement의 보편적 분리
- 복수 displacement 중 narrative상 기준 leg 선택

어려운 부분은 parameter version, `manual_review_reason`, 백테스트 결과로 관리한다.

## 핵심 문장

1. Displacement는 빠르고 energetic한 방향성 repricing이다.
2. Lethargic move와 wick spike를 displacement로 자동 처리하지 않는다.
3. 최소 구현은 body 크기와 body ratio를 사용하는 단일 봉 프록시이다.
4. 단일 봉 프록시 threshold는 ICT 고정값이 아니라 research default이다.
5. Baseline 계산에는 현재 봉과 미래 봉을 포함하지 않는다.
6. Bullish displacement와 bearish displacement를 분리한다.
7. 구조 확인형 displacement는 반대 방향 short-term swing 종가 돌파를 기록한다.
8. Displacement와 MSS는 연결되지만 동일한 사건은 아니다.
9. FVG는 displacement leg 내부에서 찾되 FVG 존재만으로 setup을 확정하지 않는다.
10. Displacement를 본 뒤 추격하지 않고 retracement model을 기다린다.

## 이 개념의 본질

Displacement는 sweep 이후 가격이 실제로 반대 방향으로 재평가되고 있는지를 OHLC의 상대 크기, 종가 위치, 구조 돌파, FVG metadata로 객관화하는 repricing 프록시이다.

## Transcript 근거 분류

### 에피소드 전반에 반복되는 핵심 근거

- Episode 5: old high 위 거래 후 obvious displacement, lethargic move 배제, sudden displacement 뒤 FVG 탐색
- Episode 6: quick shift, energetic move, swing 종가 돌파, displacement high/low와 내부 FVG
- Episode 20: FVG에서 충분히 멀어지는 energetic movement 확인
- Episode 22: high 회수와 low break 뒤 energetic displacement 확인
- Episode 24: sell stops 회수 뒤 displacement가 없으면 long model 배제, sweep 뒤 displacement leg 내부 FVG 탐색
- Episode 40: BSL taken 뒤 energetic displacement와 08:30 news driver
- Episode 41: FVG가 있어도 low 아래 meaningful displacement가 없으면 참여 배제

### 특정 구간에 집중된 보조 근거

- Episode 19: consolidation 뒤 displacement가 range를 제공하는 설명
- Episode 24: 09:30 Judas swing 뒤 displacement와 FVG long 예시
- Episode 38: energetic price run 뒤 premium/discount measurement 예시

### 검색 분포

정리본 `transcripts/*.en.txt` 전체 검색 기준으로 `displacement`는 23개 에피소드 파일, `energetic`은 15개, `displacement leg`는 3개에 등장한다. 수치는 자막 중복을 포함하므로 반복 분포 확인용이다.
