# Lessons Learned: 카카오 비즈 잔액 알림 개선 PR

**PR**: [TRACER-bf/tracer#17150](https://github.com/TRACER-bf/tracer/pull/17150)
**날짜**: 2026-03-20
**작업 내용**: `#mw-giftshop-alert` 슬랙 알림에서 카카오 비즈 @멘션 및 `:warning:` 문구 제거

---

## 이전 PR 대비 개선한 점

### 1. Plan 모드로 계획 먼저 수립
- 작업 시작 전 구현 방향을 정리하고 컨펌 받은 뒤 코딩 진행
- 방향이 잘못된 채로 진행해서 나중에 되돌리는 낭비를 줄임
- Plan 모드에서 작업 요구사항을 Claude에게 전달하고, 실행에 최적화된 프롬프트를 먼저 설계하도록 요청함

### 2. Git 워크플로우 체계화 → `/tracer` 커맨드로 자동화
- 매번 반복하던 요구사항(main 최신화 → 새 브랜치 → 작업별 커밋 → push)을 `/tracer` 커맨드에 반영
- 브랜치명·커밋 메시지는 프로젝트 기존 컨벤션을 자동으로 참고하도록 설정
- 다음 PR부터는 같은 지시를 반복하지 않아도 됨

### 3. 리뷰 피드백을 `/tracer` 커맨드 룰로 즉시 반영
- 리뷰 받은 내용을 다음 작업에서 재발하지 않도록 커맨드에 체크리스트 추가

---

## 리뷰에서 지적받은 사항 & 재발 방지 룰

### 지적 1: 불필요한 중간 변수
```ts
// Before (지적받은 코드)
const needChargeXoxo = xoxoBalance < BALANCE_THRESHOLDS.XOXO;
const needCharge = needChargeXoxo; // 그대로 복사만 하는 변수

// After
const needCharge = xoxoBalance < BALANCE_THRESHOLDS.XOXO;
```
**룰**: 변수를 만들 때 "이 변수가 바로 다른 변수에 대입만 되지는 않는가?" 확인

### 지적 2: Dead code (미사용 상수) 방치
```ts
// Before (지적받은 코드)
const BALANCE_THRESHOLDS = {
  GIFTISHOW: 1400000,
  XOXO: 6000,
  KAKAO_BIZ: 65000000, // needChargeKakaoBiz 제거 후 아무도 참조하지 않음
} as const;

// After
const BALANCE_THRESHOLDS = {
  GIFTISHOW: 1400000,
  XOXO: 6000,
} as const;
```
**룰**: 기능 제거 시 해당 코드를 참조하는 미사용 상수, 변수, import도 함께 정리

---

## `/tracer` 커맨드에 추가된 체크리스트

```
- 기능 제거 시 해당 코드를 참조하는 미사용 상수, 변수, import도 함께 정리한다
- 불필요한 중간 변수(바로 다른 변수에 대입만 하는 변수)가 없는지 확인한다
```
