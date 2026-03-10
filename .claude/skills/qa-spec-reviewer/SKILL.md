---
name: qa-spec-reviewer
description: Playwright spec 파일 품질 리뷰. Flaky 패턴 감지, 태그 규칙 준수, 테스트 구조 검증. qa-hybrid-lib 전용.
allowed-tools: Read, Grep, Glob
---

# QA Spec Reviewer

Playwright spec 파일의 품질을 리뷰합니다. Hybrid App (PWA + Capacitor) QA 컨텍스트에 특화된 스킬입니다.

## 1. Flaky 패턴 감지 (자동 차단)

### Critical

| 패턴 | Severity | 이유 |
|------|----------|------|
| `page.waitForTimeout()` | Critical | 절대 금지. `waitForSelector`, `waitForLoadState`, `waitForResponse` 사용 |
| `page.wait(숫자)` | Critical | 하드코딩 대기 금지 |
| `sleep()` / `setTimeout()` in test code | Critical | 테스트 코드 내 하드코딩 대기 금지 |

### Major

| 패턴 | Severity | 이유 |
|------|----------|------|
| 고정 좌표 클릭 (`page.click({position: {x, y}})`) | Major | 해상도/레이아웃 변경 시 실패 |
| `nth()` 셀렉터 | Major | DOM 순서 의존. 구조 변경 시 실패 |
| `page.evaluate()` 내 DOM 직접 조작 | Major | assertion 우회. 실제 사용자 행동과 괴리 |
| `test.describe.serial` (명시적 사유 없음) | Major | 순서 의존 테스트. 병렬 실행 불가 |

### Minor

| 패턴 | Severity | 이유 |
|------|----------|------|
| 하드코딩된 URL (`http://localhost:3000` 등) | Minor | `BASE_URL` 환경변수 사용 필요 |
| 매직 넘버 (타임아웃 등 숫자 리터럴) | Minor | 상수 또는 config로 추출 필요 |
| 주석 없는 복잡한 셀렉터 | Minor | 유지보수성 저하 |

## 2. 태그 규칙 검증

- **`@smoke`** — spec당 최대 5개 test. 핵심 플로우만 포함해야 합니다.
- **`@regression`** — 전체 시나리오. 태그 없는 테스트는 자동으로 regression으로 간주합니다.
- **`@native`** — Capacitor 관련 테스트만 사용. `cap-mock` 프로젝트에서만 실행되는지 확인합니다.
- **`@ios`** — iOS 전용 테스트. macOS Runner 필수 표시가 있는지 확인합니다.
- 미태깅 테스트는 경고를 발생시킵니다.

## 3. 구조 규칙 검증

- **네이밍**: `{도메인}.{시나리오}.spec.ts` 패턴 준수 여부를 확인합니다.
- **테스트 수**: 1 spec 파일당 최대 10 test case.
- **중첩 깊이**: `describe`/`it` 중첩 최대 3레벨.
- **상태 복원**: `beforeAll`/`afterAll`에서 상태 변경 시 `afterAll`에서 복원하는지 확인합니다.
- **설정 위치**: `test.use()` 설정이 spec 상단에 있는지 확인합니다.

## 4. Assertion 품질

- 모든 test에 최소 1개 assertion이 있어야 합니다.
- `expect(true).toBe(true)` 같은 무의미한 assertion을 감지합니다.
- 시각적 회귀 테스트: `toHaveScreenshot()` 사용 시 `maxDiffPixels` 설정이 있는지 확인합니다.
- API 응답 검증: status code와 body를 모두 확인하는지 검증합니다.

## 5. 출력 포맷

리뷰 결과는 다음 형식으로 출력합니다:

```
## Spec Review Report

### 파일: {파일명}
- **점수**: {점수}/100
- **Flaky Risk Score**: {점수} (낮을수록 좋음)

### 발견 이슈

#### Critical
- [ ] {이슈 설명} (라인 {번호})

#### Major
- [ ] {이슈 설명} (라인 {번호})

#### Minor
- [ ] {이슈 설명} (라인 {번호})

### 개선 코드 예시

**Before:**
```typescript
// 문제 코드
```

**After:**
```typescript
// 개선 코드
```
```

### 점수 산정 기준

| 항목 | 감점 |
|------|------|
| Critical 이슈 | -20점/건 |
| Major 이슈 | -10점/건 |
| Minor 이슈 | -3점/건 |
| 미태깅 테스트 | -2점/건 |
| assertion 없는 테스트 | -15점/건 |
