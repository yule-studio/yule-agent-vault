---
title: "Kubernetes 실습 hub"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T04:04:00+09:00
tags: [devops, kubernetes, practice]
---

# Kubernetes 실습 hub

**[[../kubernetes|↑ k8s]]**

---

| # | 실습 | 시간 |
| --- | --- | --- |
| 1 | [[01-kind-cluster]] — 로컬 k8s 띄우기 | 30분 |
| 2 | [[02-deploy-spring-boot]] — Spring Boot 배포 + ingress | 1시간 |
| 3 | [[03-helm-chart]] — Helm chart 작성 | 1시간 |

---

## 준비

```bash
docker --version
kubectl version --client
helm version
kind --version             # 또는 minikube
```

설치:
- [https://kind.sigs.k8s.io](https://kind.sigs.k8s.io)
- [https://kubernetes.io/docs/tasks/tools/](https://kubernetes.io/docs/tasks/tools/)
- [https://helm.sh/docs/intro/install/](https://helm.sh/docs/intro/install/)

---

## 관련

- [[../kubernetes|↑ k8s]]
