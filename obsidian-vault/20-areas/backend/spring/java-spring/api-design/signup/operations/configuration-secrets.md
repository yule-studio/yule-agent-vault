---
title: "Configuration & Secrets — env / KMS / Vault"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T06:20:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - operations
  - secrets
  - config
---

# Configuration & Secrets — env / KMS / Vault

**[[operations|↑ operations hub]]**

> "어떻게 환경변수 / secret 관리하나" — 잘못 관리 시 **git leak / process list 노출 / DB 비번 평문 유출**.

---

## 1. 본 vault 결정

- **AWS Secrets Manager** (1순위) 또는 **HashiCorp Vault** (2순위).
- **환경변수 직접 X** (process list / heap dump 유출).
- **코드에 hardcoded 절대 X**.
- **dev / staging / prod 별도 secret**.

---

## 2. 환경변수 종류

### 2.1 비-secret (config)

| 변수 | 의미 | 환경별 |
| --- | --- | --- |
| `SPRING_PROFILES_ACTIVE` | profile | dev / staging / prod |
| `CORS_ORIGINS` | 허용 origin (csv) | 환경 별 |
| `SERVER_PORT` | 포트 | 보통 8080 |
| `LOG_LEVEL` | 로그 레벨 | dev=DEBUG, prod=INFO |

### 2.2 Secret

| 변수 | 의미 | 보관 |
| --- | --- | --- |
| `SPRING_DATASOURCE_PASSWORD` | DB 비번 | KMS |
| `AUTH_JWT_SECRET` | JWT signing 256-bit | KMS |
| `AWS_SECRET_ACCESS_KEY` | AWS IAM | IAM Role 권장 (key 없이) |
| `NCP_SENS_SECRET` | NCP API | KMS |
| `APPLE_PRIVATE_KEY` | Apple Sign In | KMS |
| `KAKAO_CLIENT_SECRET` / `NAVER_CLIENT_SECRET` | OAuth | KMS |

---

## 3. AWS Secrets Manager 통합

### 3.1 Dependency

```kotlin
implementation("org.springframework.cloud:spring-cloud-starter-aws-secrets-manager-config:2.5.0")
```

### 3.2 application.yml

```yaml
spring:
  config:
    import: "aws-secretsmanager:yule-studio/prod/db,yule-studio/prod/jwt"

spring:
  datasource:
    password: ${db-password}                    # secret 안의 key

auth:
  jwt:
    secret: ${jwt-secret}
```

### 3.3 IAM 권한

```json
{
  "Effect": "Allow",
  "Action": ["secretsmanager:GetSecretValue"],
  "Resource": ["arn:aws:secretsmanager:ap-northeast-2:*:secret:yule-studio/prod/*"]
}
```

### 3.4 왜 IAM Role (Access Key 없이)

- Access Key 가 secret = 보관 부담.
- EC2 / ECS / Lambda 가 IAM Role 자동 — key 노출 0.

---

## 4. 환경별 분리

### 4.1 Secret 네이밍

```
yule-studio/dev/db
yule-studio/dev/jwt
yule-studio/staging/db
yule-studio/staging/jwt
yule-studio/prod/db
yule-studio/prod/jwt
```

### 4.2 왜 분리

- prod secret 이 dev 로 leak 가능 (개발자 access).
- 사고 시 영향 범위 ↓.
- 환경별 정책 (rotation 주기) 다를 수 있음.

### 4.3 application.yml

```yaml
# application.yml (공통)
spring:
  config:
    activate:
      on-profile: prod
    import: "aws-secretsmanager:yule-studio/${spring.profiles.active}/*"
```

---

## 5. Secret Rotation

### 5.1 Rotation 주기

| Secret | 주기 | 자동/수동 |
| --- | --- | --- |
| DB password | 90일 | AWS Secrets Manager 자동 |
| JWT signing key | 180일 | 수동 (재배포 동반) |
| AWS Access Key | 90일 | IAM Role 권장 (rotation 불필요) |
| OAuth Client Secret | 1년 | 수동 (provider 와 동기) |
| TLS 인증서 | 1년 | ACM 자동 |

### 5.2 JWT key rotation 흐름

```
1. 새 JWT secret 생성 (256-bit)
2. application.yml — old + new 둘 다 secret 사용
   - 발급: new
   - 검증: old + new (둘 다 시도)
3. 옛 access token 만료 (15분) 후 old secret 제거
4. 모든 인스턴스 재배포
```

### 5.3 왜 graceful rotation

- 한 번에 secret 변경 시 → 옛 JWT 검증 실패 → 사용자 강제 로그아웃.
- 옛 + 새 둘 다 검증 = 무중단 rotation.

---

## 6. 코드 / git 보호

### 6.1 .gitignore

```
.env
.env.local
.env.*.local
application-local.yml
secrets/
*.pem
*.key
```

### 6.2 git-secrets (pre-commit)

```bash
brew install git-secrets
git secrets --install
git secrets --register-aws
```

→ commit 전 AWS key / secret 패턴 자동 감지.

### 6.3 GitHub Secret Scanning

- public repo = 자동 활성.
- private repo = Advanced Security (옵션).

### 6.4 사고 대응 — secret git leak

```
1. 즉시 secret rotate (새 발급)
2. git history 에서 제거 (BFG Repo-Cleaner)
3. 영향 분석 — 누가 access 했는지 (CloudTrail)
4. 모니터링 (의심 활동)
5. PIPC 신고 검토 (사용자 PII 영향 시)
```

---

## 7. Spring Boot 의 config 우선순위

```
1. command line argument
2. environment variable
3. application-{profile}.yml
4. application.yml
5. default
```

**왜 이 순서**
- 운영 시 environment variable 가 가장 강력 — emergency override.
- application.yml 은 기본값.

---

## 8. Feature Flag

### 8.1 왜 사용

- 점진 배포 — 1% → 10% → 100%.
- 즉시 비활성 — 사고 시 코드 배포 없이 끄기.
- A/B 테스트.

### 8.2 도구

| 도구 | 비용 | 통합 |
| --- | --- | --- |
| LaunchDarkly | 비싸 | Spring 통합 우수 |
| Unleash | 무료 (self-hosted) | OSS |
| Flagsmith | 무료 (limited) | OSS / cloud |
| 자체 구현 | 무료 | DB / Redis |

### 8.3 본 vault 권장

- 단순 — 자체 구현 (DB row).
- 복잡 — Unleash.

```java
@Service
public class FeatureFlagService {
    public boolean isEnabled(String flag, UserId userId) {
        // DB / Redis lookup
    }
}

@PostMapping("/signup")
public ResponseEntity<?> signup(...) {
    if (!flag.isEnabled("new-signup-flow", userId))
        return oldSignupFlow.handle(...);
    return newSignupFlow.handle(...);
}
```

---

## 9. dotenv 같은 dev 도구

### 9.1 .env (dev only)

```
# .env (gitignore)
DB_PASSWORD=dev_password
JWT_SECRET=dev-only-secret-32-bytes-min
```

```yaml
spring:
  config:
    import:
      - optional:file:.env[.properties]
```

### 9.2 왜 dev 만

- prod 의 .env = git leak 위험.
- prod = Secrets Manager.

---

## 10. 함정 모음

### 함정 1 — 코드에 hardcoded secret
git history 영구 노출.
→ Secrets Manager / Vault.

### 함정 2 — 환경변수 평문
process list / heap dump 노출.
→ KMS / Secrets Manager.

### 함정 3 — 모든 환경에 같은 secret
prod = dev secret 으로 가능.
→ 환경별 분리.

### 함정 4 — Rotation 없음
영구 secret = leak 시 영구 사고.
→ 주기 + 자동 rotation.

### 함정 5 — JWT rotation 한 번에
옛 token 즉시 invalid → 강제 logout.
→ graceful (old + new 검증).

### 함정 6 — IAM Access Key 사용 (Role 가능한데)
key 노출 risk.
→ IAM Role (EC2 / ECS / Lambda).

### 함정 7 — .env 가 git 에
public repo = 즉시 사고.
→ .gitignore + git-secrets pre-commit.

### 함정 8 — Heap dump 가 secret 포함
JVM heap dump (OOM) 분석 시 secret 노출.
→ prod heap dump 제한 + 즉시 폐기.

### 함정 9 — Spring config 우선순위 헷갈림
"왜 내 yml 안 먹어?" → env var 가 override.
→ 우선순위 명시.

### 함정 10 — Feature flag 영구 잔존
"임시" flag 가 1년+ → 코드 복잡도 ↑.
→ flag 도 lifecycle (배포 후 30일 cleanup).

### 함정 11 — Secret leak 사고 시 rotation 만
영향 분석 / 모니터링 없음 → 도용 발견 X.
→ CloudTrail + 의심 활동 분석.

### 함정 12 — Config 변경에 재배포 강제
긴급 변경 어려움.
→ Spring Cloud Config / Consul (runtime refresh).

---

## 11. 관련

- [[operations|↑ operations hub]]
- [[../design-decisions/dependencies]] — 라이브러리
- [[../security/sensitive-data-handling]] — secret 처리
- 외부 — AWS Secrets Manager Docs, HashiCorp Vault
