# qa-hybrid-lib 구현 계획서

**기준 문서:** PRD v1.3 + PRD Review v1.3
**작성일:** 2026-03-09
**상태:** Phase 0 시작 전

---

## 1. docs 폴더 구조 (용도별 분리)

```
docs/
├── prd/                           # 제품 요구사항 문서
│   ├── PRD_v1.3_qa-hybrid-lib.md
│   └── PRD_REVIEW_v1.3.md
├── architecture/                  # 아키텍처 의사결정 기록
│   ├── layer-separation.md        # Core/Recipes/POM/Specs 레이어 원칙
│   ├── error-handling-strategy.md # [C-1] 모듈별 에러 처리 전략
│   ├── security-policy.md         # [C-2] fixture 보안 정책
│   └── appium-boundary.md         # [M-4] Mock 가능/불가 경계 목록
├── guides/                        # 개발자 가이드
│   ├── getting-started.md         # 신규 프로젝트 도입 가이드 (10분 셋업)
│   ├── writing-specs.md           # [M-3] spec 파일 작성 규칙
│   ├── page-object-guide.md       # [M-2] POM 작성 규칙 + selector 전략
│   └── visual-regression.md       # [M-5] 스냅샷 기준 환경 정책
├── decisions/                     # ADR (Architecture Decision Records)
│   └── 001-scaffolding-over-npm.md
└── IMPLEMENTATION_PLAN.md         # 본 문서
```

---

## 2. 구현 Phase 순서

| Phase | 핵심 산출물 | 반영 이슈 | 의존 관계 |
|-------|------------|----------|----------|
| **0. 초기화** | package.json, tsconfig, .gitignore, .env.example, docs 구조 재편 | — | 없음 |
| **1. Core Layer** | `src/core/` — errors/, types/, capacitor-mock, auth-handler, base-config | C-1, C-2, M-1, M-4 | Phase 0 |
| **2. POM + Specs** | `src/page-objects/base.page.ts`, 예시 PO 2개, 예시 spec 2개 | M-2, M-3 | Phase 1 |
| **3. Recipes** | `src/recipes/` — example-ecommerce, example-dashboard (2개 예시) | m-2 | Phase 1, 2 |
| **4. Playwright Config** | playwright.config.ts 완성, 스냅샷 정책, auth.setup.ts | M-5, M-1 | Phase 1, 2 |
| **5. CI/CD** | GitHub Actions 4개 workflow + 시크릿 스캔 | m-4, C-2 | Phase 4 |
| **6. 문서 완성** | architecture/, guides/, decisions/ 문서 + README 업데이트 | m-1, m-3 | 전체 완료 후 |

---

## 3. Phase 상세

### Phase 0: 프로젝트 초기화 및 문서 정리

**생성 파일:**

| 파일 | 설명 |
|------|------|
| `package.json` | engines: `>=18`, Playwright `^1.42.0`, TypeScript |
| `tsconfig.json` | `strict: true`, `noImplicitAny: true` |
| `playwright.config.ts` | 멀티 프로젝트 뼈대 (Phase 4에서 완성) |
| `.env.example` | `BASE_URL`, `API_URL`, `TEST_PLATFORM`, `MOCK_NATIVE` |
| `.gitignore` | `.env`, `node_modules/`, `test-results/`, `playwright-report/`, `playwright/.auth/` |
| `.eslintrc.js` | TypeScript strict 규칙, `any` 금지 |
| `.prettierrc` | 코드 포맷 설정 |

---

### Phase 1: Core Layer (Immutable) — Critical 이슈 반영

Core는 앱 비즈니스 로직에 의존하지 않는 불변 레이어. C-1(에러 처리), C-2(보안 정책)를 근본적으로 반영한다.

#### 1-1. 에러 처리 기반 인프라 [C-1 반영]

**생성 파일:**

| 파일 | 설명 |
|------|------|
| `src/core/errors/index.ts` | 커스텀 에러 클래스 (`CapacitorMockError`, `AuthHandlerError`, `ConfigError`) |
| `src/core/errors/error-handler.ts` | 중앙 에러 처리 유틸리티 |
| `src/core/types/index.ts` | 공통 인터페이스/타입 정의 |

**에러 처리 전략:**

| 모듈 | 에러 시나리오 | 처리 방식 |
|------|-------------|----------|
| capacitor-mock | Mock 주입 실패 | `test.skip()` + `console.warn` 로그 |
| auth-handler | 토큰 만료 | 자동 재발급 시도 (최대 3회) 후 실패 시 `AuthHandlerError` throw |
| auth-handler | 로그인 API 불가 | configurable timeout 후 명확한 에러 메시지 |
| base-config | 필수 환경변수 누락 | 즉시 `ConfigError` throw (fail-fast) |

#### 1-2. base-config.ts

**생성 파일:** `src/core/base-config.ts`

- 환경변수 로딩 인터페이스 (`BaseConfig` interface)
- 필수 환경변수 검증 (누락 시 `ConfigError` throw — fail-fast)
- `getConfig()` 함수: 타입 안전한 환경변수 접근
- configurable timeout 값 (기본 30000ms, 환경변수로 오버라이드 가능)
- NFR 수치 상수 포함 [M-1 반영]:

```typescript
/** 비기능 요구사항 제한 시간 (ms) */
export const NFR_LIMITS = {
  SMOKE_SUITE_TIMEOUT: 5 * 60 * 1000,      // @smoke: 5분
  REGRESSION_SUITE_TIMEOUT: 20 * 60 * 1000, // @regression: 20분
  SINGLE_SPEC_TIMEOUT: 30 * 1000,           // 단일 spec: 30초
} as const;
```

#### 1-3. capacitor-mock.ts

**생성 파일:** `src/core/capacitor-mock.ts`

- `CapacitorMock.setup(page, config)` — `page.addInitScript()`로 `window.Capacitor.Plugins` 오버라이드
- Mock 가능 플러그인 목록 [M-4 반영]:
  - **Mock 가능:** `Storage`, `Geolocation`, `Camera`(메타데이터), `Haptics`, `StatusBar`, `Keyboard`
  - **Mock 불가 (Appium 필요):** 실제 카메라 촬영, 생체인증, 딥링크, Push 알림 수신
- 에러 처리: Mock 주입 실패 시 `test.skip()` + warning 로그 [C-1 반영]
- 플러그인별 타입 인터페이스 (`CapacitorMockConfig`, `PluginMockMap`)

#### 1-4. auth-handler.ts

**생성 파일:** `src/core/auth-handler.ts`

- `AuthHandler.login(config)`: API 기반 토큰 획득
- `AuthHandler.injectSession(page, session)`: LocalStorage/Cookie 주입
- 자동 재발급 로직: 토큰 만료 감지 시 재시도 (최대 3회, configurable) [C-1 반영]
- `RetryConfig` 인터페이스: `maxRetries`, `retryDelay` 외부 주입 가능

#### 1-5. Fixture 보안 정책 [C-2 반영]

**생성 파일:**

| 파일 | 설명 |
|------|------|
| `src/fixtures/.gitkeep` | 디렉토리 유지 |
| `src/fixtures/README.md` | fixture 작성 규칙 문서화 |
| `.github/hooks/pre-commit-secret-scan.sh` | 시크릿 스캔 스크립트 |
| `.husky/pre-commit` | husky를 통한 pre-commit hook 등록 |

**보안 규칙:**
- fixture 파일 내 실제 토큰/비밀번호/API 키 금지
- Mock 데이터 인터페이스에 `MOCK_` 접두사 컨벤션 강제
- pre-commit hook으로 `fixtures/` 디렉토리 시크릿 패턴 스캔

---

### Phase 2: Page Objects + Spec 구조 — Major 이슈 반영

#### 2-1. Base Page Object [M-2 반영]

**생성 파일:** `src/page-objects/base.page.ts`

```typescript
export abstract class BasePage {
  constructor(protected readonly page: Page) {}
  abstract readonly url: string;
  async navigate(): Promise<void>;
  async waitForPageLoad(): Promise<void>;
}
```

**Selector 전략 우선순위:** `data-testid` > `role` > CSS selector

#### 2-2. 예시 Page Object

**생성 파일:**

| 파일 | 설명 |
|------|------|
| `src/page-objects/example/login.page.ts` | `BasePage` 상속 예시 |
| `src/page-objects/example/dashboard.page.ts` | 두 번째 예시 |

#### 2-3. Spec 구조 규칙 [M-3 반영]

**생성 파일:**

| 파일 | 설명 |
|------|------|
| `src/specs/example/auth.login.spec.ts` | 로그인 시나리오 예시 |
| `src/specs/example/dashboard.navigation.spec.ts` | 대시보드 네비게이션 예시 |

**규칙:**
- 네이밍: `{도메인}.{시나리오}.spec.ts`
- 1 spec당 최대 10 test case
- 태그: `test.describe`에 `@smoke`, `@regression` 등 명시
- `test.use({ timeout: NFR_LIMITS.SINGLE_SPEC_TIMEOUT })` 적용

---

### Phase 3: Recipes Layer (Mutable) — m-2 반영

#### 3-1. Recipe 예시 확장 [m-2 반영]

**생성 파일:**

| 디렉토리 | 파일 |
|----------|------|
| `src/recipes/example-ecommerce/` | `ecommerce.config.ts`, `product.page.ts`, `checkout.spec.ts` |
| `src/recipes/example-dashboard/` | `dashboard.config.ts`, `analytics.page.ts`, `report.spec.ts` |

각 recipe는 `src/core/`를 import하여 사용하는 패턴을 보여준다.

---

### Phase 4: Playwright Config + 시각적 회귀

#### 4-1. playwright.config.ts (상세 구현)

Phase 0에서 생성한 뼈대를 완성:

- 4개 프로젝트: `web`, `mobile-pwa`, `mobile-ios`, `cap-mock`
- `globalSetup`: `src/core/auth-handler.ts` 기반 인증 상태 저장
- `retries`: CI에서만 2회
- `timeout`: `NFR_LIMITS.SINGLE_SPEC_TIMEOUT` (30초) [M-1 반영]
- `grep`/`grepInvert`: 태그 기반 필터링
- `snapshotPathTemplate`: OS별 분리

#### 4-2. 시각적 회귀 테스트 환경 고정 [M-5 반영]

**생성 파일:** `tests/snapshots/.gitkeep`

**정책:**
- 기준 스냅샷은 CI(Linux ubuntu-latest)에서만 생성
- 로컬 개발자는 `--update-snapshots` 사용 금지 (CI에서만 업데이트)
- `maxDiffPixels` 설정으로 미세 렌더링 차이 허용
- 스냅샷 파일명: `{testName}-{projectName}-{platform}.png`

#### 4-3. Auth Setup

**생성 파일:** `src/auth.setup.ts`

globalSetup에서 1회 UI 로그인 후 `playwright/.auth/user.json` 저장.

---

### Phase 5: CI/CD 파이프라인

#### 5-1. GitHub Actions Workflows

**생성 파일:**

| 파일 | 트리거 | 실행 범위 | Runner | Timeout |
|------|--------|----------|--------|---------|
| `.github/workflows/pr-smoke.yml` | Pull Request | `@smoke` | Linux | 5분 |
| `.github/workflows/main-regression.yml` | Main merge | `@regression` 전체 | Linux | 20분 |
| `.github/workflows/native-ios.yml` | 수동/스케줄 | `@native` + `@ios` | macOS | — |
| `.github/workflows/secret-scan.yml` | Pull Request | fixture 시크릿 스캔 | Linux | — |

**포함 사항:**
- Sharding 설정 (테스트 병렬 분산)
- Playwright HTML 리포트 + Trace Zip Artifact 저장
- Allure 리포트 생성 및 Artifact 저장 [m-4 반영]
- 시간 제한 강제 (`timeout-minutes`)

#### 5-2. Allure 리포트 설정 [m-4 반영]

Allure reporter를 Playwright에 추가, CI에서 `allure generate` 후 Artifact로 저장. GitHub Pages 자동 배포는 Phase 2에서 검토.

---

### Phase 6: 문서 완성 + 최종 검증

**생성 파일:**

| 파일 | 반영 이슈 |
|------|----------|
| `docs/architecture/layer-separation.md` | — |
| `docs/architecture/error-handling-strategy.md` | C-1 |
| `docs/architecture/security-policy.md` | C-2 |
| `docs/architecture/appium-boundary.md` | M-4 |
| `docs/guides/getting-started.md` | — |
| `docs/guides/writing-specs.md` | M-3 |
| `docs/guides/page-object-guide.md` | M-2 |
| `docs/guides/visual-regression.md` | M-5 |
| `docs/decisions/001-scaffolding-over-npm.md` | — |
| `README.md` 업데이트 | m-1 |

**Scale Grade 명시 [m-1 반영]:**
- Phase 1 = Startup급 (단일 팀, 1~3개 프로젝트)
- Phase 2 = Growth급 (다수 팀, npm 패키지화 검토)

---

## 4. 최종 파일 트리 (완성 시)

```
qa-hybrid-lib/
├── .github/
│   ├── hooks/
│   │   └── pre-commit-secret-scan.sh
│   └── workflows/
│       ├── pr-smoke.yml
│       ├── main-regression.yml
│       ├── native-ios.yml
│       └── secret-scan.yml
├── .husky/
│   └── pre-commit
├── docs/
│   ├── prd/
│   │   ├── PRD_v1.3_qa-hybrid-lib.md
│   │   └── PRD_REVIEW_v1.3.md
│   ├── architecture/
│   │   ├── layer-separation.md
│   │   ├── error-handling-strategy.md
│   │   ├── security-policy.md
│   │   └── appium-boundary.md
│   ├── guides/
│   │   ├── getting-started.md
│   │   ├── writing-specs.md
│   │   ├── page-object-guide.md
│   │   └── visual-regression.md
│   ├── decisions/
│   │   └── 001-scaffolding-over-npm.md
│   └── IMPLEMENTATION_PLAN.md
├── src/
│   ├── core/
│   │   ├── errors/
│   │   │   ├── index.ts
│   │   │   └── error-handler.ts
│   │   ├── types/
│   │   │   └── index.ts
│   │   ├── capacitor-mock.ts
│   │   ├── auth-handler.ts
│   │   └── base-config.ts
│   ├── recipes/
│   │   ├── example-ecommerce/
│   │   │   ├── ecommerce.config.ts
│   │   │   ├── product.page.ts
│   │   │   └── checkout.spec.ts
│   │   └── example-dashboard/
│   │       ├── dashboard.config.ts
│   │       ├── analytics.page.ts
│   │       └── report.spec.ts
│   ├── page-objects/
│   │   ├── base.page.ts
│   │   └── example/
│   │       ├── login.page.ts
│   │       └── dashboard.page.ts
│   ├── fixtures/
│   │   ├── README.md
│   │   └── .gitkeep
│   ├── specs/
│   │   └── example/
│   │       ├── auth.login.spec.ts
│   │       └── dashboard.navigation.spec.ts
│   └── auth.setup.ts
├── tests/
│   └── snapshots/
│       └── .gitkeep
├── .env.example
├── .eslintrc.js
├── .gitignore
├── .prettierrc
├── package.json
├── playwright.config.ts
├── tsconfig.json
└── README.md
```

---

## 5. 잠재적 도전과 대응

| 도전 | 대응 |
|------|------|
| Capacitor 플러그인 타입 변경이 잦음 | `src/core/types/index.ts`에서 자체 인터페이스 정의, Capacitor 공식 타입은 devDependency로만 참조 |
| Auth 병렬 테스트 시 토큰 재발급 race condition | `auth.setup.ts` globalSetup에서 1회만 인증 수행, `storageState`로 공유 |
| 스냅샷 기준 환경 로컬 강제 불가 | CI에서만 스냅샷 커밋, PR 리뷰에서 로컬 생성 스냅샷 거부 프로세스 가이드 명시 |
| pre-commit hook 배포 | `package.json`의 `prepare` 스크립트에 `husky install` 등록, `npm install` 시 자동 설치 |
