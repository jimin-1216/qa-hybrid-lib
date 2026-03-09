# PRD Analysis Report

## 분석 대상
- **문서**: `docs/PRD_v1.3_qa-hybrid-lib.md`
- **분석일**: 2026-03-09
- **분석 도구**: WIGTN prd-reviewer (4-category parallel analysis)

## 요약

| 카테고리 | 발견 | Critical | Major | Minor |
|----------|------|----------|-------|-------|
| 완전성 | 5 | 1 | 3 | 1 |
| 실현가능성 | 3 | 0 | 2 | 1 |
| 보안 | 3 | 1 | 1 | 1 |
| 일관성 | 3 | 0 | 1 | 2 |
| **총계** | **14** | **2** | **7** | **5** |

## Quality Gate: BLOCKED (Critical 2개)

---

## Critical (즉시 수정 필요)

### C-1. 에러 처리 전략 완전 부재
- **위치**: 전체 문서
- **문제**: 테스트 실패 시 재시도 정책은 CI에만 `retries: 2`가 있고, 테스트 코드 레벨의 에러 핸들링이 전혀 정의되지 않음. Capacitor Mock이 실패하면? Auth 토큰 만료되면? 네트워크 타임아웃이면?
- **영향**: 개발자마다 다른 방식으로 에러를 처리 → core 레이어의 "표준화" 가치 훼손
- **개선안**: `src/core/` 각 모듈별 에러 시나리오 + 처리 전략 정의 필요
  ```
  CapacitorMock 실패 → 테스트 skip + 경고 로그
  Auth 토큰 만료 → 자동 재발급 후 retry
  네트워크 타임아웃 → configurable timeout + 명확한 에러 메시지
  ```

### C-2. 테스트 데이터(fixtures) 보안 정책 누락
- **위치**: Section 2.1 `src/fixtures/`, Section 5
- **문제**: fixtures에 Mock 데이터를 넣는다고 했지만, 실제 계정 정보/API 키가 fixture에 들어갈 위험에 대한 가드가 없음. `.env.example`은 있지만 fixture 파일 내 민감 데이터 규칙 없음
- **영향**: 실수로 실제 인증 토큰이 fixture JSON에 포함되어 Git에 커밋될 위험
- **개선안**:
  - fixture 파일 내 민감 데이터 금지 규칙 명시
  - `fixtures/` 디렉토리에 대한 git pre-commit hook으로 시크릿 스캔
  - Mock 데이터 인터페이스에 `MOCK_` 접두사 컨벤션

---

## Major (구현 전 수정 권장)

### M-1. 비기능 요구사항(NFR) 수치 부재
- **위치**: Section 8 (성공 지표)
- **문제**: 성공 지표는 있지만 성능 NFR이 없음. 테스트 실행 시간 목표, 병렬 실행 시 리소스 제한, CI 파이프라인 전체 소요 시간 등
- **영향**: "느려도 통과하면 OK"가 되어 CI 파이프라인이 점점 느려질 위험
- **개선안**:
  ```
  @smoke: 5분 이내, @regression: 20분 이내, 단일 spec: 30초 이내
  ```

### M-2. Page Object 패턴 규칙 불충분
- **위치**: Section 2.2
- **문제**: POM 패턴 준수라고만 되어있고, 구체적인 구조 규칙이 없음. Base Page Object 인터페이스, 네이밍 컨벤션, selector 전략(data-testid vs CSS vs XPath) 미정의
- **영향**: 개발자마다 다른 POM 구현 → "일관성" 핵심 가치 훼손
- **개선안**: Base Page Object 인터페이스 + selector 전략 우선순위 명시 (예: `data-testid` > `role` > CSS)

### M-3. Spec 파일 구조 규칙 미정의
- **위치**: Section 2.1 `src/specs/`
- **문제**: 태그 기반 실행 제어는 있지만, spec 파일 네이밍 규칙, describe/it 구조, 1 spec당 시나리오 수 제한 등이 없음
- **영향**: spec이 커지면서 유지보수 어려움, 실행 시간 불균형
- **개선안**:
  ```
  네이밍: {도메인}.{시나리오}.spec.ts (예: auth.login.spec.ts)
  제한: 1 spec 파일당 최대 10 test case
  ```

### M-4. Appium 도입 기준 모호
- **위치**: Section 3
- **문제**: "복잡한 네이티브 기능 필요 시만 도입"이라 했지만, "복잡한"의 기준이 없음. 카메라? GPS? Push 알림? 어디까지가 Capacitor Mock으로 충분하고 어디부터 Appium인지?
- **영향**: Phase 2 진입 판단이 주관적 → 팀 내 의견 충돌
- **개선안**: Capacitor Mock 커버리지 한계 목록 작성
  ```
  Mock 가능: Storage, Geolocation, Camera (메타데이터)
  Mock 불가 → Appium: 실제 카메라 촬영, 생체인증, 딥링크
  ```

### M-5. 시각적 회귀 테스트 Cross-Browser 전략 부재
- **위치**: Section 4.4
- **문제**: 스냅샷이 OS/브라우저별로 다를 수 있는데 기준 환경이 정의되지 않음. Linux CI에서 찍은 스냅샷과 macOS 개발자 로컬이 달라 항상 fail 날 수 있음
- **영향**: Flaky 테스트 → 스냅샷 테스트 신뢰도 하락 → 팀이 무시하기 시작
- **개선안**: 스냅샷 기준 환경 고정 (예: CI Linux만 기준, 로컬은 `--update-snapshots` 금지)

---

## Minor (개선 제안)

### m-1. Scale Grade 미정의
- **위치**: 전체 문서
- **문제**: Scale Grade(Hobby/Startup/Growth/Enterprise)가 없음
- **개선안**: Phase 1 = Startup급, Phase 2 = Growth급 정도로 명시

### m-2. recipes 예시가 `example-app` 하나뿐
- **위치**: Section 2.1
- **문제**: 실제 활용을 보여줄 두 번째 recipe가 없어 "어떻게 확장하는지" 감이 안 옴
- **개선안**: 최소 2개 recipe (예: `example-ecommerce`, `example-dashboard`)

### m-3. 용어 불일치
- **위치**: 전체 문서
- **문제**: "Scaffolding Template"(제목) vs "단순 복사"(본문) vs "레포지토리 구축"(Claude Code 프롬프트). 같은 개념에 3가지 표현
- **개선안**: 하나로 통일 (예: "Scaffolding 복사 방식")

### m-4. CI/CD에서 Allure 리포트 설정 상세 누락
- **위치**: Section 3, 6
- **문제**: Allure를 리포팅 도구로 언급했지만 CI에서 어떻게 생성/배포하는지 없음
- **개선안**: GitHub Pages 자동 배포 또는 Artifact 저장 방식 명시

### m-5. 성공 지표 측정 방법 부재
- **위치**: Section 8
- **문제**: "셋업 시간 1시간 이내"를 어떻게 측정하는지? 누가? 언제?
- **개선안**: 측정 프로토콜 추가 (예: "신규 팀원이 README만 보고 첫 테스트 실행까지 걸린 시간")

---

## 총평

**잘 된 점:**
- 레이어 분리 (Core/Recipes/POM/Specs)가 명확함
- "먼저 검증, 그 다음 자동화" 철학이 건강함
- Capacitor Mock 전략이 구체적
- npm 패키지화 조건이 수치화되어 있음 (3개 프로젝트 x 6개월)
- CI 비용 인식이 있음 (macOS Runner 10배)

**Critical 2개 수정 후 → `/implement` 진행 가능.**
