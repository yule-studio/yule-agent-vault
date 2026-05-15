---
title: "Secrets management — Vault / SM / SOPS / ESO ★"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T08:02:00+09:00
tags: [devops, security-ops, secrets, vault]
---

# Secrets management — Vault / SM / SOPS / ESO ★

**[[security-ops|↑ security-ops]]**

---

## 1. 왜

### 왜 필요
- 코드 / config 에 비밀 (DB password, API key) 두면 git history 영구 노출.
- 환경별 (dev/stage/prod) 다른 값.
- rotation 자동화.

### 안 하면 문제
- git 에 .env 누출 → 해커 즉시 사용.
- 직원 퇴사 시 모든 비밀 rotation 어려움.
- 어디서 어느 비밀 쓰는지 모름 → audit 불가.

### 대안
- 환경 변수 — 누가 보는지 X / 기록 X.
- HashiCorp Vault 등 dedicated.
- managed (AWS Secrets Manager / Azure Key Vault / GCP Secret Manager).

### 트레이드오프
- Vault 자체 운영 부담.
- 의존성 (Vault down = service down) → HA 필수.

---

## 2. 도구

| | type | 강점 |
| --- | --- | --- |
| **HashiCorp Vault** | self-host / cloud | 가장 강력, 다기능, dynamic secret |
| **AWS Secrets Manager** | SaaS | AWS 통합, KMS, 자동 rotation |
| **GCP Secret Manager** | SaaS | GCP 통합 |
| **Azure Key Vault** | SaaS | Azure 통합, HSM |
| **SOPS** (Mozilla) | git 친화 | encrypted yaml 을 git 에 commit |
| **External Secrets Operator** (ESO) | k8s | 외부 secret store ↔ k8s Secret 동기화 |
| **Sealed Secrets** (Bitnami) | k8s GitOps | git 에 commit 가능 (encrypted) |
| **Doppler / Infisical** | SaaS | dev-friendly UI |
| **1Password Secrets Automation** | SaaS | team 관리 친화 |

---

## 3. HashiCorp Vault 기본

```bash
# 설치 (간단)
vault server -dev

# unseal
vault operator init
vault operator unseal <key>

# 인증 (AppRole, token, k8s, AWS IAM 등)
vault auth enable approle
vault write auth/approle/role/my-app token_policies="my-policy"

# secret 저장 (KV v2)
vault kv put secret/myapp/db password=supersecret
vault kv get secret/myapp/db

# dynamic secret (DB credential 자동 생성)
vault write database/config/postgres \
    plugin_name=postgresql-database-plugin \
    connection_url="postgresql://{{username}}:{{password}}@db:5432/postgres" \
    allowed_roles="readonly"

vault write database/roles/readonly \
    db_name=postgres \
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
    default_ttl="1h" max_ttl="24h"

vault read database/creds/readonly
# → 1시간만 유효한 DB 계정 생성 (자동 회수)
```

→ **dynamic secret = static credential 의 종말**. ★

---

## 4. AWS Secrets Manager

```bash
# 저장
aws secretsmanager create-secret \
    --name prod/db/password \
    --secret-string '{"username":"admin","password":"supersecret"}'

# 조회 (Spring Cloud AWS / SDK)
aws secretsmanager get-secret-value --secret-id prod/db/password

# 자동 rotation (Lambda)
aws secretsmanager rotate-secret \
    --secret-id prod/db/password \
    --rotation-lambda-arn arn:aws:lambda:...:function:rotate \
    --rotation-rules AutomaticallyAfterDays=30
```

```yaml
# Spring (spring-cloud-aws-starter-secrets-manager)
spring:
  config:
    import:
      - "aws-secretsmanager:prod/db/password"
```

---

## 5. SOPS (git 친화)

```bash
# 파일 암호화 (AWS KMS / age / GPG)
sops --encrypt --kms arn:aws:kms:... config.yaml > config.enc.yaml

# 복호화 (런타임)
sops --decrypt config.enc.yaml > config.yaml

# git 에 config.enc.yaml 만 commit
```

```yaml
# .sops.yaml — 자동 KMS 선택
creation_rules:
  - path_regex: secrets/.*\.yaml$
    kms: arn:aws:kms:...
  - path_regex: dev/.*\.yaml$
    age: age1abc...
```

→ **GitOps 친화** — 암호화된 secret 도 git 에.

---

## 6. External Secrets Operator (★ k8s)

```yaml
# SecretStore
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata: {name: aws-store}
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-northeast-2
      auth:
        jwt:
          serviceAccountRef: {name: external-secrets-sa}

---
# ExternalSecret — AWS SM 에서 k8s Secret 자동 생성
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata: {name: app-db}
spec:
  refreshInterval: 1h
  secretStoreRef: {name: aws-store, kind: SecretStore}
  target:
    name: app-db          # 만들어질 k8s Secret 이름
  data:
    - secretKey: password
      remoteRef: {key: prod/db/password, property: password}
```

```yaml
# Pod 가 일반 Secret 처럼 사용
env:
  - name: DB_PASSWORD
    valueFrom: {secretKeyRef: {name: app-db, key: password}}
```

→ Vault / AWS SM / Azure KV / GCP SM 어디든 동기화.

---

## 7. Sealed Secrets (k8s GitOps)

```bash
# 일반 Secret → SealedSecret 으로 암호화
kubectl create secret generic mysecret \
    --from-literal=password=x \
    --dry-run=client -o yaml | \
    kubeseal -o yaml > mysealedsecret.yaml

# git 에 commit OK (private key 만 cluster 안에)
git add mysealedsecret.yaml
```

→ 단순. cluster 마다 다른 cert. ESO 보다 가볍지만 dynamic secret 없음.

---

## 8. Java / Spring 통합

```kotlin
// 직접 SDK
val client = SecretsManagerClient.create()
val secret = client.getSecretValue {
    it.secretId("prod/db/password")
}.secretString()

// Spring Boot — spring-cloud-aws-starter
@Value("\${db.password}")
lateinit var dbPassword: String
```

→ application code 가 Vault / SM 직접 호출 (또는 ESO 로 k8s Secret 변환 → env var).

---

## 9. rotation 전략

```
1. dynamic — Vault 가 매 요청마다 새 credential (★ 최강)
2. scheduled — 30일마다 자동 (AWS SM)
3. event-based — 직원 퇴사 등
4. manual — 절대 피해야
```

→ rotation 가능하도록 application 코드 — credential 변경 감지 + reconnect.

---

## 10. 함정

1. **git 에 .env / secret 누락** — history 영구 노출. 즉시 rotate + git filter-repo.
2. **CI 로그에 echo** — secret 노출. masked 설정 + secret 자동 redact.
3. **environment variable** 그대로 — `ps` / docker inspect 에서 보임.
4. **rotation 어려운 코드** — credential 변경 감지 안 함.
5. **Vault HA 없음** — Vault down = service down.
6. **secret 권한 너무 광범위** — least privilege.
7. **audit log 없음** — 누가 어떤 secret 봤는지.
8. **암호화 key 자체 도난** — KMS / HSM 보호.

---

## 11. 관련

- [[security-ops|↑ security-ops]]
- [[iam-best-practices]]
- [[../kubernetes/configmaps-secrets|↗ k8s secrets]]
