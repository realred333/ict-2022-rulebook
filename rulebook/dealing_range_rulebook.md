# Dealing Range Rulebook

공통 계약: [README.md](README.md)

## 객관적 판별 규칙

### 필수 조건
- range는 확정 swing low와 이후 swing high 또는 확정 swing high와 이후 swing low로 구성한다.
- `range_high > range_low`여야 한다.
- `range_id`, timeframe, 양 끝 pivot id를 저장한다.

### 무효 조건
- 한쪽 pivot이 미확정이다.
- high와 low가 같아 정규 위치를 계산할 수 없다.

### 필터 조건
- 최소 구현은 최근 완료 leg를 활성 후보로 둔다.
- MSS 이후 displacement leg는 별도 우선 후보 tag를 부여한다.

### 체크리스트
- [ ] 양 끝 pivot이 확정됐는가?
- [ ] range 선택 이유가 저장됐는가?
- [ ] HTF와 LTF range가 구분됐는가?

## 코드 구현 규칙

### 입력 데이터
확정 swing 목록, timeframe, 선택 정책

### 필요 조건
Swing Points Rulebook 출력이 필요하다.

### 의사코드
```text
pair alternating confirmed swings
emit range(low, high, pivot_ids, selection_reason)
mark latest completed leg as minimal active range
```

### False Signal 필터
미확정 pivot과 지나치게 작은 zero-like range를 제외한다.

### 주관적 요소
복수 range 중 narrative에 사용할 활성 range 선택.

### 객관적 요소
후보 range 생성, midpoint, normalized position.

### 최소 구현 버전
최근 완료 swing leg 한 개.

### 고급 구현 버전
HTF/LTF 중첩 range와 MSS 이후 impulse leg 우선순위.

