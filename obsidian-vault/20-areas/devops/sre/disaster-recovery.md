---
title: "Disaster Recovery (DR) — RTO / RPO"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T06:28:00+09:00
tags: [devops, sre, dr, disaster-recovery]
---

# Disaster Recovery (DR) — RTO / RPO

**[[sre|↑ sre]]**

---

## 1. DR 이란

대규모 장애 (region 단위, 자연재해, 인프라 down) 시 **복구 절차**.

- 일반 incident response 와 다름 — region / DC 단위 영향.
- 사전에 정의된 RTO / RPO.

---

## 2. RTO / RPO

| | 정의 | 예 |
| --- | --- | --- |
| **RTO** (Recovery Time Objective) | 복구까지 허용 시간 | "1시간 안에 복구" |
| **RPO** (Recovery Point Objective) | 허용 데이터 손실 | "최근 5분 데이터 손실 OK" |

→ 둘 다 작을수록 비싸짐.

---

## 3. RTO/RPO 별 비용

| RTO/RPO | 전략 | 비용 |
| --- | --- | --- |
| 24h / 24h | offline backup, 수동 복구 | 저렴 |
| 4h / 1h | warm standby (작은 인프라) | 중간 |
| 1h / 5min | hot standby (replica) | 중상 |
| 1min / 0 | active-active multi-region | 매우 비쌈 |

→ business critical 영역 만 active-active.

---

## 4. 4 가지 DR 전략 (AWS)

| 전략 | RTO | RPO | 비용 |
| --- | --- | --- | --- |
| **Backup & Restore** | 시간 ~ 일 | 시간 | $ |
| **Pilot Light** | 분 ~ 시간 | 분 | $$ |
| **Warm Standby** | 분 | 초 | $$$ |
| **Multi-site Active/Active** | 0 | 0 | $$$$ |

---

## 5. Backup 표준 (3-2-1)

```
3 copies
2 different media
1 offsite

예:
- prod (NVMe SSD)
- nightly backup (S3 같은 region)
- weekly backup (S3 다른 region, cross-region replication)
```

---

## 6. DB DR

### 6-1. PostgreSQL

```bash
# 일일 logical backup
pg_dump --format=custom --compress=9 prod > backup-$(date +%Y%m%d).dump

# S3 업로드
aws s3 cp backup-*.dump s3://backups/postgres/

# 복구 검증 (★ 안 한 backup = 가치 0)
pg_restore --create --dbname=test backup-2026-05-12.dump
```

### 6-2. Streaming replication (hot standby)

```
master (primary) --[WAL stream]--> standby (read-only)
                                   ↑
                                   region B (또는 같은 region 다른 AZ)
```

→ failover 시 standby 를 promote.

### 6-3. PITR (Point-in-Time Recovery)

```
WAL archive + base backup → 임의 시점으로 복구.
```

→ "10분 전 까지 복구" 가능.

---

## 7. multi-region failover

```
Route53 health check → primary region down 감지
                    → DNS 변경 → secondary region 으로
```

```yaml
# Route53 record set
- type: A
  name: api.example.com
  routing: failover
  set-identifier: primary
  alias: alb-us-east-1.example.com
  health-check: us-east-1-health
- type: A
  name: api.example.com
  routing: failover
  set-identifier: secondary
  alias: alb-us-west-2.example.com
```

---

## 8. data replication

| 데이터 | 전략 |
| --- | --- |
| **DB** | streaming replication (cross-region read replica) |
| **object** | S3 cross-region replication |
| **cache** | 보통 안 함 (warm-up cost) |
| **search** | Elasticsearch cross-cluster replication |
| **secrets** | KMS / Secrets Manager multi-region |

---

## 9. DR drill (★)

```
quarterly:
- 시나리오 정의 ("us-east-1 down")
- 실제 failover 실행 (drill 또는 read traffic 만)
- RTO / RPO 측정
- 발견된 issue → action item
- 다음 drill 까지 fix
```

→ **drill 안 한 DR = 동작 안 함**.

---

## 10. DR runbook 예시

```markdown
# DR — us-east-1 region failure

## RTO: 30min / RPO: 5min

## Detection
- AlertManager: HighRegionUnavailability
- Route53 health check fails

## Decision (IC)
- 영향 < 30min 예상 → wait (failover 비용)
- 영향 > 30min OR critical → failover

## Failover (15min)

### Step 1 — DB promote
```bash
aws rds promote-read-replica \
    --db-instance-identifier prod-db-us-west-2
```

### Step 2 — DNS switch
```bash
aws route53 change-resource-record-sets \
    --hosted-zone-id Z123 \
    --change-batch file://failover.json
```

### Step 3 — verify
- curl api.example.com/health → us-west-2 응답?
- error rate < 1%?
- write 성공?

## Rollback
- primary region 복구 시: 데이터 sync → 역 failover

## Communication
- status page: yellow (failover in progress) → green
- internal: #incident-dr
```

---

## 11. 함정

1. **backup 검증 안 함** — 복구 시 못 함 (corrupt, 호환성).
2. **drill 안 함** — 실제 disaster 시 처음 시도 → fail.
3. **secrets / config 동기화 안 됨** — secondary region 에 credential 없음.
4. **RTO/RPO 정의 안 됨** — 우선순위 / 비용 결정 불가.
5. **standby capacity 부족** — failover 시 traffic 못 받음 (auto-scale 도 부족).
6. **cross-region cost 무시** — replication / egress 비용 큼.

---

## 12. 관련

- [[sre|↑ sre]]
- [[chaos-engineering]]
- [[runbook]]
- [[../cloud-aws/cloud-aws|↗ cloud-aws]]
