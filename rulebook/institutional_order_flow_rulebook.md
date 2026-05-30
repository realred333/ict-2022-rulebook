# Institutional Order Flow Rulebook

공통 계약: [README.md](README.md)

## 객관적 판별 규칙

### 필수 조건
- 출력은 `aligned_bullish | aligned_bearish | mixed | manual_review`이다.
- proxy 입력은 HTF MSS, HTF FVG/OB, active liquidity direction이다.
- 각 입력 근거 id를 저장한다.

### 무효 조건
- OHLC proxy를 실제 기관 주문 관측값이라고 표시한다.
- HTF 데이터 없이 aligned 상태를 반환한다.

### 필터 조건
- bullish aligned: HTF bullish MSS, bullish FVG 또는 OB support, 상단 draw on liquidity가 정렬된다.
- bearish aligned: 위 조건을 반대로 적용한다.
- 충돌하면 mixed이다.

### 체크리스트
- [ ] 출력에 `proxy`임을 표시했는가?
- [ ] HTF 근거 id가 있는가?
- [ ] mixed 상태를 허용하는가?

## 코드 구현 규칙

### 입력 데이터
HTF OHLC, MSS, FVG, OB, liquidity registry

### 필요 조건
HTF 이벤트 탐지기가 필요하다.

### 의사코드
```text
bull = bullish_mss and bullish_pd_array_support and upside_liquidity
bear = bearish_mss and bearish_pd_array_resistance and downside_liquidity
state = aligned_bullish if bull and not bear else aligned_bearish if bear and not bull else mixed
```

### False Signal 필터
단일 FVG 또는 단일 OB만으로 aligned 상태를 만들지 않는다.

### 주관적 요소
HTF 선택과 draw on liquidity 우선순위.

### 객관적 요소
proxy 입력의 존재 여부와 정렬 상태.

### 최소 구현 버전
HTF MSS, FVG/OB, liquidity 3요소 정렬.

### 고급 구현 버전
다중 HTF, 상태 유효 기간, daily bias confidence metadata.

