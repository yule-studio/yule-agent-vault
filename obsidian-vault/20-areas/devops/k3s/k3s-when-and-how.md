---
title: "k3s 적용 가이드 — 시나리오 / 효율 패턴 / 마이그레이션 임계"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-18T11:30:00+09:00
tags: [devops, k3s, decision, mental-models]
home_hub: devops
related:
  - "[[k3s]]"
  - "[[concepts]]"
  - "[[ha-mode]]"
  - "[[production-checklist]]"
  - "[[../kubernetes/managed-vs-self-hosted]]"
---

# k3s 적용 가이드 — 시나리오 / 효율 패턴 / 마이그레이션 임계

**[[k3s|↑ k3s]]**

---

## 1. 목적

본 문서는 k3s 를 vanilla k8s / managed k8s (EKS / GKE / AKS) 와 비교해 **어떤 조건에서 적합하고 어떤 조건에서 부적합한지** 를 정의한다.

본 문서가 정의하는 것:
- k3s 의 적용 시나리오 분류
- 시나리오 별 권장 토폴로지 / 옵션 / 비활성화 항목
- 효율적 운영을 위한 구성 결정의 근거
- k3s → vanilla k8s / managed k8s 마이그레이션 임계 조건

본 문서가 정의하지 않는 것:
- 설치 절차 — [[installation]], [[practice/01-single-node-bootstrap]]
- 컴포넌트 깊이 — [[concepts]]
- 백업 / 업그레이드 절차 — [[backup-restore]], [[upgrade-strategy]]

---

## 2. 범위

| 구분 | 포함 |
| --- | --- |
| 대상 버전 | k3s v1.28 이상 (kine + embedded etcd 옵션 모두 지원) |
| 비교 대상 | vanilla kubeadm k8s, EKS, GKE Standard / Autopilot, AKS |
| 제외 | k0s, microk8s, kind, minikube ([[concepts]] 에 비교) |

---

## 3. 용어

| 용어 | 정의 |
| --- | --- |
| **server node** | k3s control plane + (옵션) worker. `--disable-agent` 시 control plane only. |
| **agent node** | worker only. |
| **kine** | etcd v3 gRPC API ↔ SQL DB / etcd adapter. |
| **embedded etcd** | k3s 가 내장한 HA etcd. 3 server 이상 권장. |
| **single-binary** | k3s 의 모든 컴포넌트 (apiserver / scheduler / controller-manager / kubelet / kube-proxy / containerd / flannel / klipper-lb / traefik / local-path) 가 한 binary 에 정적 link. |
| **default bundle** | k3s 가 기본 설치하는 옵션 컴포넌트 (Traefik / klipper-lb / local-path / metrics-server). 모두 `--disable` 가능. |

---

## 4. 시나리오 분류

본 문서는 k3s 의 적용 시나리오를 다음 6 가지로 분류한다.

### 4.1 적합 시나리오 (4)

| 시나리오 | 특성 | 권장 구성 |
| --- | --- | --- |
| **edge / IoT** | 단일 노드 또는 2-3 노드, 간헐적 네트워크, RAM ≤ 2GB | 1 server, SQLite, flannel host-gw |
| **on-premise small** | 5-20 노드, 사내망, RAM 충분, ops 인력 제한 | 3 server HA (embedded etcd) + N agent |
| **dev / CI cluster** | 일시적, 빠른 부트스트랩, 격리 필요 | k3d (Docker in Docker), `--disable` 전부 |
| **branch office / 매장** | 분산 다수, 원격 관리, 단일 노드 다대 | 1 server per site, GitOps (Fleet / Argo) |

### 4.2 부적합 시나리오 (2)

| 시나리오 | 부적합 사유 | 대안 |
| --- | --- | --- |
| **대규모 (100+ node, 5000+ pod)** | SQLite 한계, kine 성능 저하, single-binary 의 customization 제약 | vanilla k8s + 외부 etcd 또는 managed |
| **cloud-native HA (multi-AZ, multi-region)** | k3s 의 in-tree cloud provider 부재, 표준 EKS/GKE feature (managed control plane, IRSA, GKE Autopilot) 부재 | managed k8s |

---

## 5. 시나리오 × 토폴로지 결정 매트릭스

| 시나리오 | server 수 | data store | CNI / backend | 비활성화 권장 | 외부 LB |
| --- | --- | --- | --- | --- | --- |
| edge 단일 | 1 | SQLite | flannel host-gw | (default 유지) | (불필요) |
| edge 분산 | 1 × N site | SQLite | flannel wireguard-native | traefik (자체 ingress 미사용 시) | (불필요) |
| on-premise small | 3 server HA + N agent | embedded etcd | flannel vxlan 또는 Calico | servicelb (외부 LB 사용 시) | MetalLB / HAProxy |
| dev / CI | 1 (k3d) | SQLite | flannel | traefik / servicelb / local-path 전부 | (불필요) |
| branch office | 1 per site | SQLite | flannel | (default 유지) | (불필요) |

---

## 6. 효율적 구성의 근거

### 6.1 단일 노드의 자원 절약

| 항목 | default | 권장 (edge) | 효과 |
| --- | --- | --- | --- |
| flannel backend | vxlan | host-gw (같은 L2) | encap overhead 제거 |
| ingress | traefik 자동 설치 | `--disable=traefik` (필요 시 nginx-ingress) | RAM 약 100MB 절감 |
| local-path | 자동 설치 | 사용 시 유지, 미사용 시 `--disable=local-path` | controller pod 1 개 절감 |
| metrics-server | 자동 설치 | HPA 미사용 시 `--disable=metrics-server` | RAM 약 50MB 절감 |
| servicelb | 자동 설치 | 외부 LB 사용 시 `--disable=servicelb` | klipper-lb pod 제거 |

### 6.2 config.yaml 표준화

명령행 옵션 대신 `/etc/rancher/k3s/config.yaml` 사용을 표준으로 한다.

```yaml
# /etc/rancher/k3s/config.yaml
write-kubeconfig-mode: "0644"
tls-san:
  - k3s.example.com
  - 10.0.0.10
disable:
  - traefik
  - servicelb
node-label:
  - "role=worker"
  - "env=prod"
kubelet-arg:
  - "max-pods=110"
  - "system-reserved=cpu=500m,memory=512Mi"
  - "eviction-hard=memory.available<200Mi"
flannel-backend: "host-gw"
```

| 효과 | 결과 |
| --- | --- |
| systemd unit 의 ExecStart 안정 | 옵션 변경 시 unit 수정 불필요 |
| 재현성 | IaC / GitOps 로 관리 가능 |
| 옵션 audit | 단일 파일 grep 으로 변경 추적 |

### 6.3 token 관리

| token | 권한 | 권장 |
| --- | --- | --- |
| server-token (`/var/lib/rancher/k3s/server/token`) | server 가입 + agent 가입 | server 가입에만 사용. 60일 주기 rotation. |
| agent-token (`/var/lib/rancher/k3s/server/agent-token`) | agent 가입 only | agent 가입에 사용. 별도 rotation. |
| node-token (auto) | 노드별 유일 join | 가능 한 모든 경우 권장. |

→ token 도난은 cluster 전체 침투로 직결되므로 secret store (Vault / sealed-secrets / cloud KMS) 에서 주입.

### 6.4 HA 모드 결정

| 노드 수 | data store | 가용성 |
| --- | --- | --- |
| 1 server | SQLite | control plane 단일 장애점 |
| 3 server | embedded etcd | etcd quorum (1 노드 장애 허용) |
| 5 server | embedded etcd | 2 노드 장애 허용 |
| 3 server | external Postgres / MySQL | DB 자체 가용성에 의존 |

→ HA 가 필요하면 최소 3 server. 1 또는 2 server 의 SQLite + HA 는 성립하지 않는다.

상세: [[ha-mode]].

### 6.5 storage 결정

| 요구사항 | 선택 | 비고 |
| --- | --- | --- |
| 단일 노드 dev | local-path (default) | hostPath 기반 |
| 다중 노드 RWX | Longhorn / NFS CSI | Longhorn 권장 (k3s 친화) |
| 다중 노드 RWO | Longhorn / Ceph (Rook) | Longhorn 운영 단순 |
| 외부 storage 사용 | iSCSI CSI / cloud CSI | provider 별 |

→ 다중 노드에서 local-path 사용 시 pod 가 다른 노드로 reschedule 되면 데이터 접근 불가. operational level 에서 반드시 교체.

상세: [[storage]].

---

## 7. k3s 와 vanilla k8s 의 차이

| 항목 | vanilla k8s | k3s |
| --- | --- | --- |
| 배포 단위 | 6+ binary (apiserver / scheduler / controller / kubelet / kube-proxy / etcd / CNI) | 1 binary |
| data store | etcd | kine (SQLite / Postgres / MySQL / 내장 etcd) |
| CNI | 선택 (Calico / Cilium / Flannel) | flannel 내장 (옵션 교체 가능) |
| ingress | 선택 | traefik 내장 (옵션) |
| LB | 외부 또는 MetalLB | klipper-lb 내장 (옵션) |
| storage | 선택 | local-path 내장 (옵션) |
| metrics-server | 별도 설치 | 내장 (옵션) |
| in-tree cloud provider | 포함 | 제거 (--disable-cloud-controller) |
| 설치 시간 | 수십 분 (kubeadm) | 1 분 미만 (curl | sh) |
| memory footprint | control plane ≈ 1GB+ | server ≈ 250-500MB |
| 호환성 | 표준 conformance | 표준 conformance (CNCF certified) |

→ 옵션을 모두 `--disable` 한 k3s 는 single-binary 로 동작하는 vanilla k8s 와 거의 동등하다. 호환성 등급에서 차이 없음.

---

## 8. 마이그레이션 임계 조건

다음 조건 중 하나가 충족되면 k3s → vanilla k8s 또는 managed k8s 마이그레이션을 검토한다.

| 임계 | 신호 | 권장 행선지 |
| --- | --- | --- |
| 노드 수 ≥ 50 | api request latency p99 > 1s, scheduler queue 적체 | vanilla + 외부 etcd |
| pod 수 ≥ 5000 | kine query slow, controller-manager 의 cache resync 느림 | vanilla + 외부 etcd |
| 클러스터 수 ≥ 5 사이트 | 운영 부담, 통합 audit / RBAC 어려움 | Rancher / Fleet 또는 managed |
| 팀 multi-tenant 요구 | RBAC + NetworkPolicy + quota 의 운영 부담 | managed k8s + namespace as service |
| cloud feature 필요 | IRSA / Workload Identity / 매니지드 RDS 연동 | EKS / GKE / AKS |
| 24/7 운영 SLA | on-call / postmortem / control-plane SLA 요구 | managed (SLA 99.95% 보장) |
| 컴플라이언스 (SOC2 / HIPAA) | 정기 audit / supply chain 인증 요구 | managed (사전 인증) |

임계 조건이 충족되어도 단일 cluster 단위가 아닌 multi-cluster (Rancher / Fleet / Karmada) 로 해결 가능한 경우, k3s 유지하면서 통합 control plane 추가가 대안이 된다.

---

## 9. managed k8s 와의 비용 비교

| 항목 | k3s (self-hosted) | managed (EKS / GKE / AKS) |
| --- | --- | --- |
| control plane 비용 | $0 (자체 노드) | EKS $73/월/cluster, GKE Standard $73/월/cluster, AKS $0 (free tier) |
| worker 비용 | 동일 (compute 요금) | 동일 |
| upgrade 운영 | 수동 (절차 정의 필요) | 1-click 또는 자동 |
| on-call 부담 | 자체 (control plane 포함) | worker 만 |
| ecosystem (LB / DNS / cert) | 별도 구성 | 매니지드 통합 (ALB / Cloud DNS / ACM) |
| egress 비용 | 동일 | 동일 |
| 가시화 | 자체 모니터링 | CloudWatch / Cloud Monitoring 통합 |

→ control plane 비용 $73/월 보다 운영 인건비가 훨씬 크다. 인력 절감을 위해 managed 가 정당화되는 경우가 다수.

---

## 10. 흔한 운영 실패 모드

| 실패 | 원인 | 모델로부터의 설명 |
| --- | --- | --- |
| 1 server 의 SQLite 가 갑자기 느림 | 동시 write 폭주 (큰 deployment / job 생성) | §3 kine + SQLite 의 단일 writer 한계 |
| 3 server HA 가 quorum 상실 | 한 노드 만 down 인데 다른 server 의 etcd 도 timeout | embedded etcd 의 disk IO 요구 (SSD 권장) |
| local-path 사용 중 pod 가 다른 노드로 schedule 후 데이터 없음 | local-path 는 노드 binding | §6.5 다중 노드 RWX 필요 시 Longhorn |
| traefik 으로 ingress 만들었는데 외부 접근 불가 | servicelb (klipper-lb) 가 node 의 80/443 점유 충돌 | §6.1 default bundle 의 servicelb 가 host port 사용 |
| k3s upgrade 후 워크로드 재시작 폭주 | systemd unit restart 시 kubelet 도 재시작 | upgrade 시 `--drain` 후 `system-upgrade-controller` 사용 |
| agent 가 server 못 찾음 | server URL / tls-san 불일치 | §6.2 config.yaml 의 tls-san 누락 |
| token rotation 후 agent join 실패 | agent-token 만 변경했는데 노드들이 server-token 사용 중 | §6.3 token 별 권한 구분 미준수 |

---

## 11. 적용 결정 체크 항목

다음 항목이 모두 충족되면 k3s 적합, 1 개 이상 미충족 시 대안 검토.

| 항목 | 충족 여부 |
| --- | --- |
| 노드 수 < 50 |  |
| pod 수 < 5000 |  |
| 운영 인력이 control plane 의 upgrade / backup / restore 절차 수립 가능 |  |
| 단일 / multi-cluster 의 통합 GitOps (ArgoCD / Fleet) 구성 가능 |  |
| 외부 LB / DNS / cert provider 구성 책임 수용 |  |

미충족 시 §8 의 행선지 참고.

---

## 12. 참고

- [[k3s|↑ k3s]]
- [[concepts]]
- [[installation]]
- [[ha-mode]]
- [[storage]]
- [[networking]]
- [[upgrade-strategy]]
- [[backup-restore]]
- [[production-checklist]]
- [[pitfalls]]
- [[../kubernetes/managed-vs-self-hosted]]
- k3s 공식 docs — https://docs.k3s.io
- CNCF k3s conformance — https://www.cncf.io/certification/software-conformance/
