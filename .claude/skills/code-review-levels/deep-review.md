# Level 3: Deep Review (QA 특화)

시니어 QA 엔지니어 관점의 심층 테스트 코드 리뷰입니다. 단순 코드 품질을 넘어 **테스트 안정성**, **Flaky 테스트 탐지**, **데이터 격리**, **커버리지 효과**를 분석합니다.

## When to Use

- 핵심 비즈니스 시나리오의 E2E 테스트 변경 시
- Flaky 테스트가 빈번하게 보고될 때
- 테스트 데이터 격리 문제가 의심될 때
- 병렬 실행 시 테스트 실패가 발생할 때
- PR 리뷰어가 "자세히 봐달라"고 요청 시

---

## Deep Review Protocol

### Phase 1: Test Flow Analysis (테스트 흐름 분석)

**목적**: spec 파일의 테스트 실행 흐름과 의존성 구조를 파악

```bash
# spec 파일의 test/describe 구조 추적
Grep: "test\\(|test\\.describe\\(|test\\.beforeAll\\(|test\\.afterAll\\(" in **/*.spec.ts

# fixture/page-object 의존성 파악
Read: 대상 spec 파일 → import 문 분석 → fixture, page object 참조 추적

# setup/teardown 체인 분석
Grep: "beforeAll|beforeEach|afterAll|afterEach" in 대상 파일
```

**체크리스트:**
- [ ] spec 파일의 test 블록 구조가 논리적으로 그룹화되어 있는가?
- [ ] fixture 의존성이 명확하게 정의되어 있는가?
- [ ] beforeAll/afterAll 체인에서 리소스 누수가 없는가?
- [ ] test.describe 중첩 깊이가 3단계를 초과하지 않는가?
- [ ] 각 test 블록의 독립 실행이 가능한가?

**출력 예시:**
```
## Test Flow Analysis

### Spec 구조: login.spec.ts
1. `test.describe('로그인 시나리오')`
   ├── `test.beforeAll()` → 테스트 사용자 생성 (API)
   ├── `test('정상 로그인')` → LoginPage.login() → DashboardPage 확인
   ├── `test('잘못된 비밀번호')` → LoginPage.login() → 에러 메시지 확인
   ├── `test('계정 잠금')` → 5회 실패 → 잠금 메시지 확인
   └── `test.afterAll()` → 테스트 사용자 삭제 (API)

### Fixture Dependencies
- `LoginPage` ← `BasePage` (core)
- `testUser` fixture ← `apiClient` fixture ← `authToken` fixture

### Risk Assessment
- beforeAll에서 API 호출 실패 시 전체 describe 블록 실패 위험 ⚠️
- test('계정 잠금')이 이전 테스트의 실패 횟수에 의존 ⚠️
```

---

### Phase 2: Flaky Test Detection (불안정 테스트 탐지)

**목적**: 비결정적 테스트 실패를 유발하는 패턴 탐지

**Severity 기준:**

| Severity | 패턴 | 설명 |
|----------|------|------|
| Critical | `waitForTimeout()` 사용 | 고정 대기 시간은 환경에 따라 불안정 |
| Critical | 하드코딩된 셀렉터 | DOM 구조 변경에 취약 |
| High | 네트워크 의존 테스트 | 외부 서비스 상태에 의존 |
| High | 순서 의존적 테스트 | 이전 테스트 결과에 의존 |
| Medium | 타이밍 기반 assertion | 애니메이션/전환 완료 가정 |
| Low | 스크린샷 비교 의존 | 렌더링 차이로 실패 가능 |

**탐지 규칙:**

```bash
# Critical: waitForTimeout 사용 탐지
Grep: "waitForTimeout\\(" in **/*.spec.ts **/*.ts

# Critical: 하드코딩된 CSS 셀렉터
Grep: "page\\.locator\\(['\"]\\." in **/*.spec.ts   # class 셀렉터
Grep: "page\\.locator\\(['\"]#" in **/*.spec.ts     # id 셀렉터 (data-testid 아닌 경우)

# High: 순서 의존적 패턴
# test.describe.serial 사용 또는 공유 변수 감지
Grep: "describe\\.serial|let\\s+\\w+.*=.*;" in **/*.spec.ts

# 권장 대기 패턴
# page.waitForSelector(), page.waitForLoadState(), expect().toBeVisible()
```

**권장 대기 전략:**

| 안티패턴 | 권장 패턴 | 이유 |
|----------|----------|------|
| `page.waitForTimeout(3000)` | `page.waitForSelector('.element')` | 상태 기반 대기로 안정성 확보 |
| `page.waitForTimeout(5000)` | `page.waitForLoadState('networkidle')` | 네트워크 완료 기반 대기 |
| `page.waitForTimeout(1000)` | `await expect(locator).toBeVisible()` | assertion 기반 자동 재시도 |
| `page.waitForTimeout(2000)` | `page.waitForResponse(url)` | 특정 API 응답 대기 |

**분석 출력:**
```
## Flaky Test Detection Report

### Critical (즉시 수정)
| 파일 | 라인 | 패턴 | 현재 코드 | 권장 수정 |
|------|------|------|----------|----------|
| login.spec.ts | 45 | waitForTimeout | `await page.waitForTimeout(3000)` | `await page.waitForLoadState('networkidle')` |
| search.spec.ts | 23 | 하드코딩 셀렉터 | `page.locator('.btn-primary')` | `page.locator('[data-testid="search-btn"]')` |

### High (1주 내 수정)
| 파일 | 라인 | 패턴 | 설명 | 권장 수정 |
|------|------|------|------|----------|
| order.spec.ts | 12-55 | 순서 의존 | test들이 공유 변수 `orderId` 사용 | 각 test에서 독립적으로 주문 생성 |

### Flaky Risk Score: 35/100 (높음)
- Critical 이슈: 2건 → 즉시 수정 필요
- High 이슈: 1건 → 스프린트 내 수정
```

---

### Phase 3: Test Data Isolation (테스트 데이터 격리)

**목적**: 테스트 간 데이터 간섭으로 인한 실패 방지

**체크리스트:**
- [ ] **상태 공유**: 테스트 간 공유되는 변수/상태가 있는가?
- [ ] **DB 데이터**: 테스트가 공유 DB 데이터에 의존하는가?
- [ ] **Fixture 격리**: fixture가 테스트별로 독립 생성되는가?
- [ ] **Cleanup**: 테스트 후 생성된 데이터가 정리되는가?
- [ ] **병렬 안전**: `--workers` 옵션으로 병렬 실행 시 안전한가?
- [ ] **고유 데이터**: 테스트 데이터에 고유 식별자(timestamp, UUID)를 사용하는가?

**격리 수준 평가:**
```
## Test Data Isolation Assessment

### 격리 위반 사항
| 파일 | 유형 | 설명 | 영향 |
|------|------|------|------|
| user.spec.ts | 상태 공유 | `let userId`가 describe 레벨에서 공유 | 병렬 실행 시 race condition |
| order.spec.ts | 하드코딩 데이터 | 고정 이메일 `test@example.com` 사용 | 병렬 실행 시 충돌 |
| search.spec.ts | DB 의존 | 사전 생성된 검색 데이터에 의존 | 다른 테스트가 데이터 변경 시 실패 |

### 격리 수준 등급
| 등급 | 설명 | 현재 상태 |
|------|------|----------|
| A | 완전 격리 (테스트별 독립 데이터) | |
| B | 부분 격리 (describe 레벨 공유) | |
| C | 미흡 (글로벌 공유 데이터) | ← 현재 |

### 권장 개선
1. 테스트 데이터에 `test.info().testId` 기반 고유 접미사 사용
2. API 기반 setup/teardown으로 테스트 데이터 생명주기 관리
3. `test.describe.configure({ mode: 'parallel' })` 적용 전 격리 검증
```

---

### Phase 4: Coverage & Effectiveness (커버리지 & 효과)

**목적**: 테스트가 비즈니스 가치를 효과적으로 보호하는지 평가

**체크리스트:**
- [ ] **비즈니스 시나리오**: 핵심 사용자 플로우가 테스트되고 있는가?
- [ ] **엣지 케이스**: 에러 상태, 빈 상태, 경계값이 테스트되는가?
- [ ] **네거티브 테스트**: 잘못된 입력, 권한 없는 접근이 테스트되는가?
- [ ] **크로스 브라우저**: 지원 브라우저별 테스트가 구성되어 있는가?
- [ ] **태그 분포**: @smoke, @regression, @native 태그가 적절히 분포되어 있는가?

**태그 분포 분석:**
```
## Tag Distribution Analysis

### 현재 태그 분포
| 태그 | 테스트 수 | 비율 | 권장 비율 | 상태 |
|------|----------|------|----------|------|
| @smoke | 15 | 10% | 10-15% | 적절 |
| @regression | 120 | 80% | 60-70% | 과다 |
| @native | 5 | 3% | 10-15% | 부족 |
| 태그 없음 | 10 | 7% | 0% | 수정 필요 |

### 비즈니스 시나리오 커버리지
| 시나리오 | 테스트 존재 | 태그 | 상태 |
|----------|-----------|------|------|
| 회원 가입 | O | @smoke | 적절 |
| 로그인/로그아웃 | O | @smoke | 적절 |
| 상품 검색 | O | @regression | 적절 |
| 결제 플로우 | X | - | 누락 |
| 에러 페이지 | X | - | 누락 |

### 권장 사항
1. 결제 플로우 E2E 테스트 추가 (비즈니스 임팩트 높음)
2. @regression 태그 중 핵심 항목을 @smoke로 승격
3. 태그 없는 테스트에 적절한 태그 부여
```

---

### Phase 5: Security in Tests (테스트 보안)

**목적**: 테스트 코드 및 테스트 환경에서의 보안 취약점 검사

**체크리스트:**

**실제 Credential 노출:**
- [ ] fixture 파일에 실제 사용자 비밀번호가 하드코딩되어 있는가?
- [ ] API 키, 토큰이 소스 코드에 직접 포함되어 있는가?
- [ ] `.env` 파일이 `.gitignore`에 포함되어 있는가?

**환경변수 사용 규칙:**
- [ ] 민감 정보가 `process.env`를 통해 주입되는가?
- [ ] CI 환경에서 시크릿이 적절히 관리되는가?
- [ ] 로컬/CI 환경별 `.env` 템플릿이 제공되는가?

**스크린샷/Trace 보안:**
- [ ] 스크린샷에 민감 정보(PII, 카드번호 등)가 캡처되는가?
- [ ] trace 파일에 API 토큰/세션 정보가 포함되는가?
- [ ] CI artifact에 민감 정보가 업로드되는가?

**보안 분석 출력:**
```
## Test Security Analysis

### 발견된 취약점
| Severity | 유형 | 위치 | 설명 | 권장 수정 |
|----------|------|------|------|----------|
| Critical | 하드코딩 비밀번호 | fixtures/users.ts:12 | `password: 'RealP@ss123'` | 환경변수 사용: `process.env.TEST_PASSWORD` |
| High | API 키 노출 | helpers/api.ts:5 | `apiKey: 'sk-live-xxx'` | `.env` 파일로 분리 |
| Medium | 스크린샷 PII | order.spec.ts:78 | 결제 완료 화면 캡처 시 카드번호 노출 | 마스킹 처리 후 캡처 |

### Security Score: 45/100 (위험)
- 즉시 수정 필요: 2건
- 권장 수정: 1건
```

---

## Deep Review Output Format

```markdown
# Deep Review Report (QA)

## Overview
- **Target**: `tests/specs/login.spec.ts`
- **Review Level**: Level 3 (Deep)
- **Focus**: Flaky 테스트 탐지, 데이터 격리

## Executive Summary
로그인 테스트에서 **Flaky 패턴 3건**, **데이터 격리 위반 2건**이 발견되었습니다.
병렬 실행 안정성 확보를 위해 수정이 필요합니다.

## Findings by Phase

### 1. Test Flow Analysis
[테스트 흐름 분석 결과...]

### 2. Flaky Test Detection (3건 발견)
[불안정 패턴 목록...]

### 3. Test Data Isolation (2건 위반)
[격리 위반 사항...]

### 4. Coverage & Effectiveness
[커버리지 분석...]

### 5. Security in Tests (1건)
[보안 취약점...]

## Score Table

| Category | Score | Weight | Weighted |
|----------|-------|--------|----------|
| Test Flow 구조 | 80/100 | 15% | 12.0 |
| Flaky Risk (역산) | 35/100 | 30% | 10.5 |
| Data Isolation | 50/100 | 25% | 12.5 |
| Coverage | 70/100 | 20% | 14.0 |
| Security | 45/100 | 10% | 4.5 |
| **Total** | | | **53.5/100** |

## Recommendations

### Must Fix (배포 차단)
1. `waitForTimeout()` 전부 상태 기반 대기로 교체
2. 하드코딩된 credential 환경변수로 이동

### Should Fix (1주 내)
1. 테스트 데이터 격리 강화
2. 순서 의존적 테스트 독립화

### Nice to Have
1. 태그 분포 최적화
2. 누락된 비즈니스 시나리오 테스트 추가

## Approval Status
- [ ] 배포 불가 - Critical 이슈 존재
- [x] 조건부 승인 - Flaky 이슈 수정 후 재검토
- [ ] 승인 - 배포 가능
```

---

## Deep Review Checklist Summary

```
Deep Review Checklist (Level 3 - QA 특화)

□ Test Flow Analysis
  □ spec 파일 test 블록 구조 확인
  □ fixture/page-object 의존성 추적
  □ setup/teardown 체인 리소스 누수 확인

□ Flaky Test Detection
  □ waitForTimeout() 사용 여부 (Critical)
  □ 하드코딩된 셀렉터 사용 여부 (Critical)
  □ 네트워크 의존 테스트 존재 여부 (High)
  □ 순서 의존적 테스트 존재 여부 (High)
  □ 상태 기반 대기 패턴 권장 적용 여부

□ Test Data Isolation
  □ 테스트 간 상태 공유 여부
  □ fixture 데이터 격리 수준
  □ 병렬 실행 안전성 검증

□ Coverage & Effectiveness
  □ 핵심 비즈니스 시나리오 커버리지
  □ 엣지 케이스/네거티브 테스트 여부
  □ 태그별 분포 적절성 (@smoke, @regression, @native)

□ Security in Tests
  □ fixture에 실제 credential 포함 여부
  □ 환경변수 사용 규칙 준수 여부
  □ 스크린샷/trace에 민감 정보 노출 여부
```
