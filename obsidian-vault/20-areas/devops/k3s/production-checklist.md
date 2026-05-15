---
title: "k3s production checklist"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T09:17:00+09:00
tags: [devops, k3s, production, checklist]
---

# k3s production checklist

**[[k3s|↑ k3s]]**

---

## 1. cluster topology

```
☐ HA (3+ server) — single SPOF X
☐ server 에 NoExecute taint (CriticalAddonsOnly)
☐ N+1 agent capacity (1 node down 시 traffic 흡수 가능)
☐ multi-AZ (zone 별 서로 다른 server)
☐ external LB (HAProxy / kube-vip / NLB) for apiserver
☐ LB health check 설정
☐ DNS 설정 (k3s.example.com → LB)
☐ pod CIDR / service CIDR 회사 network 와 안 충돌
```

---

## 2. control plane

```
☐ embedded etcd 또는 external DB (HA 필요)
☐ etcd snapshot schedule + S3 upload
☐ snapshot retention 정책
☐ etcd disk = SSD (fsync latency)
☐ NTP 동기 (chrony / systemd-timesyncd)
☐ swap off (k8s 권장)
☐ kernel parameter 권장값 (net.ipv4.ip_forward=1 등)
☐ cgroup v2 (k8s 1.25+ 권장)
☐ kubelet system-reserved / kube-reserved 설정
```

---

## 3. networking

```
☐ CNI 선택 (flannel default OK / Calico / Cilium for policy)
☐ NetworkPolicy 활성 (default deny → 명시 allow)
☐ encryption (wireguard backend 또는 mesh mTLS)
☐ ingress controller 선택 (Traefik default / nginx)
☐ TLS cert (Let's Encrypt / cert-manager)
☐ DNS policy (CoreDNS upstream / NodeLocalDNS)
☐ external 외부 LB / firewall rule (6443, 10250, 8472)
☐ pod-to-pod latency (cross-AZ < 5ms)
```

---

## 4. storage

```
☐ storage class default 정의 (의도된)
☐ local-path 만 = production X (HA workload 면 Longhorn / cloud CSI)
☐ Longhorn 의 replica count = 3
☐ PV backup 정책 (Velero / Longhorn recurring)
☐ disk monitoring (>= 80% alert)
☐ inode 모니터링 (작은 파일 많을 때)
☐ open-iscsi 설치 (Longhorn 필요)
☐ reclaimPolicy 검토 (Delete vs Retain)
```

---

## 5. observability

```
☐ Prometheus + Grafana
☐ kube-state-metrics + node-exporter
☐ Loki + Promtail (log 집계)
☐ Tempo / Jaeger (trace)
☐ AlertManager + PagerDuty / Slack
☐ Grafana dashboard (Golden Signals)
☐ SLO 정의 + Error Budget
☐ k3s 자체 monitoring (etcd health / apiserver latency)
☐ retention 정책
```

---

## 6. security

```
☐ admin kubeconfig 권한 600 (root only)
☐ RBAC 활성 + least privilege
☐ NetworkPolicy default deny
☐ Pod Security Admission (restricted profile)
☐ image scanning (Trivy in CI)
☐ image signing + admission (cosign + Kyverno)
☐ secrets external (Vault / AWS SM / ESO)
☐ etcd encryption at rest
☐ audit log (k8s audit-policy)
☐ TLS for everything (mTLS service mesh 권장)
☐ 정기 cert rotation
☐ node OS patch 자동 (unattended-upgrades)
☐ 정기 token rotation
☐ firewall (host + cloud)
```

---

## 7. backup / DR

```
☐ etcd snapshot 6h 간격 + S3
☐ retention 30일+
☐ manifests / config 별도 backup
☐ Velero (workload + PV)
☐ DR drill 분기 1회
☐ RTO / RPO 정의
☐ DR runbook 문서화
☐ multi-region 또는 multi-cluster failover (critical)
```

---

## 8. CI/CD

```
☐ GitOps (ArgoCD / Fleet) 사용
☐ image build → scan → sign → push
☐ progressive delivery (canary)
☐ rollback 절차
☐ deploy frequency / MTTR 측정 (DORA)
☐ secrets in CI = OIDC token (access key X)
```

---

## 9. resource management

```
☐ resource requests / limits 명시
☐ HPA 활성 (CPU / memory / custom)
☐ VPA recommendation 모니터
☐ PDB (PodDisruptionBudget) for critical service
☐ priorityClass for critical pod
☐ node taint / toleration / affinity 잘 사용
☐ overcommit ratio 검토
```

---

## 10. application

```
☐ readiness / liveness / startup probe
☐ graceful shutdown (SIGTERM handle, preStop)
☐ terminationGracePeriodSeconds 합리적
☐ Pod 의 1 image (sidecar 외)
☐ rolling update strategy (maxSurge / maxUnavailable)
☐ image:tag 명시 (latest X)
☐ multi-arch image (RPi + AMD64)
☐ Helm / Kustomize 표준
```

---

## 11. upgrade strategy

```
☐ version 고정 (latest X)
☐ system-upgrade-controller (자동, canary first)
☐ minor 하나씩
☐ pre-upgrade etcd snapshot
☐ rollback runbook
☐ deprecated API 사전 점검 (pluto)
☐ off-peak time
```

---

## 12. runbook

```
☐ cluster bootstrap
☐ node add / remove
☐ disk fill 처리
☐ etcd restore (quorum 잃음)
☐ certificate renewal
☐ network partition 처리
☐ pod crash loop 분석
☐ OOM 분석
☐ traffic spike 대응
☐ DR failover
```

→ 모든 runbook 정기 drill.

---

## 13. team / process

```
☐ on-call rotation
☐ SLO / Error Budget Policy
☐ incident response 표준
☐ postmortem culture
☐ chaos engineering (분기 GameDay)
☐ documentation (TechDocs)
```

---

## 14. cost

```
☐ unused resource cleanup (정기)
☐ HPA / VPA 통한 right-sizing
☐ image registry storage 비용
☐ log retention 비용
☐ network egress 비용
☐ cluster autoscaler (cloud k3s)
```

---

## 15. 흔한 빠뜨리는 것

1. ❌ NoExecute taint on server — control plane 영향
2. ❌ external LB for apiserver — single server IP 사용
3. ❌ etcd snapshot S3 — server disk 잃으면 끝
4. ❌ NetworkPolicy default deny — flat network
5. ❌ Pod Security Admission — privileged pod 가능
6. ❌ image scan in CI — known CVE production
7. ❌ DR drill — 실제 disaster 시 처음 시도
8. ❌ readiness probe — traffic 너무 일찍
9. ❌ PDB — drain 시 service 끊김
10. ❌ resource limit — 한 pod 가 node 전부

---

## 16. 관련

- [[k3s|↑ k3s]]
- [[ha-mode]]
- [[backup-restore]]
- [[../sre/sre|↗ SRE]]
- [[../security-ops/security-ops|↗ security-ops]]
