# ICT 2022 Mentorship Research Repository

## 이 저장소는 무엇인가

이 저장소는 ICT(The Inner Circle Trader) 2022 Mentorship 자막을 연구 자료로 사용해, ICT 개념을 코드로 구현 가능한 규칙으로 변환하는 프로젝트이다.

목표는 영상 내용을 짧게 요약하는 것이 아니다. 최종 목표는 다음 파이프라인을 구축하는 것이다.

```text
transcripts
-> concepts
-> rulebook
-> pine
-> scanner
```

- `transcripts`: ICT가 실제로 설명한 내용을 확인하는 원문 자료
- `concepts`: 사람이 개념을 이해하고 연결 관계를 파악하기 위한 문서
- `rulebook`: Pine Script와 Python Scanner가 같은 방식으로 판정하기 위한 규칙 계약
- `pine`: TradingView 차트에서 규칙을 시각적으로 검증할 Pine Script
- `scanner`: 여러 종목과 timeframe을 자동으로 탐색할 Python Scanner

현재는 `transcripts -> concepts -> rulebook` 단계가 진행 중이다. `pine`과 `scanner`는 아직 생성하지 않았다.

## Concepts는 무엇인가

`concepts/`는 ICT 개념 위키이다. 사람이 읽는 문서다.

예를 들어 [swing_points.md](concepts/swing_points.md)는 다음 질문에 답한다.

- Swing high와 swing low는 무엇인가?
- ICT는 일반적인 pivot과 무엇을 다르게 보는가?
- 차트에서 어떤 순서로 찾는가?
- liquidity, MSS, dealing range와 어떻게 연결되는가?
- 사람이 차트를 볼 때 판단하는 부분과 코드가 자동화할 수 있는 부분은 무엇인가?

Concept 문서는 단순 정의만 기록하지 않는다. 실제 운영 흐름 안에서 개념의 역할을 설명한다.

```text
Swing Points
-> Liquidity
-> BSL / SSL
-> Liquidity Sweep
-> Displacement
-> MSS
-> FVG
-> Order Block
```

Concept 문서의 주요 목적은 다음과 같다.

1. 자막 전반에서 반복되는 설명을 추출한다.
2. 특정 Episode에만 등장하는 설명은 보조 설명으로 분리한다.
3. 다른 개념과의 선행·후행 관계를 기록한다.
4. 코드 구현이 쉬운 부분과 어려운 부분을 분리한다.
5. 사람이 읽어도 이해할 수 있는 언어로 규칙의 배경을 설명한다.

즉, `concepts/`는 **왜 이 규칙이 필요한지 설명하는 문서**이다.

## Rulebook은 무엇인가

`rulebook/`은 코드 구현 명세이다. 사람이 읽을 수 있지만, 최종 독자는 Pine Script와 Python Scanner 구현자다.

예를 들어 [swing_points_rulebook.md](rulebook/swing_points_rulebook.md)는 short-term swing을 다음처럼 정의한다.

```text
short_term_swing_high(i) =
  high[i] > high[i - 1]
  and high[i] > high[i + 1]

short_term_swing_low(i) =
  low[i] < low[i - 1]
  and low[i] < low[i + 1]
```

이 규칙은 코드로 그대로 옮길 수 있다.

```text
is_sth = high[1] > high[2] and high[1] > high[0]
is_stl = low[1] < low[2] and low[1] < low[0]
```

Rulebook에는 다음 내용이 들어간다.

- 목적
- 객관적 정의
- 필수 조건
- 무효 조건
- 예외 조건
- 상태 변화(State Machine)
- 입력 데이터와 필요 변수
- 조건식과 의사코드
- OHLC만 사용하는 최소 구현
- HTF, 세션, SMT, 경제지표를 포함하는 고급 구현
- False Signal 제거 방법
- Pine Script 구현 가이드
- Python Scanner 구현 가이드
- 백테스트 시 주의사항

즉, `rulebook/`은 **코드가 무엇을 어떤 조건으로 판정해야 하는지 정의하는 문서**이다.

## Concepts와 Rulebook의 차이

| 구분 | `concepts/` | `rulebook/` |
| --- | --- | --- |
| 목적 | 개념 이해 | 코드 구현 계약 |
| 주요 독자 | 연구자, 트레이더 | Pine/Python 구현자 |
| 핵심 질문 | 왜 필요한가? 무엇과 연결되는가? | 정확히 언제 `true`인가? 언제 무효인가? |
| 표현 방식 | 설명, 연결 관계, 차트 관찰 순서 | 수식, 상태, 변수, 의사코드 |
| 모호한 판단 | 사람이 보는 기준으로 명시 | `manual_review` 상태로 분리 |
| 예시 | 차트 해석 흐름 | OHLC 조건식 |

같은 내용을 두 번 쓰는 것이 아니다.

Concept는 의미를 보존하고, Rulebook은 의미를 잃지 않는 범위에서 코드를 위한 판정 규칙으로 좁힌다.

## 왜 두 단계로 나누는가

ICT 설명에는 사람이 차트를 보면서 판단하는 부분이 있다. 이를 바로 코드로 옮기면 다음 문제가 생긴다.

- 같은 단어를 구현자마다 다르게 해석한다.
- 백테스트 결과를 재현할 수 없다.
- 미래 봉을 미리 사용하는 look-ahead bias가 들어간다.
- `강한`, `명확한`, `이상적인` 같은 표현이 조건식으로 바뀌지 않는다.
- 한 개념의 규칙이 다른 개념의 규칙과 충돌한다.

따라서 다음 순서를 지킨다.

```text
원문 확인
-> 반복 설명 추출
-> 개념 의미 정리
-> 객관적 조건과 주관적 조건 분리
-> Rulebook 작성
-> Pine Script 시각 검증
-> Python Scanner와 백테스트
```

## 폴더 구조

```text
ICT/
├─ README.md
├─ ICT_LEARNING_MAP.md
├─ TRANSCRIPT_SOURCE_INDEX.md
├─ raw/
├─ transcripts/
├─ concepts/
│  ├─ README.md
│  ├─ swing_points.md
│  ├─ liquidity.md
│  └─ ...
└─ rulebook/
   ├─ README.md
   ├─ swing_points_rulebook.md
   ├─ liquidity_rulebook.md
   └─ ...
```

### `raw/`

원본 `.vtt` 자막 파일이다.

### `transcripts/`

검색과 연구에 사용하는 `.txt` 자막 파일이다.

- `*.en.txt`: 정리본
- `*-orig.txt`: 표현이 불명확할 때 대조하는 원본

현재 정리본은 40개이며 Episode 28 파일은 없다. 정리본에는 동일 문장이 반복되는 구간이 있으므로 단순 검색 횟수를 절대 빈도로 해석하지 않는다.

### `ICT_LEARNING_MAP.md`

[ICT_LEARNING_MAP.md](ICT_LEARNING_MAP.md)는 전체 설계도다.

- 핵심 개념 목록
- 개념 의존 관계
- 초보자 학습 순서
- Pine/Python 구현 순서
- 객관화 원칙

새 규칙을 추가할 때는 이 문서의 의존 관계와 충돌하지 않는지 확인한다.

### `TRANSCRIPT_SOURCE_INDEX.md`

[TRANSCRIPT_SOURCE_INDEX.md](TRANSCRIPT_SOURCE_INDEX.md)는 대표 자막 근거 색인이다.

- 어떤 Episode를 다시 확인해야 하는지
- 어느 줄에 대표 설명이 있는지
- 어떤 개념이 여러 Episode에 반복되는지
- 어떤 설명이 특정 Episode에 집중되는지

를 기록한다.

### `concepts/`

개념 위키다. 파일명은 snake_case를 사용한다.

```text
concepts/swing_points.md
concepts/fair_value_gap.md
concepts/order_block.md
```

### `rulebook/`

개념별 코드 구현 계약이다. 대응 concept 이름 뒤에 `_rulebook`을 붙인다.

```text
rulebook/swing_points_rulebook.md
rulebook/fair_value_gap_rulebook.md
rulebook/order_block_rulebook.md
```

## 문서 상태 표기

문서에서는 다음 상태를 구분한다.

| 상태 | 의미 |
| --- | --- |
| `objective` | OHLC 또는 외부 구조화 데이터로 계산 가능 |
| `research default` | 자막에 고정 수치가 없어 백테스트용 초기값을 둔 상태 |
| `manual_review` | 사람이 chart context를 검토해야 하는 상태 |

Rulebook의 event 상태 머신은 개념별로 다음 상태를 사용한다.

| 상태 | 의미 |
| --- | --- |
| `candidate` | 일부 조건을 만족했지만 확인 조건이 아직 부족함 |
| `confirmed` | 필수 조건을 모두 만족함 |
| `invalidated` | 필수 조건 또는 유지 조건을 위반함 |
| `expired` | 제한 시간 또는 제한 봉 수 안에 확인되지 않음 |
| `manual_review` | 자동 판정만으로 결론을 내릴 수 없음 |

zone 개념은 `confirmed` 이후 `mitigated` 상태를 추가할 수 있다.

## 문서 작성 원칙

### 설명보다 규칙을 우선한다

개념 설명은 필요하지만, 실제 판별 조건까지 내려가야 한다.

잘못된 예:

```text
가격이 중요한 swing low를 깼다.
```

개선된 예:

```text
active SSL source로 등록된 confirmed swing low 아래로 low가 내려갔다.
이후 reclaim 조건에 따라 sweep 또는 breakout으로 분류한다.
```

### 규칙보다 코드화를 우선한다

조건은 가능한 한 OHLC 식으로 표현한다.

잘못된 예:

```text
강한 캔들이 발생했다.
```

개선된 예:

```text
body >= 1.50 * median(previous 20 bodies)
and body / (high - low) >= 0.60
```

### 모호한 표현을 금지한다

다음 표현은 판별 규칙으로 사용하지 않는다.

```text
강한 캔들
좋은 OB
좋은 FVG
의미 있는 유동성
명확한 움직임
```

수치화할 수 없으면 삭제하지 않고 `manual_review`로 분리한다.

### Look-Ahead Bias를 금지한다

미래 봉을 사용해 과거 시점에 이미 신호를 알았다고 가정하지 않는다.

예를 들어 short-term swing은 오른쪽 확인 봉이 닫힌 뒤에만 확정한다.

```text
pivot_at     = 중심 봉 시각
confirmed_at = 오른쪽 확인 봉 close 시각
```

백테스트 진입은 최소한 `confirmed_at` 이후에만 허용한다.

## 현재 진행 상태

문서 파일은 전체 개념에 대해 초안이 존재한다. 다만 새 형식에 맞춘 심층 검증은 중요도 순서대로 진행한다.

| 우선순위 | 개념 | 상태 |
| ---: | --- | --- |
| 1 | [Swing Points](concepts/swing_points.md) | concepts + rulebook 심층 검증 완료 |
| 2 | [Liquidity](concepts/liquidity.md) | concepts + rulebook 심층 검증 완료 |
| 3 | [BSL / SSL](concepts/bsl_ssl.md) | concepts + rulebook 심층 검증 완료 |
| 4 | [Liquidity Sweep](concepts/liquidity_sweep.md) | concepts + rulebook 심층 검증 완료 |
| 5 | [Displacement](concepts/displacement.md) | concepts + rulebook 심층 검증 완료 |
| 6 | [Market Structure Shift](concepts/market_structure_shift.md) | concepts + rulebook 심층 검증 완료 |
| 7 | [Fair Value Gap](concepts/fair_value_gap.md) | concepts + rulebook 심층 검증 완료 |
| 8 | [Order Block](concepts/order_block.md) | concepts + rulebook 심층 검증 완료 |
| 9 | [Breaker Block](concepts/breaker_block.md) | concepts + rulebook 심층 검증 완료 |
| 10 | [OTE](concepts/ote.md) | concepts + rulebook 심층 검증 완료 |

`prompt.txt`에 명시된 우선순위 1~10의 concepts와 Rulebook 심층 검증을 완료했다. 나머지 concept와 Rulebook도 초안이 존재하며, 후속 우선순위를 정한 뒤 순서대로 재검증한다.

## 처음 읽는 순서

1. 이 `README.md`를 읽는다.
2. [ICT_LEARNING_MAP.md](ICT_LEARNING_MAP.md)에서 전체 의존 관계를 본다.
3. [concepts/swing_points.md](concepts/swing_points.md)에서 첫 번째 개념을 읽는다.
4. [rulebook/swing_points_rulebook.md](rulebook/swing_points_rulebook.md)에서 같은 개념이 코드 규칙으로 바뀌는 과정을 확인한다.
5. [TRANSCRIPT_SOURCE_INDEX.md](TRANSCRIPT_SOURCE_INDEX.md)에서 원문 근거를 확인한다.
6. 이후 현재 진행 상태 표의 다음 개선 대상부터 순서대로 진행한다.

## 최종 산출물의 형태

완성된 프로젝트는 다음 역할 분리를 가진다.

```text
transcripts = 원문 근거
concepts    = 사람이 이해하는 ICT Wiki
rulebook    = 코드가 따르는 판정 계약
pine        = 차트 위 시각 검증 도구
scanner     = 자동 탐색, 저장, 백테스트 도구
```

핵심 목표는 ICT를 설명하는 문서가 아니라, 원문 근거를 추적할 수 있고 코드 결과를 재현할 수 있는 ICT 운영체제를 만드는 것이다.
