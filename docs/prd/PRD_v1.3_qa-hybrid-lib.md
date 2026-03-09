# 하이브리드 앱 QA 자동화 표준 프레임워크
**PRD v1.3 최종 실행 명세** | `qa-hybrid-lib` | Phase 1: Scaffolding Template

---

## 0. 문서 목적 및 사용 방법

이 문서는 Claude Code에게 프로젝트 전체 맥락을 한 번에 전달하기 위한 기술 명세서입니다. 단순 아이디어 문서가 아니며, Claude Code가 코드 생성 전 반드시 숙지해야 할 구조적 제약과 기술적 결정사항을 포함합니다.

> **Claude Code 시작 프롬프트 (v1.3 최종)**
>
> "이 PRD v1.3은 `qa-hybrid-lib` 레포지토리 구축을 위한 최종 명세야.
> 1. 먼저 이 문서 전체를 숙지해.
> 2. 코드 생성 전, 네가 판단하기에 명세가 부족하거나 내가 결정해줘야 할 기술적 세부사항이 있다면 먼저 리스트업해서 물어봐.
> 3. 내 확인이 끝나면 표준 폴더 구조부터 생성을 시작하자.
> 4. TypeScript strict mode 준수, `any` 사용 금지, `page.waitForTimeout()` 사용 금지."

---

## 1. 개요 (Overview)

### 1.1 목적

PWA + Capacitor 조합의 하이브리드 앱 개발 시, 반복되는 QA 구축 비용을 절감하고 테스트 신뢰성을 확보하기 위한 표준화된 자동화 환경 구축.

### 1.2 핵심 가치

- **재사용성:** 신규 앱 프로젝트 도입 시 설정 시간 10분 이내 단축
- **일관성:** 모든 앱에서 동일한 품질 측정 지표와 테스트 시나리오 구조 유지
- **확장성:** MLOps 파이프라인(CI/CD)과의 즉각적인 통합

### 1.3 레포지토리 명 및 배포 전략

**레포지토리명:** `qa-hybrid-lib`

- **현재 전략:** 단순 복사(Scaffolding) 방식 — npm publish, Git Submodule 없이 신규 프로젝트에 `src/core` 폴더를 직접 복사하여 시작
- **npm 패키지화 전환 조건:** `src/core` 파일들이 3개 이상의 실제 프로젝트에서 6개월 이상 수정 없이 유지될 때 검토
- **이유:** 초기 단계에서 잘못된 추상화는 나중에 더 큰 재앙. 먼저 검증하고 그다음 자동화

---

## 2. 프로젝트 구조 (Project Architecture)

### 2.1 표준 폴더 구조

Claude Code가 프로젝트 생성 시 최우선으로 준수해야 할 구조입니다.

```
qa-hybrid-lib/
├── .github/
│   └── workflows/            # CI/CD (GitHub Actions)
├── src/
│   ├── core/                 # ⚠️ Immutable Layer — 앱 로직 의존 금지
│   │   ├── capacitor-mock.ts # Capacitor Bridge 인터셉터
│   │   ├── auth-handler.ts   # API 기반 세션 주입
│   │   └── base-config.ts    # 환경 변수 주입 인터페이스
│   ├── recipes/              # Mutable Layer — 앱별 시나리오 예시
│   │   └── example-app/
│   ├── page-objects/         # POM 패턴 기반 UI 캡슐화
│   ├── fixtures/             # 테스트 데이터 (JSON, Mock Data)
│   └── specs/                # 실제 테스트 시나리오 (.spec.ts)
├── tests/
│   └── snapshots/            # 시각적 회귀 테스트 기준 이미지 (.png)
├── playwright.config.ts      # 멀티 프로젝트 설정
├── .env.example              # 필수 환경 변수 가이드
└── package.json              # Node.js >=18, Playwright v1.42+
```

### 2.2 레이어 분리 원칙

| 레이어 | 경로 | 원칙 | 변경 빈도 |
|--------|------|------|-----------|
| Core (Immutable) | `src/core/` | 앱 비즈니스 로직 의존 금지. 모든 설정은 외부 주입 | 거의 없음 |
| Recipes (Mutable) | `src/recipes/` | core를 호출/상속하는 방식으로만 작성. 앱별 시나리오 | 프로젝트마다 다름 |
| Page Objects | `src/page-objects/` | POM 패턴 준수. UI 변경 시 이 레이어만 수정 | UI 변경 시 |
| Specs | `src/specs/` | 태그 기반 실행 제어. 비즈니스 시나리오 단위로 구성 | 기능 추가 시 |

---

## 3. 기술 스택 및 버전 명세

| 항목 | 선택 | 버전 / 비고 |
|------|------|-------------|
| 언어 | TypeScript | Strict Mode: ON / `any` 사용 엄격 금지 |
| 테스트 러너 | Playwright | v1.42+ (안정성 및 최신 브라우저 지원) |
| 런타임 | Node.js | `>=18` (`package.json` engines 필드 명시) |
| 네이티브 검증 | Appium | 선택적 — 복잡한 네이티브 기능 필요 시만 도입 |
| CI/CD | GitHub Actions | Ubuntu(기본) / macOS(`@ios` 태그 선택 실행) |
| 리포팅 | Playwright HTML + Allure | Trace Zip은 Artifact로 저장 |

---

## 4. 핵심 구현 상세 (Technical Decisions)

### 4.1 Capacitor Mocking 전략

- **방식:** `page.addInitScript()`를 활용한 `window.Capacitor.Plugins` 오버라이드
- **이유:** 네트워크 레벨 인터셉트보다 JS 브릿지 레벨에서 가로채는 것이 Capacitor 앱 동작을 가장 정확하게 시뮬레이션

호출 인터페이스 (설정 주입 방식):

```typescript
// core는 외부 설정값에만 의존 — Hard-coding 금지
CapacitorMock.setup(page, {
  plugins: ['Camera', 'Geolocation', 'Storage']
});
```

### 4.2 Auth Handler (고속 로그인)

- **기본 방식:** API를 통한 토큰 획득 후 LocalStorage/Cookie 주입 — UI 로그인 플로우 매번 타지 않음
- **전체 세션 공유:** `playwright/auth.setup.ts` (globalSetup)에서 1회만 실제 UI 로그인 후 상태를 `playwright/.auth/user.json`에 저장
- 모든 테스트 프로젝트에서 `storageState`로 이 파일을 참조

### 4.3 Playwright 멀티 프로젝트 구성 (playwright.config.ts)

```typescript
projects: [
  { name: 'web',        use: { ...devices['Desktop Chrome'] } },
  { name: 'mobile-pwa', use: { ...devices['Pixel 5'] } },
  { name: 'mobile-ios', use: { ...devices['iPhone 13'] } }, // @ios 태그 시만 실행
  { name: 'cap-mock',   use: { /* capacitor-mock.ts 주입 */ } },
],
retries: process.env.CI ? 2 : 0,  // CI에서만 2회 재시도
```

### 4.4 시각적 회귀 테스트

- **도구:** Playwright 내장 `expect(page).toHaveScreenshot()` — 외부 서비스 불필요
- **임계값:** `maxDiffPixels` 설정으로 미세 렌더링 차이에 의한 Flaky 방지
- **스냅샷 저장:** `tests/snapshots/` 폴더에 `.png` 파일 커밋하여 관리
- **업데이트 방법:** UI 변경이 의도된 경우 로컬에서 `npx playwright test --update-snapshots` 실행 후 PR에 포함
- **CI 제약:** CI 환경에서 기준 스냅샷이 없을 경우 자동 실패 처리

---

## 5. 환경 변수 규칙 (Environment Config)

| Key 이름 | 타입 | 값 예시 | 설명 |
|----------|------|---------|------|
| `BASE_URL` | string | `https://app.example.com` | 테스트 대상 앱 URL |
| `API_URL` | string | `https://api.example.com` | 인증 및 API 호출 엔드포인트 |
| `TEST_PLATFORM` | string | `web \| android \| ios` | 실행 플랫폼 스위칭 |
| `MOCK_NATIVE` | boolean | `true \| false` | Capacitor 브릿지 모킹 활성화 여부 |

`.env.example` 파일을 레포에 포함하여 신규 프로젝트 도입 시 필수 변수 가이드 제공. 실제 값은 `.gitignore` 처리.

---

## 6. CI/CD 파이프라인 전략

### 6.1 태그 — 트리거 — Runner 매핑

| 트리거 (Event) | 실행 범위 (Tag) | 실행 프로젝트 (Project) | Runner |
|----------------|----------------|------------------------|--------|
| Pull Request | `@smoke` | Web (Desktop Chrome) | Linux (ubuntu-latest) |
| Main Merge | `@regression` (전체) | Web + PWA (Pixel 5) | Linux (ubuntu-latest) |
| Manual / Schedule | `@native` | Web + PWA + iOS | macOS (**비용 유의**) |

> **macOS Runner 비용 주의:** GitHub Actions macOS Runner는 Linux 대비 약 10배 비쌈. `@ios` 태그가 붙은 테스트에 한해 선택적으로 실행.

### 6.2 Android 네이티브 CI 전략

> ⚠️ **Android 에뮬레이터 CI 제약사항**
>
> - Android AVD(에뮬레이터) 부팅은 3~5분 소요 + Linux Runner에서 KVM 설정 별도 필요
> - Phase 1에서는 Android를 Headless 브라우저 에뮬레이션으로 제한
> - 실 디바이스 연동 및 Appium 기반 Android 네이티브 테스트는 Phase 2로 미룸
> - Claude Code는 Android native 구현 시 이 제약을 반드시 명시할 것

### 6.3 테스트 속도 최적화

- GitHub Actions sharding 사용: 테스트를 N개 병렬 Job으로 분산 실행
- 향후 MLOps 파이프라인 확장 고려 구조로 설계

---

## 7. Claude Code 코딩 컨벤션 (Ground Rules)

### 7.1 필수 준수 사항

| 항목 | 규칙 |
|------|------|
| TypeScript | Strict Mode ON — `tsconfig.json`에 `strict: true` 명시 |
| 타입 금지 | `any` 타입 사용 엄격 금지. 모든 Mock 데이터는 Interface 정의 필수 |
| 대기 함수 | `page.waitForTimeout()` 사용 금지 — `waitForSelector`, `waitForLoadState` 등 상태 기반 대기 사용 |
| Hard-coding | URL, 계정 정보 등 앱 특화 값은 core 코드에 직접 작성 금지 — 환경 변수 또는 주입 방식으로만 |
| 주석 언어 | 로직 설명은 한국어 작성, 함수명/변수명은 영문 표준 |
| 불명확한 명세 | 구현 전 반드시 질문 먼저. 임의로 결정하지 않음 |

### 7.2 테스트 태그 규칙

- `@smoke` — PR 단계 기본 실행. 핵심 플로우만 포함, 빠른 피드백
- `@regression` — Main 머지 시 전체 실행. 전체 시나리오 포함
- `@native` — Capacitor 플러그인 관련 테스트. macOS Runner 선택 실행
- `@ios` — iOS 에뮬레이터 필요 테스트. macOS Runner만 실행

---

## 8. 성공 지표 (Success Metrics)

| 지표 | 기준 | 측정 시점 |
|------|------|-----------|
| QA 자동화 초기 셋업 시간 | 신규 프로젝트 도입 1시간 이내 | Phase 1 완료 후 2~3번째 프로젝트 |
| 공통 모듈 재사용률 | `src/core` 사용 비중 50% 이상 | 3개 프로젝트 적용 후 |
| npm 패키지화 전환 조건 | core 파일 3개 프로젝트 × 6개월 무수정 | 지속 모니터링 |
| 치명적 버그 사전 차단율 | 자동화 단계에서 90% 이상 감지 | `@regression` 기준 |

---

## 9. 버전 이력 (Revision History)

| 버전 | 주요 변경사항 |
|------|--------------|
| v1.0 | 초기 PRD 작성 — 목표, 기술 스택, 기능 요구사항 기본 정의 |
| v1.1 | 폴더 구조 추가, Capacitor Mocking 방식 명세, Auth 전략, 코딩 컨벤션 추가 |
| v1.2 | playwright.config.ts 멀티 프로젝트 구조, CI 태그-Runner 매핑, 스냅샷 관리 전략, Flaky 정책 추가 |
| v1.3 (최종) | 배포 전략 확정(단순 복사 우선), core/recipes 레이어 분리, Android CI 제약 명시, Node.js 버전 명세, npm 전환 조건 수치화 |
