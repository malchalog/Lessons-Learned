# Lessons Learned — 코드 리뷰 & 어드민 커맨드 작업

> 작성일: 2026-03-12
> 작업 브랜치: `feat/add-update-coupon-kinds-admin-command`

---

## 1. 코드 리뷰 코멘트를 어떻게 읽어야 하나

### 리뷰어별 역할 구분
| 리뷰어 | 역할 | 중요도 |
|--------|------|--------|
| Claude bot | 버그, 컨벤션 자동 탐지 | 높음 |
| Cursor bot | 특정 버그 자동 수정 제안 | 중간 |
| Andy (팀원) | 팀 컨벤션, 설계 방향 | 매우 높음 |
| SonarQube | 코드 품질 (중복 등) | 참고용 |

### 핵심 교훈
- 리뷰 코멘트는 **하나씩 이해하고 수정**해야 한다. 이해 없이 수정하면 부분적으로만 고쳐지거나 놓치는 부분이 생긴다.
- `@learn` 태그가 붙은 코멘트는 팀 전체에 저장되는 **학습 포인트**다.

---

## 2. `?? undefined` 패턴

### 언제 필요한가
- Yup 스키마에서 `.optional()` 또는 `.nullable()`로 선언된 필드
- DB에 저장할 때 `null`이 실수로 저장되는 걸 방지하고 싶을 때

### 개념
```ts
// null → undefined → DB에 저장 안 됨 (기존 값 유지)
draw_price: drawPrice ?? undefined

// null → null → DB에 null이 저장됨 (의도치 않은 덮어쓰기)
draw_price: drawPrice
```

### 예외: `applyPatch`
`applyPatch`는 `null`을 **"필드 삭제"**로 처리하도록 설계됐다. update 커맨드에서는 `?? undefined`를 쓰면 안 된다.
```ts
if (value === undefined) return;         // 건드리지 않음
if (value === null) $unset (필드 삭제)   // 의도적 삭제
else $set (값 저장)                      // 값 설정
```

---

## 3. Non-null Assertion (`!`) 을 쓰면 안 되는 이유

### 개념
```ts
items!  // "이거 null 아니야, 믿어" → TypeScript 안전망 강제 해제
```

### 문제
- 개발자의 "믿음"이 틀리면 런타임에 앱이 터진다
- 팀 lint 규칙 (`no-non-null-assertion`)으로 막혀있다

### 해결: `Yup.InferType`
```ts
const { items } = argsSchema.validateSync(args, {...})
  as Yup.InferType<typeof argsSchema>;
// TypeScript가 Yup 스키마 기준으로 타입을 정확히 알게 됨
// → items! 없이도 안전하게 접근 가능
```

### TypeScript vs Yup 역할 구분
| | TypeScript | Yup |
|--|------------|-----|
| 언제? | 코드 작성 시 (실행 전) | 앱 실행 중 (실시간) |
| 검사 대상 | 개발자 코드 구조 | 외부 입력 데이터 |
| 비유 | 맞춤법 검사기 | 공항 입국 심사관 |

---

## 4. `Promise.all` vs `Promise.allSettled` vs `promiseAllPatiently`

### `Promise.all`
- 하나가 실패하면 **즉시 중단**, 나머지 실행 안 됨
- 200개 중 1개 실패 → 전체 실패

### `promiseAllPatiently`
```ts
// 내부 구현
const results = await Promise.allSettled(promises);
results.map(r => {
  if (r.status === "rejected") throw r.reason; // 결국 에러를 던짐
});
```
- 모든 항목 끝까지 실행하지만, 실패하면 **여전히 에러를 던짐**
- 개별 실패 정보 수집 불가능
- **DB 트랜잭션과 함께 사용 금지**

### `Promise.allSettled` (직접 사용)
- 모든 항목 끝까지 실행
- 성공/실패를 **각각 수집** 가능
- 부분 성공 처리 + 실패 로깅이 필요할 때 적합

### 언제 무엇을 쓸까
| 상황 | 선택 |
|------|------|
| 하나 실패 시 전체 취소 | `Promise.all` |
| 전부 실행 후 에러 던지기 (어떤 항목이 실패했는지 몰라도 됨) | `promiseAllPatiently` |
| 전부 실행 후 어떤 항목이 실패했는지 수집해서 계속 진행 | `Promise.allSettled` 직접 |

---

## 5. 트랜잭션과 부분 실패

### 트랜잭션이란
- "전부 성공" 또는 "전부 실패" — 중간 상태 없음
- `.save({ session })` 으로 트랜잭션에 참여

### 부분 실패를 허용하려면
- 트랜잭션을 **포기**해야 한다
- `.save()` (session 없음) → 각 저장이 즉시 개별 커밋됨
- 실패해도 나머지는 계속 진행 가능

### 선택 기준
| 상황 | 선택 |
|------|------|
| "전부 아니면 전부" | 트랜잭션 유지 |
| 부분 성공 허용 + 실패 알림 | 트랜잭션 포기 + 실패 슬랙 |

---

## 6. Bulk 커맨드 설계 원칙

### 단건 → Bulk 변환 시 반드시 확인할 것

1. **스키마 동기화**: 단건에 추가된 필드가 bulk에도 있는가
2. **DB 저장 동기화**: `generateDoc` 호출에 모든 필드가 전달되는가
3. **실패 처리**: 부분 실패 시 어떻게 처리할 것인가
4. **인덱스 추적**: 실패 항목의 원본 인덱스를 정확히 알 수 있는가

### 입력 구조 설계
```ts
// 모든 쿠폰에 같은 값 적용
{ kindIds: ["a", "b", "c"], marketPrice: 5000 }

// 각 쿠폰마다 다른 값 적용
{ items: [
  { kindId: "a", marketPrice: 5000 },
  { kindId: "b", hiddenAt: "2025-11-30..." },
]}
```
→ "updateCouponKind를 여러 개 한꺼번에 처리"라는 말은 **후자**를 의미한다.

---

## 7. DB 필드 네이밍 컨벤션

### 규칙
- DB 필드명: **snake_case** (`vendor_price`, `standard_price`)
- JS 변수명: **camelCase** (`vendorPrice`, `standardPrice`)

```ts
// 잘못된 예
vendorPrice: vendorPrice ?? undefined  // camelCase → DB에 잘못 저장

// 올바른 예
vendor_price: vendorPrice ?? undefined  // snake_case → DB 컨벤션 준수
```

### `applyPatch`도 동일
```ts
applyPatch(updateQuery, "vendor_price", vendorPrice);  // ✅
applyPatch(updateQuery, "vendorPrice", vendorPrice);   // ❌
```

---

## 8. 실패 로깅 설계

### 실패 알림에 포함할 정보
- 어떤 항목인지 특정할 수 있는 **유니크한 식별자**
- `sourceId`처럼 유니크하지 않은 값만으로는 부족하다
- 조합으로 보완: `index` + `name` + `sourceId` + `marketPrice` + `countryCode`

### `originalIndex` 추적
```ts
// 번역 실패가 있으면 itemsWithTranslations의 인덱스 ≠ 원본 인덱스
// 처음부터 originalIndex를 함께 담아야 한다
items.map(async (item, originalIndex) => ({
  item,
  originalIndex,
  translated: await translateCouponFields(...)
}))
```

---

## 9. 요구사항 전달 방법

### 이번에 발생한 오해
- **말한 것**: "updateCouponKind를 한꺼번에 여러 개 처리하는 커맨드"
- **AI가 이해한 것**: "여러 kindId에 동일한 값 적용"
- **실제 의도**: "각 쿠폰마다 다른 값을 적용하는 bulk 버전"

### 오해를 줄이는 요구사항 전달법
```
❌ "bulk로 처리하는 커맨드 만들어줘"

✅ "createCouponKinds가 createCouponKind의 bulk 버전이듯,
   updateCouponKinds도 updateCouponKind의 bulk 버전이야.
   입력 구조는 items: [{ kindId, ...fields }] 형태로."
```

→ **입력 구조를 명시적으로** 보여주거나, 기존 패턴을 참조하면 오해가 줄어든다.

---

## 10. 비판적 사고 습관

### AI 코드 리뷰 시 스스로 물어볼 것
1. "이 수정이 정말 완전한가? 빠진 곳은 없는가?"
2. "단건과 bulk 커맨드가 동기화돼 있는가?"
3. "이 인덱스가 원본 배열 기준인가, 필터된 배열 기준인가?"
4. "트랜잭션을 포기했을 때 생기는 부작용은 없는가?"
5. "AI가 제시한 방향이 팀 컨벤션과 맞는가?"

### 교훈
- AI가 "범위 밖"이라고 판단한 것도 실제로는 수정 대상일 수 있다
- 코드를 설명할 때 틀린 내용이 있으면 **바로 지적**해야 반복 실수를 막는다
