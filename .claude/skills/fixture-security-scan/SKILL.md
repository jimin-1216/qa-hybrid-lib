---
name: fixture-security-scan
description: 테스트 fixture 파일 보안 스캔. 실제 인증 정보, API 키, 토큰 등 민감 데이터 자동 감지. PRD C-2 해결.
allowed-tools: Read, Grep, Glob
---

# Fixture Security Scan

테스트 fixture 파일에서 실제 인증 정보, API 키, 토큰 등 민감 데이터를 자동으로 감지하는 보안 스캔 스킬.

## 1. 스캔 대상 범위

```
src/fixtures/**/*.json
src/fixtures/**/*.ts
src/recipes/**/fixtures/**
*.fixture.ts
*.fixture.json
.env* (except .env.example)
playwright/.auth/**
```

## 2. 감지 패턴 (Critical → Minor)

### Critical (즉시 차단)

| 패턴 | 정규식 예시 | 설명 |
|------|-----------|------|
| AWS 키 | `AKIA[0-9A-Z]{16}` | AWS Access Key ID |
| 실제 JWT 토큰 | `eyJ[A-Za-z0-9-_]{20,}\.eyJ[A-Za-z0-9-_]{20,}` | Base64 인코딩된 실제 JWT |
| Private Key | `-----BEGIN (RSA\|EC\|OPENSSH) PRIVATE KEY-----` | 개인 키 |
| 실제 비밀번호 패턴 | `password["':=\s]+[^$\{MOCK_].{8,}` | MOCK_ 접두사 없는 비밀번호 값 |
| API Key 패턴 | `(api[_-]?key\|apikey)["':=\s]+[a-zA-Z0-9]{20,}` | 긴 API 키 값 |
| GitHub Token | `gh[ps]_[A-Za-z0-9_]{36,}` | GitHub Personal/Service Token |
| Slack Token | `xox[bprs]-[A-Za-z0-9-]{10,}` | Slack API Token |

### Major (수정 권장)

| 패턴 | 설명 |
|------|------|
| 실제 이메일 주소 (회사 도메인) | `*@company.com` 같은 실제 조직 이메일 |
| 실제 전화번호 패턴 | `010-\d{4}-\d{4}` 한국 전화번호 |
| 실제 URL (내부 서버) | 내부 IP나 스테이징 URL |
| `.env` 파일이 gitignore에 없는 경우 | 환경 파일 노출 위험 |

### Minor (개선 권장)

| 패턴 | 설명 |
|------|------|
| MOCK_ 접두사 미사용 | Mock 데이터에 `MOCK_` 접두사 없음 |
| 하드코딩된 Bearer 토큰 | `Bearer` 뒤에 긴 문자열 |
| 주석 처리된 credential | 주석이라도 민감 데이터는 위험 |

## 3. 안전한 Mock 데이터 컨벤션

```typescript
// 올바른 패턴
interface MockUser {
  email: string;
  token: string;
}

const MOCK_USER: MockUser = {
  email: "MOCK_user@test.example.com",    // MOCK_ 접두사 + test 도메인
  token: "MOCK_jwt_token_for_testing",     // MOCK_ 접두사
};

const MOCK_API_KEY = "MOCK_api_key_12345"; // MOCK_ 접두사

// 위험한 패턴
const user = {
  email: "admin@company.com",              // 실제 이메일
  token: "eyJhbGciOiJIUzI1NiIs...",       // 실제 JWT
  password: "MyR3alP@ssw0rd!",            // 실제 비밀번호
};
```

## 4. playwright/.auth/ 보안

- `user.json` (storageState 파일)은 `.gitignore`에 반드시 포함
- CI에서는 환경 변수로 인증 처리
- 로컬 auth 파일에 실제 세션이 커밋되지 않도록 체크

## 5. .gitignore 검증

필수 포함 항목:
```
.env
.env.local
.env.*.local
playwright/.auth/
*.pem
*.key
```

## 6. 출력 포맷

```markdown
# Fixture Security Scan Report

## 스캔 요약
- **스캔일**: YYYY-MM-DD
- **스캔 파일 수**: N개
- **발견 이슈**: Critical X, Major Y, Minor Z

## 발견 사항

### Critical (즉시 수정)
| # | 파일 | 라인 | 패턴 | 설명 | 조치 |
|---|------|------|------|------|------|

### Major (수정 권장)
### Minor (개선 권장)

## .gitignore 검증
- [ ] .env 포함
- [ ] playwright/.auth/ 포함
- [ ] *.pem, *.key 포함

## 권장 조치
1. 발견된 credential 즉시 제거 + 키 로테이션
2. MOCK_ 접두사 컨벤션 적용
3. pre-commit hook 추가 (credential 스캔)
```

## 7. 자동화 권장사항

pre-commit hook 예시:
```bash
#!/bin/bash
# fixture 파일에서 민감 데이터 패턴 검색
if git diff --cached --name-only | grep -E "fixture|\.env" | xargs grep -lE "AKIA|eyJ[A-Za-z0-9]{20,}\.|-----BEGIN.*PRIVATE KEY"; then
  echo "민감 데이터가 포함된 파일이 감지되었습니다. 커밋을 중단합니다."
  exit 1
fi
```
