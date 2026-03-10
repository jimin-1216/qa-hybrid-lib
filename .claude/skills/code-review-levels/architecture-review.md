# Level 4: Architecture Review (QA 특화)

테스트 아키텍처 관점의 설계 리뷰입니다. 개별 테스트 코드가 아닌 **레이어 구조**, **POM 패턴 준수**, **CI/CD 통합성**, **재사용성**을 분석합니다.

## When to Use

- 새로운 테스트 모듈/프로젝트 추가 시
- Core 레이어 변경이 있을 때
- Page Object 구조 리팩토링 시
- CI/CD 파이프라인 최적화 검토 시
- 새 프로젝트에 qa-hybrid-lib 도입 시
- 분기별 테스트 아키텍처 건강도 점검 시

---

## Architecture Review Protocol

### Phase 1: Layer Compliance (레이어 준수)

**목적**: Core(Immutable) / Recipes(Mutable) / Page Objects / Specs 레이어 분리 검증

**표준 레이어 구조:**
```
┌────────────────────────────────────────┐
│              Specs Layer               │
│     (*.spec.ts - 테스트 시나리오)      │
├────────────────────────────────────────┤
│           Page Objects Layer           │
│    (페이지별 액션/검증 캡슐화)         │
├────────────────────────────────────────┤
│          Recipes Layer (Mutable)       │
│   (프로젝트별 확장, 커스터마이징)      │
├────────────────────────────────────────┤
│          Core Layer (Immutable)        │
│    (공통 유틸, Base 클래스, 헬퍼)      │
└────────────────────────────────────────┘
```

**레이어 의존 규칙:**

| From | To | 허용 | 위반 시 문제 |
|------|----|----|------------|
| Specs | Page Objects | O | - |
| Specs | Recipes | O | - |
| Specs | Core | O (유틸만) | - |
| Page Objects | Core | O | - |
| Page Objects | Recipes | X | 역방향 의존 |
| Recipes | Core | O | - |
| Core | Recipes | X | Immutable 원칙 위반 |
| Core | Page Objects | X | 역방향 의존 |
| Core | Specs | X | 역방향 의존 |

**체크리스트:**
- [ ] Core 레이어에 앱 비즈니스 로직 의존이 없는가?
- [ ] Core 레이어가 특정 프로젝트에 종속되지 않는가?
- [ ] Recipes 레이어가 Core를 올바르게 확장하는가?
- [ ] Page Object에서 assertion을 직접 수행하지 않는가?
- [ ] Specs에서 Page Object를 통하지 않은 직접 DOM 조작이 없는가?

**위반 탐지 출력:**
```
## Layer Compliance Report

### 위반 사항
| Severity | From → To | 파일 | 설명 | 수정 방안 |
|----------|-----------|------|------|----------|
| Critical | Core → Recipes | core/utils.ts:34 | 프로젝트 전용 헬퍼 import | Recipes로 이동 |
| High | PageObject → assertion | pages/LoginPage.ts:56 | `expect()` 직접 호출 | Spec으로 assertion 이동 |
| Medium | Spec → DOM 직접 | login.spec.ts:23 | `page.locator()` 직접 사용 | Page Object 메서드 추가 |

### Layer Health: 65/100
```

---

### Phase 2: POM Pattern Compliance (Page Object 패턴 준수)

**목적**: Page Object Model 설계 원칙 준수 여부 검증

**체크리스트:**

#### Base Page Object 상속
- [ ] 모든 Page Object가 BasePage를 상속하는가?
- [ ] BasePage에 공통 메서드(navigate, waitForLoad 등)가 정의되어 있는가?
- [ ] 상속 깊이가 3단계를 초과하지 않는가?

#### Selector 전략 일관성
- [ ] `data-testid` 우선 사용하는가?
- [ ] Selector 우선순위: `data-testid > role > CSS` 준수하는가?
- [ ] Selector가 Page Object 내에 캡슐화되어 있는가? (Spec에 노출 금지)
- [ ] 동적 Selector가 메서드 파라미터로 처리되는가?

#### 액션 캡슐화
- [ ] 사용자 액션이 의미 있는 메서드명으로 캡슐화되어 있는가?
- [ ] 메서드가 적절한 반환값(다음 Page Object 또는 void)을 제공하는가?
- [ ] 복합 액션이 단일 메서드로 제공되는가? (예: `login(id, pw)`)
- [ ] getter 메서드로 상태 조회가 가능한가?

**Selector 전략 점검표:**
```
## Selector Strategy Analysis

### Selector 사용 분포
| 전략 | 사용 횟수 | 비율 | 권장 비율 | 상태 |
|------|----------|------|----------|------|
| data-testid | 45 | 30% | 60%+ | 부족 |
| role | 25 | 17% | 20% | 적절 |
| CSS class | 60 | 40% | 10% 이하 | 과다 |
| XPath | 20 | 13% | 5% 이하 | 과다 |

### 위반 사례
| 파일 | 라인 | 현재 | 권장 |
|------|------|------|------|
| LoginPage.ts:12 | `.login-btn` | `[data-testid="login-submit"]` |
| SearchPage.ts:8 | `//div[@class='result']` | `[data-testid="search-result"]` |

### POM Score: 55/100
```

---

### Phase 3: Config & Environment (설정/환경 분리)

**목적**: 테스트 설정의 유연성과 환경별 분리 검증

**체크리스트:**

#### playwright.config.ts 멀티 프로젝트 구성
- [ ] 브라우저별 프로젝트가 정의되어 있는가? (chromium, firefox, webkit)
- [ ] 모바일 뷰포트 프로젝트가 포함되어 있는가?
- [ ] 프로젝트별 설정(timeout, retries 등)이 적절한가?
- [ ] baseURL이 환경변수로 주입되는가?

#### 환경 변수 주입
- [ ] 테스트 대상 URL이 환경변수로 관리되는가?
- [ ] 테스트 계정 정보가 환경변수로 관리되는가?
- [ ] `.env.example` 파일이 제공되는가?
- [ ] CI/CD 환경에서 시크릿이 안전하게 주입되는가?

#### Hard-coding 검사
- [ ] URL이 하드코딩되어 있지 않은가?
- [ ] 포트 번호가 하드코딩되어 있지 않은가?
- [ ] 타임아웃 값이 config에서 중앙 관리되는가?

**설정 분석 출력:**
```
## Config & Environment Analysis

### playwright.config.ts 분석
| 항목 | 상태 | 설명 |
|------|------|------|
| 멀티 프로젝트 | O | chromium, firefox, webkit |
| 모바일 뷰포트 | X | 미구성 |
| baseURL 환경변수 | O | `process.env.BASE_URL` |
| 글로벌 timeout | O | 30000ms |
| retries | O | CI: 2, Local: 0 |

### Hard-coding 발견
| 파일 | 라인 | 하드코딩 값 | 권장 |
|------|------|------------|------|
| helpers/api.ts:3 | `http://localhost:3000` | `process.env.API_URL` |
| fixtures/config.ts:7 | `timeout: 5000` | `config.timeout` 참조 |

### Config Score: 70/100
```

---

### Phase 4: CI/CD Integration (CI/CD 통합성)

**목적**: 테스트가 CI/CD 파이프라인에 효과적으로 통합되어 있는지 검증

**체크리스트:**

#### Tag-Runner 매핑
- [ ] @smoke 태그가 PR 빌드에서 실행되는가?
- [ ] @regression 태그가 정기 빌드에서 실행되는가?
- [ ] @native 태그가 특정 환경에서만 실행되는가?
- [ ] 태그별 실행 시간이 합리적인가? (smoke < 5분, regression < 30분)

#### Sharding 구성
- [ ] 대규모 테스트 실행 시 sharding이 구성되어 있는가?
- [ ] shard 간 테스트 분배가 균등한가?
- [ ] shard 결과가 올바르게 병합되는가?

#### Artifact 관리
- [ ] 실패 시 trace 파일이 저장되는가?
- [ ] 실패 시 스크린샷이 저장되는가?
- [ ] HTML 리포트가 생성/배포되는가?
- [ ] artifact 보관 기간이 적절한가?
- [ ] artifact 크기가 관리 가능한 수준인가?

**CI/CD 분석 출력:**
```
## CI/CD Integration Analysis

### Tag-Runner 매핑
| 태그 | 실행 시점 | 예상 시간 | 실제 시간 | 상태 |
|------|----------|----------|----------|------|
| @smoke | PR 빌드 | < 5분 | 3분 | 적절 |
| @regression | Nightly | < 30분 | 45분 | 초과 |
| @native | Manual | < 10분 | 8분 | 적절 |

### Artifact 구성
| 항목 | 상태 | 설정 |
|------|------|------|
| Trace on failure | O | `trace: 'retain-on-failure'` |
| Screenshot on failure | O | `screenshot: 'only-on-failure'` |
| HTML Report | O | `reporter: 'html'` |
| 보관 기간 | X | 미설정 (기본값 사용) |

### 개선 권장
1. @regression 실행 시간 최적화 (sharding 도입 또는 테스트 분류)
2. artifact 보관 기간 명시 (30일 권장)
3. 테스트 결과 대시보드 연동

### CI/CD Score: 65/100
```

---

### Phase 5: Reusability Assessment (재사용성 평가)

**목적**: 테스트 코드의 재사용성과 새 프로젝트 도입 용이성 평가

**체크리스트:**

#### Core 모듈 독립성
- [ ] Core 모듈이 특정 앱/도메인에 의존하지 않는가?
- [ ] Core 유틸리티가 범용적인가?
- [ ] Core의 npm 패키지화가 가능한 수준인가?
- [ ] Core 변경 시 하위 호환성이 유지되는가?

#### Recipe 확장 패턴
- [ ] 새 프로젝트에서 Recipe를 쉽게 생성할 수 있는가?
- [ ] Recipe 생성 가이드/템플릿이 제공되는가?
- [ ] Recipe 간 독립성이 보장되는가?
- [ ] Recipe가 Core의 올바른 확장 포인트를 사용하는가?

#### 새 프로젝트 도입 용이성
- [ ] 초기 설정 가이드가 문서화되어 있는가?
- [ ] boilerplate 코드가 최소화되어 있는가?
- [ ] 프로젝트별 커스터마이징 포인트가 명확한가?
- [ ] 의존성 설치가 간단한가?

**재사용성 분석 출력:**
```
## Reusability Assessment

### Core 모듈 독립성
| 모듈 | 프로젝트 의존 | 독립 사용 가능 | 상태 |
|------|-------------|--------------|------|
| BasePage | 없음 | O | 적절 |
| ApiHelper | 없음 | O | 적절 |
| TestDataFactory | 프로젝트A 스키마 의존 | X | 리팩토링 필요 |
| ReportHelper | 없음 | O | 적절 |

### Recipe 확장 용이성
| 항목 | 상태 | 설명 |
|------|------|------|
| 템플릿 제공 | X | Recipe 생성 템플릿 미제공 |
| 문서화 | 부분 | 일부 Recipe만 문서화 |
| 독립성 | O | Recipe 간 의존 없음 |

### 새 프로젝트 도입 체크리스트
| 항목 | 상태 | 예상 소요 |
|------|------|----------|
| 초기 설정 | 복잡 | 2-3일 |
| 첫 테스트 작성 | 보통 | 1일 |
| CI/CD 통합 | 복잡 | 1-2일 |
| 총 도입 기간 | | 4-6일 |

### Reusability Score: 60/100
```

---

## Architecture Review Output Format

```markdown
# Architecture Review Report (QA)

## Overview
- **Scope**: 전체 테스트 아키텍처 / 특정 모듈
- **Review Level**: Level 4 (Architecture)
- **Architecture Style**: Core-Recipe Layered + POM

## Executive Summary
테스트 아키텍처는 전반적으로 **양호**하나, **레이어 위반** 3건과
**POM 패턴 위반** 5건이 발견되었습니다. CI/CD 통합성과
재사용성 측면에서 개선이 필요합니다.

## Architecture Health Score

| Category | Score | Grade |
|----------|-------|-------|
| Layer Compliance | 65/100 | C |
| POM Pattern | 55/100 | D+ |
| Config & Environment | 70/100 | B- |
| CI/CD Integration | 65/100 | C |
| Reusability | 60/100 | D+ |
| **Overall** | **63/100** | **C-** |

## Key Findings

### 1. Layer Compliance
[레이어 위반 사항...]

### 2. POM Pattern Violations
[POM 패턴 위반 목록...]

### 3. Config & Environment
[설정/환경 분석...]

### 4. CI/CD Integration
[CI/CD 통합성 분석...]

### 5. Reusability
[재사용성 평가...]

## Improvement Roadmap

### Immediate (이번 스프린트)
1. Core 레이어의 프로젝트 의존 코드 Recipes로 이동
2. Page Object 내 assertion 제거

### Short-term (1개월)
1. Selector 전략 통일 (data-testid 우선)
2. @regression 실행 시간 최적화
3. Hard-coding 환경변수 전환

### Long-term (분기)
1. Recipe 생성 템플릿/가이드 작성
2. Core 모듈 npm 패키지화 검토
3. 테스트 결과 대시보드 구축

## Approval Status
- [ ] 아키텍처 재검토 필요
- [x] 조건부 승인 - 로드맵 이행 필요
- [ ] 승인 - 현 아키텍처 유지
```

---

## Architecture Review Checklist Summary

```
Architecture Review Checklist (Level 4 - QA 특화)

□ Layer Compliance
  □ Core(Immutable) / Recipes(Mutable) 분리 검증
  □ Core에 앱 비즈니스 로직 의존 여부
  □ Page Object에서 assertion 직접 수행 여부
  □ 레이어 간 의존 방향 검증

□ POM Pattern Compliance
  □ Base Page Object 상속 구조
  □ Selector 전략 일관성 (data-testid > role > CSS)
  □ 액션 캡슐화 수준
  □ 메서드 반환값 적절성

□ Config & Environment
  □ playwright.config.ts 멀티 프로젝트 구성
  □ 환경 변수 주입 방식
  □ Hard-coding 여부

□ CI/CD Integration
  □ Tag-Runner 매핑 적절성
  □ Sharding 구성
  □ Artifact 관리 (trace, screenshot, report)
  □ 실행 시간 최적화

□ Reusability
  □ Core 모듈 독립성
  □ Recipe 확장 패턴
  □ 새 프로젝트 도입 용이성
  □ 문서화 수준
```
