# Transcript Source Index

## 목적

개념과 Rulebook을 점진적으로 개선할 때 다시 확인할 자막 위치를 기록한다. 줄 번호는 현재 `transcripts/*.en.txt` 정리본 기준이다. 자막 중복 라인이 있으므로 인접 줄을 함께 읽고, 표현이 불명확하면 대응 `*-orig.txt`를 대조한다.

## 코퍼스 확인

- 정리본: 40개, 약 760,333 words
- 원본 대조 파일: 각 정리본에 대응하는 `*-orig.txt`
- 포함 Episode: 01~27, 29~41
- 현재 누락: Episode 28

## 대표 근거

| 개념 | 대표 자막 위치 | 확인할 내용 |
| --- | --- | --- |
| Swing Points | Episode 5, lines 1347~1383 | swing high/low 3캔들 구조와 BSL/SSL 연결 |
| Swing Points | Episode 10, lines 1823~1838 | swing low는 좌우 higher low를 가진 3캔들 구조 |
| Swing Points | Episode 12, lines 1631~1650 | intermediate-term high 계층의 보조 규칙 |
| Swing Points | Episode 40, lines 555~570 | swing high 3캔들 구조와 fractal 구분 |
| Liquidity, BSL, SSL | Episode 3, lines 75~80 | MSS와 buy-side/sell-side liquidity 연결 |
| Liquidity Sweep | Episode 3, lines 4248~4254 | stop hunt 이후 MSS 관찰 |
| MSS | Episode 3, lines 1587~1615 | bullish swing high traversal은 필요하지만 level 위 종가 마감은 필수가 아님 |
| MSS | Episode 3, lines 2091~2123 | buy stops taken 뒤 short-term low 아래 거래와 bearish shift |
| MSS, Displacement | Episode 6, lines 2723~2755 | 강화된 bullish profile은 short-term high 위 종가와 energetic displacement를 확인 |
| Swing Points, MSS | Episode 7, lines 2840~2847 | swing high break와 bullish shift |
| MSS | Episode 12, lines 3528~3536 | short-term low break 이후 imbalance 되돌림 |
| Displacement, FVG | Episode 5, lines 3532~3540 | sudden displacement 이후 FVG와 retracement |
| FVG | Episode 6, lines 1319~1423 | bearish FVG의 3캔들 formation과 wick 비중첩 구간 |
| FVG | Episode 6, lines 1931~1967 | bearish FVG의 방향별 zone 경계 |
| FVG | Episode 6, lines 2596~2647 | bullish FVG, institutional order flow, 3캔들 formation과 zone 경계 |
| FVG 필터 | Episode 6, lines 1719~1763, 2451~2531 | liquidity run과 displacement 문맥 안에서 FVG를 선택 |
| FVG 필터 | Episode 41, lines 1391~1419 | FVG가 있어도 meaningful displacement가 없으면 참여 배제 |
| OB | Episode 3, lines 1731~1779 | 복수 down-close candle을 하나의 continuous OB로 보고 시작 open을 연장 |
| OB | Episode 3, lines 2871~3003 | series 시작 open 반대 방향 관통과 state of delivery 변화 |
| OB 오해 | Episode 3, lines 3511~3586 | 모든 반대색 candle을 자동 OB로 보지 않으며 복수 candle이 하나의 OB가 될 수 있음 |
| OB 오해 | Episode 12, lines 4103~4199 | 마지막 반대색 봉 하나로 축약하지 않고 consecutive series와 내부 imbalance를 봄 |
| OB refinement | Episode 17, lines 1751~1787 | 긴 wick보다 body retracement를 선호하는 사례 |
| Breaker Block | Episode 39, lines 2395~2499 | BSL 제거, short-term low 아래 거래, bearish narrative, lowest down-close source candle |
| Breaker Block refinement | Episode 10, lines 4067~4187 | bearish breaker 패턴과 source candle lower half refinement |
| Breaker Block 오해 | Episode 5, lines 3531~3567; Episode 25, lines 6083~6111 | 단순 모델에서 breaker가 필수 신호는 아님 |
| OTE | Episode 19, lines 5931~5967; Episode 38, lines 5191~5203 | OTE의 `62~79` retracement 구간과 bullish low-high 측정 사례 |
| OTE refinement | Episode 22, lines 5271~5299, 5411~5514 | OTE level `62`, `79`와 swing 내부 lowest/highest open-close anchor |
| OTE transcript conflict | Episode 41, lines 2179~2191 | `62 to 70` 전사 표현은 대응 `-orig`에도 남아 있어 영상 수동 확인 필요 |
| OTE 오해 | Episode 1, lines 2811~2838; Episode 7, lines 2615~2638 | 정밀 OTE 또는 Fibonacci가 모든 단순 모델의 필수 조건은 아님 |
| Killzone | Episode 8, lines 564~572 | New York killzone과 time of day |
| Power of Three | Episode 9, lines 19~27 | accumulation, manipulation, distribution |
| Power of Three | Episode 10, lines 1243~1251 | 세 구성요소 재확인 |
| Opening Price | Episode 38, lines 2023~2027 | midnight opening price |
| Opening Price | Episode 39, lines 5120~5175 | New York local midnight open과 bearish day short 문맥 |
| Daily Bias | Episode 7, lines 5543~5548 | 매일 유리한 bias가 아님 |
| Daily Bias | Episode 40, lines 2283~2435 | 매일 predetermined bias를 강제하지 않는 단순화 규칙 |
| SMT Divergence | Episode 38, lines 5664~5673 | lower low와 higher low 비교 |
| Economic Calendar | Episode 10, lines 19~30 | economic calendar events와 open |
| Economic Calendar | Episode 40, lines 3012~3020 | killzone과 high/medium impact 이벤트 결합 |

## 반복 용어 분포

아래 수치는 자막 중복 라인을 제거하지 않은 검색 결과다. 절대 빈도가 아니라 문서화 우선순위를 판단하는 보조 지표이다.

| 용어 | 검색 횟수 | 등장 Episode 수 |
| --- | ---: | ---: |
| liquidity | 970 | 37 |
| fair value gap | 827 | 39 |
| order block | 593 | 28 |
| imbalance | 633 | 32 |
| opening price | 449 | 16 |
| discount | 393 | 26 |
| displacement | 255 | 23 |
| premium | 257 | 21 |
| swing high | 234 | 21 |
| swing low | 233 | 27 |
| relative equal | 443 | 33 |
| equilibrium | 120 | 18 |
| daily bias | 88 | 10 |
| market structure shift | 61 | 7 |
| shift in market structure | 66 | 13 |
| killzone | 58 | 6 |
| SMT | 54 | 6 |
| institutional order flow | 30 | 5 |
| breaker | 66 | 8 |
| optimal trade entry | 60 | 11 |

## 다음 개선 시 기록할 것

- 영상 차트에서 확인한 symbol, trading date, timeframe, 가격 좌표
- Rulebook `research default`를 바꾼 백테스트 근거
- false positive와 false negative 사례
- 자막 설명과 구현 proxy가 다른 지점
