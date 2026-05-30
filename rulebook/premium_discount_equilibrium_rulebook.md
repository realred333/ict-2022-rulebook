# Premium / Discount / Equilibrium Rulebook

공통 계약: [README.md](README.md)

## 객관적 판별 규칙

### 필수 조건
- `equilibrium = (range_high + range_low) / 2`
- `price > equilibrium`이면 premium, `price < equilibrium`이면 discount, 같으면 equilibrium이다.
- 활성 `range_id`를 함께 저장한다.

### 무효 조건
- dealing range가 없거나 `range_high <= range_low`이다.
- 계산 시점 이후 정보로 range를 선택했다.

### 필터 조건
- bullish 되돌림 후보는 discount overlap, bearish 되돌림 후보는 premium overlap을 metadata로 기록한다.

### 체크리스트
- [ ] range id가 있는가?
- [ ] midpoint가 재현 가능한가?
- [ ] normalized position을 저장했는가?

## 코드 구현 규칙

### 입력 데이터
dealing range, 관찰 가격

### 필요 조건
유효한 range가 필요하다.

### 의사코드
```text
eq = (high + low) / 2
position = (price - low) / (high - low)
zone = premium if position > 0.5 else discount if position < 0.5 else equilibrium
```

### False Signal 필터
range 선택이 바뀌면 기존 계산을 새 `range_id`로 다시 산출한다.

### 주관적 요소
활성 range 선택.

### 객관적 요소
midpoint, normalized position, zone.

### 최소 구현 버전
활성 range 하나의 50% 구분.

### 고급 구현 버전
HTF/LTF range별 zone overlap.

