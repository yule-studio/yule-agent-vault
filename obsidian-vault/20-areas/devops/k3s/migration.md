---
title: "k3s ↔ k8s migration"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T09:23:00+09:00
tags: [devops, k3s, migration]
---

# k3s ↔ k8s migration

**[[k3s|↑ k3s]]**

---

## 1. 왜 마이그레이션

```
k3s → k8s (EKS / GKE):
  - cluster 성장 (수백 pod+)
  - cloud managed service 사용
  - 회사 표준 변경
  - enterprise feature (advanced scheduling 등)

k8s → k3s:
  - 비용 절감
  - edge 진출
  - 작은 footprint 필요
  - vendor lock-in 회피
```

---

## 2. 두 방식

### A. live migration (★ workload 만)

```
1. 두 cluster 모두 운영
2. external service (DB / Redis / Kafka) 가 양쪽 cluster 접근
3. ingress / DNS weighted routing
4. 한 cluster 씩 traffic 옮김
5. 옛 cluster 비움 후 폐쇄
```

→ zero downtime. 복잡.

### B. cluster 자체 transform

```
A → A' (in-place)
   - etcd / kine state 그대로
   - control plane 교체
   - workload migration
```

→ 거의 불가능. **B 보다 A 권장**.

---

## 3. manifest 호환

```
99% 호환 (둘 다 k8s API).

확인 필요:
  - storageClass name 차이
    (local-path → gp3 / standard-rwo)
  - Ingress class
    (traefik default → nginx)
  - LoadBalancer (klipper-lb → ALB)
  - CSI driver 차이
  - secrets / configmap 위치
  - default storage class
  - PVC 의 storageClassName 명시
```

---

## 4. CSI / storage 마이그레이션

```
local-path (k3s) → gp3 (EKS):

1. velero backup snapshot from k3s
   (PV restore 도구 사용)

2. EKS 에 새 PVC (gp3) 생성

3. data 복사:
   - rsync (Pod 간)
   - DB dump + restore (RDS)
   - S3 sync

4. application 가리킴 변경
```

→ 가장 까다로운 부분. data 마이그레이션.

---

## 5. workload 옮기기 (★ Velero)

```bash
# k3s 에서 backup
velero backup create k3s-snapshot --include-namespaces=prod

# backup 을 S3 에 (양쪽 cluster 가 접근 가능)
velero backup describe k3s-snapshot

# EKS 에 Velero 설치
# 같은 S3 bucket 지정

# restore
velero restore create --from-backup k3s-snapshot
```

→ namespace / manifest / 일부 PV 까지.

→ image registry 도 양쪽 접근 가능해야 (또는 mirror).

---

## 6. ingress / DNS 전환

```
phase 1: k3s 만
  api.example.com → k3s LB

phase 2: 양쪽 운영
  api.example.com → DNS round-robin (k3s 90% + EKS 10%)

phase 3: 점진 ramp
  k3s 50% + EKS 50%
  k3s 10% + EKS 90%

phase 4: EKS 만
  api.example.com → EKS LB
  k3s 폐쇄
```

→ Route53 weighted routing 또는 ingress controller.

---

## 7. data layer 전략

| | 권장 |
| --- | --- |
| **stateless app** | 동시 양쪽 OK, ingress 로 traffic shift |
| **stateful (DB)** | external managed (RDS) — 양쪽 access |
| **cache (Redis)** | external (ElastiCache) — 양쪽 access |
| **queue (Kafka)** | external (MSK) — 양쪽 producer/consumer |
| **PV / 파일** | S3 + signed URL 으로 추상화 |

→ "external 화" 가 migration 의 열쇠.

---

## 8. specific cloud feature 적응

```
k3s → EKS 시:
  ☐ kube-system 의 image (kube-proxy 등) — EKS native 사용
  ☐ CNI (flannel → AWS VPC CNI)
  ☐ IngressController (Traefik → ALB Ingress / nginx)
  ☐ storage (local-path → EBS gp3 / EFS)
  ☐ LB (klipper-lb → AWS NLB)
  ☐ IAM (직접 credential → IRSA)
  ☐ secrets (k8s Secret → Secrets Manager + ESO)
  ☐ DNS (CoreDNS upstream)
  ☐ autoscaling (manual → cluster autoscaler / Karpenter)

EKS → k3s 시:
  ☐ IRSA 사용 안 함 — 직접 credential 또는 다른 IDP
  ☐ EBS → local-path / Longhorn
  ☐ ALB → klipper-lb / MetalLB
  ☐ AWS VPC CNI → flannel
```

---

## 9. 검증 (★)

```
☐ 새 cluster 의 모든 namespace 존재
☐ pod 모두 Running
☐ service / ingress 응답
☐ persistent data 일치
☐ secrets / configmap 정상
☐ monitoring (Prometheus) 메트릭 수집
☐ alert (AlertManager) 발화
☐ CI/CD pipeline target 변경
☐ backup 시스템 새 cluster 인식
☐ DR runbook 업데이트
```

---

## 10. rollback 계획

```
phase 별 rollback gate:
  phase 1 → 2: traffic 10% → 0% (DNS revert)
  phase 2 → 3: traffic 50% → 0% 또는 90% 로 빠르게 후퇴
  phase 3 → 4: EKS 90% 에서 안정성 검증 → 폐쇄 결정
  최종 폐쇄 후 1주 후 cluster 삭제 (안 짠전 안전망)

DR snapshot:
  마이그 직전 양쪽 cluster snapshot 보관
```

---

## 11. cost / risk 비교

```
factor                 k3s          k8s managed (EKS)
─────────────────────────────────────────────────────
control plane $        0            $73/mo per cluster
worker node $          서버비       서버비 + EKS markup
management            self          AWS
HA setup              manual        자동
upgrade               manual / 자동  AWS-managed
SLA                   self          99.95%
multi-AZ              직접           native
audit / compliance     manual        SOC2 / HIPAA 등 included
```

---

## 12. 함정

1. **k3s 의 default storageClass 가 prod 와 다름** — manifest 검토.
2. **klipper-lb hostPort vs ALB DNS** — Service type 다름.
3. **Traefik annotation → nginx annotation** — 호환 X.
4. **secret 평문 git 커밋** — 마이그 사이 누출.
5. **DR drill 안 함** — 새 cluster 의 backup 검증.
6. **DNS TTL 길게** — phase 전환 늦음.
7. **양쪽 cluster 가 같은 DB write** — 경합 / 데이터 손상.

---

## 13. 관련

- [[k3s|↑ k3s]]
- [[backup-restore]]
- [[../kubernetes/managed-vs-self-hosted|↗ managed vs self-hosted]]
- [[../cloud-aws/cloud-aws|↗ AWS EKS]]
