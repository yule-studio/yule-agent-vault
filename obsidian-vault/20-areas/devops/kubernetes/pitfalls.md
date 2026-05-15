---
title: "Kubernetes pitfalls"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T04:02:00+09:00
tags: [devops, kubernetes, pitfalls]
---

# Kubernetes pitfalls

**[[kubernetes|↑ k8s]]**

---

1. **resources.requests 누락** → HPA 동작 X + 노드 packing 망가짐.
2. **resources.limits 없음** → 노드 OOM 한 Pod 가 다른 Pod 영향.
3. **liveness probe 가 너무 strict** → 정상 Pod 도 죽음.
4. **readiness probe 없음** → 시작 안 된 Pod 에 traffic.
5. **DB 를 Deployment 사용** (StatefulSet X) → Pod 이동 시 데이터 손실.
6. **bind mount + DB** → 권한 문제.
7. **PV reclaimPolicy=Delete** → 실수 PVC delete = 영구 손실.
8. **secret git commit** → sealed-secrets / ESO / SOPS.
9. **default SA 권한** → 모든 Pod 가 cluster API 접근.
10. **cluster-admin 남발** → 최소 권한.
11. **NetworkPolicy 없음** → Pod 간 자유 통신.
12. **Pod anti-affinity 없음** → 같은 노드 의 Pod 다 죽음.
13. **PodDisruptionBudget 없음** → drain 시 모두 죽음.
14. **autoscaler 없는 1 replica** → 노드 fail 시 down.
15. **latest tag** — 재현 X.
16. **CRD 누락** → Helm install 실패.
17. **logs 없음** → kubectl logs 빈 결과 → stdout/stderr 로 write.
18. **stuck termination** (PID 1 못 받음) → graceful shutdown.
19. **DNS pod 부족** → cluster DNS resolution timeout.
20. **control plane 업그레이드 후 자동 노드 업그레이드 X** → skew.

---

## 관련

- [[kubernetes|↑ k8s]]
- [[deployments]]
- [[concepts]]
