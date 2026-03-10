---
name: pom-validator
description: Page Object Model 패턴 준수 검증. Selector 전략, 캡슐화, 재사용성 체크. Playwright + TypeScript 전용.
allowed-tools: Read, Grep, Glob
---

# POM Validator

Page Object Model 패턴 준수 여부를 검증한다. Playwright + TypeScript 프로젝트 전용.

## 1. Base Page Object 구조 검증

기대하는 Base 구조:

```typescript
// src/page-objects/base.page.ts
export abstract class BasePage {
  constructor(protected page: Page) {}

  abstract url(): string;

  async navigate() {
    await this.page.goto(this.url());
    await this.page.waitForLoadState('networkidle');
  }

  // 공통 유틸리티 메서드
}
```

체크리스트:
- [ ] BasePage 클래스가 존재하는가
- [ ] 모든 Page Object가 BasePage를 상속하는가
- [ ] `page` 인스턴스가 `protected`로 선언되어 있는가
- [ ] `navigate()` 메서드가 구현되어 있는가

## 2. Selector 전략 우선순위

| 우선순위 | Selector 방식 | 예시 | 이유 |
|----------|--------------|------|------|
| 1 (최우선) | `data-testid` | `[data-testid="login-btn"]` | UI 변경에 강건, 테스트 전용 |
| 2 | `role` | `getByRole('button', {name: '로그인'})` | 접근성 + 안정성 |
| 3 | `text` | `getByText('로그인')` | 다국어 시 주의 필요 |
| 4 (비권장) | CSS selector | `.btn-primary` | 스타일 변경에 취약 |
| 5 (금지) | XPath | `//div[@class="..."]` | 깨지기 쉬움, 가독성 나쁨 |

감지 규칙:
- XPath 사용 → Major 경고
- CSS class 기반 셀렉터 → Minor 경고 (대안 제시)
- `nth()` 사용 → Major 경고 (DOM 순서 의존)
- 하드코딩 텍스트 셀렉터 → Minor (i18n 위험)

## 3. 캡슐화 규칙

| 규칙 | Severity | 설명 |
|------|----------|------|
| Page Object 내 assertion 금지 | Critical | PO는 액션+조회만. assertion은 spec에서 |
| Selector 외부 노출 금지 | Major | selector는 private, 메서드로만 접근 |
| 직접 page 조작 금지 (spec에서) | Major | `page.click()` 대신 `loginPage.clickSubmit()` |
| 하나의 PO = 하나의 페이지/컴포넌트 | Minor | SRP 준수 |

## 4. 네이밍 컨벤션

```
src/page-objects/
├── base.page.ts          # Base class
├── login.page.ts         # 로그인 페이지
├── dashboard.page.ts     # 대시보드 페이지
├── components/
│   ├── header.component.ts   # 공통 헤더
│   └── modal.component.ts    # 공통 모달
```

규칙:
- 파일명: `{name}.page.ts` 또는 `{name}.component.ts`
- 클래스명: `{Name}Page` 또는 `{Name}Component`
- 메서드명: `동사 + 명사` (예: `clickLoginButton`, `getUsername`, `fillEmail`)

## 5. 재사용성 점검

- 동일 셀렉터가 2+ PO에서 중복 → 공통 컴포넌트로 분리 권고
- 3줄 이상 동일 액션 시퀀스가 2+ spec에서 반복 → PO 메서드 추출 권고
- Capacitor Mock 관련 셀렉터가 일반 PO에 혼재 → 분리 권고

## 6. 출력 포맷

검증 완료 시 아래 형식으로 리포트를 출력한다.

### POM Validation Report

```
═══════════════════════════════════════
  POM Validation Report
═══════════════════════════════════════

POM Compliance Score: XX / 100

카테고리별 점수:
  구조      : XX / 20
  셀렉터    : XX / 25
  캡슐화    : XX / 25
  네이밍    : XX / 15
  재사용성  : XX / 15

위반 목록:
  [Critical] 파일명 — 설명
  [Major]    파일명 — 설명
  [Minor]    파일명 — 설명

Before/After 코드 예시:
  // Before (위반)
  ...
  // After (수정)
  ...
═══════════════════════════════════════
```

모든 사용자 대면 콘텐츠는 한국어로 작성한다.
