# Lessons Learned: GDPR 유저 삭제 커맨드 구현

**PR**: feat/gdpr-delete-user-command
**날짜**: 2026-03-25
**작업 내용**: `gdprDeleteUser` admin command — Firebase 삭제, User 익명화, 관련 컬렉션 하드 삭제, 건강/쿠폰 PII 처리

---

## 리뷰에서 지적받은 사항 & 재발 방지 룰

### 지적 1: `Promise.all` → `promiseAllPatiently` 전환이 트랜잭션에서 잘못됨

```ts
// ❌ 잘못된 수정 (트랜잭션 내부에서 promiseAllPatiently 사용)
await promiseAllPatiently([
  StepModel.deleteMany({ user_id }).session(session),
  ...
]);

// ✅ 올바른 코드 (트랜잭션 내부는 Promise.all)
// eslint-disable-next-line no-restricted-syntax -- Promise.all은 트랜잭션 내부에서 올바른 선택
await Promise.all([
  StepModel.deleteMany({ user_id }).session(session),
  ...
]);
```

**배경**: ESLint `no-restricted-syntax` 규칙이 `Promise.all`을 경고하지만, 이 규칙은 트랜잭션 맥락을 알지 못한다. `promiseAllPatiently`는 실패 후에도 나머지 항목을 계속 실행하기 때문에 트랜잭션 내부에서 사용하면 안 된다.

**룰**: 트랜잭션(`runMongooseTransaction`) 내부에서 `promiseAllPatiently` 절대 사용 금지. ESLint 경고가 뜨면 `eslint-disable` 처리.

---

### 지적 2: 기존 Skill 체계가 LESSONS_LEARNED.md를 포함하지 않아 자동 검토 누락

**배경**: `LESSONS_LEARNED.md`가 `packages/server-internal/` 루트에 untracked 파일로만 존재했고, skill 체크 흐름에 포함되지 않았다. 결과적으로 PR 전 convention 체크 시 이 파일을 참조하지 않아 `promiseAllPatiently` 규칙을 놓쳤다.

**룰**: 앞으로 PR 전 체크는 `lessons-learned` skill의 checklist.md를 반드시 포함한다.

---

### 지적 3: import 정렬 순서 위반 (sort-imports)

```ts
// ❌ 알파벳 순서 위반
  CashHistoryModel,
  AiDietContextModel,  // A가 C보다 앞이어야 함

// ✅ 올바른 순서
  AiDietContextModel,
  CashHistoryModel,
```

**룰**: 새 import 추가 시 알파벳 순서 확인. 특히 기존 블록 중간에 끼워 넣을 때 주의.

---

### 지적 4: import 그룹 내 순서 위반 (import/order)

```ts
// ❌ aws-lambda 뒤에 async-retry import
import type { APIGatewayProxyHandler } from "aws-lambda";
import retry from "async-retry";  // async-retry < aws-lambda 알파벳 순

// ✅ 올바른 순서
import retry from "async-retry";
import type { APIGatewayProxyHandler } from "aws-lambda";
```

**룰**: 새 패키지 import 추가 시 같은 그룹 내 알파벳 순서 확인.

---

## checklist.md에 추가된 항목

- 트랜잭션(`runMongooseTransaction`) 내부에서 `promiseAllPatiently` 사용했는가? → 금지, `Promise.all`로 교체 (ESLint 경고는 `eslint-disable` 처리)
- 새 import 추가 시 알파벳 순서(sort-imports, import/order) 확인
