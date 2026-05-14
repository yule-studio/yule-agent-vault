---
title: "Deployment — Blue-green / Canary / DB 마이그레이션"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T06:30:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - operations
  - deployment
  - rollback
---

# Deployment — Blue-green / Canary / DB 마이그레이션

**[[operations|↑ operations hub]]**

> "어떻게 배포하나 + 사고 시 어떻게 되돌리나" — 운영 사고의 80%가 배포 시점에 발생.

---

## 1. 본 vault 결정

- **Blue-green** 기본 (안정).
- **Canary** — 큰 변경 시 (UI / 가격).
- **DB 마이그레이션** = code 배포 전 별도 단계.
- **Forward-only** 마이그레이션 (롤백 시 새 V).

---

## 2. 배포 전략 — 4구조 비교

### 2.1 Blue-green

**왜 본 vault default**
- 단순 — 두 환경 (blue=현재, green=새) 만.
- 즉시 rollback — green 문제 시 blue 로 트래픽 복귀.
- staging 환경이 green 으로 활용.

**구현**
```
[ALB]
  → blue (현재 prod)   100%
  → green (새 버전)     0%
   ↓ 검증 후 (smoke test)
  → blue                0%
  → green             100%
   ↓ 24h 모니터링
  → blue 종료 (또는 다음 배포 시 swap)
```

**왜 안 됨 (큰 변경)**
- 0% → 100% 한 번에 = 사용자 영향 검증 X.
- 큰 UI 변경 / 가격 변경 시 부담.

---

### 2.2 Canary

**왜 적합**
- 점진 배포 — 1% → 10% → 50% → 100%.
- 각 단계에서 메트릭 모니터링 + 사고 시 stop.

**도구**
- AWS — App Mesh / ALB weighted target group.
- Kubernetes — Argo Rollouts / Flagger.
- 서비스 mesh — Istio / Linkerd.

**구현**
```yaml
# Argo Rollouts canary
spec:
  strategy:
    canary:
      steps:
        - setWeight: 1
        - pause: { duration: 10m }
        - setWeight: 10
        - pause: { duration: 30m }
        - setWeight: 50
        - pause: { duration: 1h }
        - setWeight: 100
```

**자동 분석**
- 각 단계에서 메트릭 비교 (canary vs stable).
- 5xx / latency / 비즈니스 지표 차이 > threshold = 자동 rollback.

---

### 2.3 Rolling update

**왜 X (본 vault)**
- 두 버전 동시 운영 — code + DB schema 호환 부담.
- Blue-green / Canary 가 더 명확.

**언제 적합**
- 무상태 / 작은 변경.
- 인프라 비용 절감 (blue-green 의 2배 자원 회피).

---

## 3. DB 마이그레이션 + Code 배포 순서

### 3.1 Expand / Contract 패턴

```
[V1: Expand]
  schema: 새 컬럼 ADD (nullable)
  code v1: 옛 컬럼 read + 새 컬럼 dual-write
  배포 → 안정

[V2: Backfill]
  schema: 옛 컬럼 → 새 컬럼 마이그레이션 (점진적)
  code: 동일
  배포 → 안정

[V3: Contract]
  code v2: 새 컬럼만 read/write
  배포 → 안정

[V4: Drop]
  schema: 옛 컬럼 DROP
```

**왜 4단계**
- 각 단계가 작은 변경 — rollback 안전.
- code 와 schema 호환 항상 유지.

**왜 한 번에 (배포 + ALTER) X**
- 큰 ALTER (1시간+) = 서비스 중단.
- rollback 어려움 (DROP 후 복구).

자세히: [[../database/migrations]].

---

## 4. Code 와 Schema 호환성

### 4.1 호환성 매트릭스

| | Code v1 | Code v2 |
| --- | --- | --- |
| Schema v1 | ✅ 현재 | ⚠️ v2 새 컬럼 X → fail 가능 |
| Schema v2 | ✅ v1 가 새 컬럼 무시 | ✅ |

→ **순서**: schema v2 먼저 (v1 가 무시), 그 다음 code v2.

### 4.2 흔한 함정

```
잘못된 순서:
  Day 1: Code v2 배포 — 새 컬럼 read 시도
  Day 2: Schema v2 배포 — 새 컬럼 추가

문제:
  Day 1 ~ Day 2 사이 — 새 컬럼 없는데 read 시도 → 모든 요청 fail
```

→ 항상 schema 먼저.

---

## 5. 자동 배포 (CI/CD)

### 5.1 GitHub Actions 예시

```yaml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { java-version: '21' }
      - run: ./mvnw clean package -DskipTests
      - run: docker build -t myrepo:${{ github.sha }} .
      - run: docker push myrepo:${{ github.sha }}

  test:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: ./mvnw test

  migrate-staging:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - run: flyway migrate -url=$STAGING_DB

  deploy-staging:
    needs: migrate-staging
    runs-on: ubuntu-latest
    steps:
      - run: aws ecs update-service ... --image myrepo:${{ github.sha }}

  smoke-test-staging:
    needs: deploy-staging
    runs-on: ubuntu-latest
    steps:
      - run: ./scripts/smoke-test.sh https://staging.example.com

  manual-approve-prod:
    needs: smoke-test-staging
    runs-on: ubuntu-latest
    steps:
      - uses: trstringer/manual-approval@v1

  migrate-prod:
    needs: manual-approve-prod
    runs-on: ubuntu-latest
    steps:
      - run: flyway migrate -url=$PROD_DB

  deploy-prod:
    needs: migrate-prod
    runs-on: ubuntu-latest
    steps:
      - run: aws ecs update-service ... --image myrepo:${{ github.sha }}
```

### 5.2 단계별 게이트

- staging 자동 배포 → smoke test 자동.
- prod = 사람 승인 + 수동 migrate.

---

## 6. Rollback

### 6.1 Code Rollback

**Blue-green**
```
green (v2) 에서 메트릭 비정상
  ↓
ALB weight: green 0%, blue 100% (즉시)
```

→ 수 초 내 복구.

**Canary**
- Argo Rollouts 자동 rollback (메트릭 기반).
- 또는 manual `argo rollouts abort`.

### 6.2 DB Schema Rollback — Forward-Only

```
V8__add_status_column.sql  (실수)
   ↓ prod 적용
   ↓ 문제 발견
   ↓ DROP COLUMN 의 V9 작성
V9__drop_status_column.sql
   ↓ prod 적용
```

**왜 V8 수정 X**
- Flyway checksum mismatch → 배포 멈춤.
- 모든 환경의 schema_history 깨짐.

자세히: [[../database/migrations#5 Rollback]].

### 6.3 Code + Schema 호환 사고 시

```
사고:
  Code v2 + Schema v1 = code v2 가 새 컬럼 read 시도 → fail

대응:
  1. Code rollback → v1 (즉시)
  2. Schema 분석 — v2 가 적용됐는지 확인
  3. 필요시 schema rollback (V9 추가)
```

---

## 7. 첫 30분 모니터링

```
배포 직후 30분 — 핵심 메트릭:
  - 5xx 비율
  - p95 latency
  - signup / login 성공 rate
  - 이메일 outbox 큐 길이
  - JVM heap / GC
```

자세히: [[runbook#3 배포 직후]].

---

## 8. 큰 변경의 Feature Flag 전략

```
Day 1: Code v2 배포 (flag=OFF)
  - 새 로직 dormant
  - 옛 로직 그대로
  - 안정성 검증

Day 5: flag=ON for 1% user
  - 메트릭 모니터링

Day 10: flag=ON for 100%
  - 옛 로직 제거 후 Code v3
```

**왜**
- 배포와 활성화를 분리 — 사고 시 코드 배포 없이 flag OFF.
- 점진 노출.

---

## 9. 함정 모음

### 함정 1 — 한 번에 (배포 + 큰 ALTER)
서비스 중단.
→ expand/contract.

### 함정 2 — Code 먼저 + Schema 나중
새 컬럼 없는데 read = fail.
→ schema 먼저.

### 함정 3 — Blue-green 의 DB 공유 불호환
v1 / v2 가 같은 schema 의존 — 한쪽 깨짐.
→ 호환성 보장.

### 함정 4 — DB 마이그레이션 자동 prod
실수 SQL 즉시 적용.
→ 사람 승인.

### 함정 5 — Smoke test 없음
배포 후 검증 X — 사고 발견 늦음.
→ smoke test 필수.

### 함정 6 — Rollback 절차 없음
사고 시 우왕좌왕.
→ runbook + 정기 drill.

### 함정 7 — Schema rollback 시 V 수정
checksum 깨짐.
→ forward-only.

### 함정 8 — Canary 의 메트릭 비교 없음
설정만 점진 — 사고 시 자동 stop X.
→ Argo Rollouts analysis template.

### 함정 9 — Feature flag 영구 잔존
코드 복잡도.
→ cleanup 정책.

### 함정 10 — 배포 시간이 트래픽 피크
사고 시 영향 극대화.
→ off-peak (새벽 2-4시) 또는 점진.

### 함정 11 — Health check 가 단순 200
앱 시작 직후라 DB 연결 X 인데 200.
→ deep health check (DB ping 등).

### 함정 12 — 옛 인스턴스 즉시 종료
in-flight 요청 cut.
→ graceful shutdown (Spring `lifecycle.timeout-per-shutdown-phase`).

---

## 10. 관련

- [[operations|↑ operations hub]]
- [[../database/migrations]] — Flyway 정책
- [[runbook]] — 배포 직후 / 사고 대응
- [[observability]] — 메트릭 / 알람
