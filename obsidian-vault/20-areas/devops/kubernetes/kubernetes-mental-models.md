---
title: "Kubernetes mental models — control loop / declarative / etcd / scheduler"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-18T13:00:00+09:00
tags: [devops, kubernetes, mental-models]
home_hub: devops
related:
  - "[[kubernetes]]"
  - "[[concepts]]"
  - "[[operator-pattern]]"
  - "[[admission-webhook-deep]]"
  - "[[advanced-scheduling]]"
  - "[[../docker/docker-mental-models]]"
---

# Kubernetes mental models — control loop / declarative / etcd / scheduler

**[[kubernetes|↑ kubernetes]]**

---

## 1. 목적

본 문서는 Kubernetes 의 동작을 명령 사용법이 아니라 **분산 시스템의 reconciliation 모델** 로 정의한다.

본 문서가 정의하는 것:
- declarative API + control loop 의 동작 모델
- API server / etcd / controller / scheduler / kubelet 의 책임 분리
- watch / informer / cache 메커니즘
- scheduler 의 결정 절차
- 위 모델에서 파생되는 운영 현상 (drift / split-brain / leader election / resync)

본 문서가 정의하지 않는 것:
- 리소스 종류별 사용법 — [[concepts]], [[deployments]], [[services-networking]]
- 디버깅 명령 — [[debugging]], [[pitfalls]]
- helm / GitOps 도구 — [[helm]], [[gitops-argocd]]

---

## 2. 범위

| 구분 | 포함 |
| --- | --- |
| 대상 버전 | k8s v1.28 이상 |
| 대상 컴포넌트 | kube-apiserver, etcd, kube-controller-manager, kube-scheduler, kubelet, kube-proxy |
| 대상 패턴 | declarative API, watch/informer, leader election, operator |
| 제외 | 특정 CNI / CSI / ingress controller — 각 deep 노트 |

---

## 3. 용어

| 용어 | 정의 |
| --- | --- |
| **declarative API** | 사용자가 desired state 를 선언하고 시스템이 actual state 를 그쪽으로 수렴시키는 모델. |
| **reconciliation loop** | desired vs actual 차이를 읽고 보정 액션을 실행하는 controller 의 무한 루프. |
| **resource** | API 가 다루는 객체 종류 (Pod / Deployment / Service / Node 등). |
| **kind / apiVersion** | 한 resource 의 타입 식별자. |
| **CRD (Custom Resource Definition)** | 사용자 정의 resource 의 스키마 등록. |
| **operator** | CRD + 해당 reconcile loop 를 구현한 controller. |
| **informer** | API server 의 watch stream 을 cache 로 변환하는 client-side 컴포넌트. |
| **resync** | informer 가 정기적으로 cache 의 모든 객체에 대해 reconcile 을 재호출하는 주기. |
| **finalizer** | resource 가 삭제되기 전 실행되어야 하는 cleanup 의 식별자. |

---

## 4. 모델 1 — declarative API

### 4.1 정의

사용자는 **desired state** 를 manifest 로 선언하고 API server 에 등록한다. 시스템은 controller 들의 reconciliation 으로 **actual state** 를 desired 에 맞춘다.

```
[user] → manifest (desired) → [API server] → [etcd]
                                    ↑
                                    │ watch
                                    │
                              [controller]
                                    │
                              [actual state 보정]
                                    ↓
                              [kubelet / cloud / runtime]
```

### 4.2 imperative 와의 비교

| 항목 | imperative | declarative |
| --- | --- | --- |
| 표현 | "Pod 3개 만들어라" | "Pod 가 3개 있어야 한다" |
| 실패 처리 | 호출자가 재시도 | 시스템이 desired 로 수렴 |
| idempotency | 호출자 책임 | 시스템이 보장 |
| drift | 감지 책임 호출자 | controller 가 자동 보정 |
| 분산 환경 적합 | 낮음 | 높음 (transient 실패가 normal) |

### 4.3 이로부터 도출되는 행동

| 행동 | 결과 |
| --- | --- |
| `kubectl delete pod <name>` | Pod 삭제. 하지만 ReplicaSet 의 desired 가 3 이면 즉시 새 Pod 가 생성됨. |
| 노드 장애로 Pod 가 unreachable | NodeLifecycle controller 가 Pod 를 다른 노드로 재스케줄. |
| ConfigMap 변경 | Pod 의 env 가 자동 갱신되지 않음 (Deployment 의 template hash 변경이 트리거). |
| etcd 에 직접 쓰기 | controller 가 즉시 다시 reconcile — etcd 직접 수정 금지. |

---

## 5. 모델 2 — control plane 컴포넌트 책임

### 5.1 컴포넌트 그래프

```
                ┌────────────┐
                │  kubectl   │
                │  (client)  │
                └──────┬─────┘
                       │ HTTPS
                       ▼
       ┌─────────────────────────────┐
       │   kube-apiserver (REST)     │   ← 유일한 etcd 진입점
       └────┬───────┬──────────┬─────┘
            │       │          │
            │ watch │ watch    │ watch
            ▼       ▼          ▼
       ┌──────┐ ┌────────┐ ┌────────┐
       │ etcd │ │ kcm    │ │scheduler│
       └──────┘ │ (loops)│ └────────┘
                │  - rs  │
                │  - dep │
                │  - sa  │
                │  - ... │
                └────┬───┘
                     ▼
                 [node]
              ┌──────────┐
              │ kubelet  │ ← watch only own node 의 Pods
              │ kube-prx │
              └──────────┘
```

### 5.2 책임

| 컴포넌트 | 책임 |
| --- | --- |
| kube-apiserver | REST API / auth / admission / etcd 게이트 — **유일한 etcd writer** |
| etcd | 모든 cluster state 의 KV store. consistency 보장. |
| kube-controller-manager | 표준 controller (ReplicaSet / Deployment / Node / ServiceAccount / Endpoint / Job / ...) 의 reconcile loop. |
| kube-scheduler | unscheduled Pod 에 노드 할당 결정. |
| cloud-controller-manager | cloud provider 통합 (LB / volume / route). |
| kubelet | 노드의 Pod 실제 실행 / health check / status 보고. |
| kube-proxy | service 의 iptables / IPVS 규칙 관리. (Cilium 등이 대체 가능) |

→ 모든 컴포넌트는 **etcd 를 직접 보지 않는다**. apiserver 를 watch.

---

## 6. 모델 3 — watch / informer / cache

### 6.1 정의

apiserver 는 client 의 `watch` 요청에 대해 변경 event 를 stream 으로 push 한다. controller / kubelet 은 informer 라는 client-side 컴포넌트로 watch stream 을 cache 로 변환해 사용한다.

```
[controller]
    │
    ▼
informer (watch + cache)
    ├─ event: ADDED → queue
    ├─ event: UPDATED → queue
    └─ event: DELETED → queue
                          │
                          ▼
                     workqueue (rate-limited)
                          │
                          ▼
                     reconcile(key) loop
                          │
                          ▼
                     apiserver 호출 (status update / 자식 생성)
```

### 6.2 resync 주기

informer 는 watch 외에 정기적으로 (보통 10-30분) cache 의 모든 객체에 대해 reconcile 을 다시 호출한다. 목적:

| 목적 | 효과 |
| --- | --- |
| watch event 누락 보정 | 네트워크 단절로 event 가 사라져도 다음 resync 에서 정정 |
| controller 재시작 후 일관성 | restart 직후 한 번 전체 sync |
| 외부 의존 (cloud) 의 drift 보정 | LB / DNS 의 외부 변경 감지 |

### 6.3 이로부터 도출되는 행동

| 행동 | 결과 |
| --- | --- |
| controller 가 잠시 다운돼도 cluster 정상 동작 | informer 가 복귀 시 watch + resync 로 재동기 |
| 동일 event 가 여러 번 reconcile 호출 | rate limit + idempotency 필수 |
| `kubectl get` 응답이 빠름 | apiserver 의 watch cache hit |
| `kubectl get -A --watch` 가 controller 와 같은 메커니즘 | watch protocol 공통 |

---

## 7. 모델 4 — controller pattern + leader election

### 7.1 controller 의 표준 구조

모든 controller (표준 + operator) 는 다음 구조이다.

```python
# pseudo-code
for event in informer.events():
    workqueue.add(event.key)

while True:
    key = workqueue.get()
    desired = get_desired_state(key)        # from spec
    actual  = get_actual_state(key)         # from cluster
    diff    = compute_diff(desired, actual)
    if diff:
        apply(diff)                          # apiserver 호출
    workqueue.done(key)
```

### 7.2 leader election

controller 는 보통 HA 로 여러 replica 가 실행되지만 reconcile 은 1 replica 만 수행한다. 이를 위해 leader election 사용.

| 메커니즘 | 위치 |
| --- | --- |
| ConfigMap / Lease lock | `kube-system` namespace 의 Lease (`kube-controller-manager`, `kube-scheduler`) |
| holder identity | leader replica 의 식별자 |
| TTL | leader 가 갱신 못하면 다른 replica 가 인계 |

```bash
kubectl get lease -n kube-system
kubectl get lease -n kube-system kube-controller-manager -o yaml
```

### 7.3 이로부터 도출되는 행동

| 행동 | 결과 |
| --- | --- |
| controller pod 3개 실행 시 1개만 일함 | leader 만 reconcile, 나머지는 standby |
| leader 교체 시 짧은 reconcile 공백 | TTL 만큼 (보통 15s) |
| operator 도 leader election 필요 | controller-runtime / kubebuilder 의 옵션 |
| split-brain 가능성 | TTL 짧고 etcd 일관성에 의존 — 정상 동작 시 미발생 |

---

## 8. 모델 5 — scheduler

### 8.1 결정 절차

scheduler 는 unscheduled Pod (spec.nodeName == "") 를 watch 하고 다음 2 phase 로 노드를 결정한다.

| phase | 동작 |
| --- | --- |
| filter | 각 plugin 이 노드를 후보에서 제외 (NodeName / TaintToleration / NodeAffinity / Resources / VolumeBinding / ...) |
| score | 남은 후보에 점수 부여 (NodeResourcesFit / ImageLocality / InterPodAffinity / TopologySpread / ...) |

최종 노드는 score 최댓값. 동점 시 무작위.

### 8.2 주요 filter / score plugin

| plugin | filter | score |
| --- | --- | --- |
| NodeName / NodeAffinity | ✓ | ✓ |
| TaintToleration | ✓ | ✓ |
| NodeResourcesFit | ✓ (request 충족 여부) | ✓ (가용 자원 균형) |
| VolumeBinding | ✓ (storage class topology) | |
| ImageLocality | | ✓ (이미 image 있는 노드 가산) |
| InterPodAffinity | ✓ | ✓ |
| PodTopologySpread | ✓ | ✓ |

### 8.3 결정 이후

scheduler 는 Pod 의 `spec.nodeName` 만 set 한다. 해당 노드의 kubelet 이 watch 로 감지하고 컨테이너 실행.

### 8.4 이로부터 도출되는 행동

| 행동 | 결과 |
| --- | --- |
| Pod 가 Pending 상태로 멈춤 | filter phase 에서 모든 노드 제외 — `kubectl describe pod` 의 events 확인 |
| HPA 가 scale-up 했는데 Pod schedule 안 됨 | 모든 노드의 가용 request 부족 — cluster autoscaler 필요 |
| 같은 노드에 같은 Pod 가 몰림 | TopologySpread 미설정 |
| 노드 자원은 남았는데 OOM | request 와 limit 의 비율 (`overcommit`) + scheduler 는 request 만 본다 |
| spot/preemptible 노드 사용 시 자주 reschedule | TaintToleration + PriorityClass 결합 |

---

## 9. 모델 6 — operator + CRD

### 9.1 operator 의 정의

operator = CRD + 해당 reconcile loop.

```
CRD (사용자 정의 resource 스키마)
  + Controller (해당 resource 의 reconcile loop)
  = Operator
```

예: PostgreSQL operator 는 `PostgresCluster` CRD 와 그 reconcile loop (StatefulSet + Service + ConfigMap + backup CronJob 생성) 로 구성.

### 9.2 작성 도구

| 도구 | 특징 |
| --- | --- |
| controller-runtime (Go) | 표준 — kubebuilder, operator-sdk 의 기반 |
| kubebuilder | CRD scaffold + reconciler 템플릿 |
| operator-sdk | kubebuilder + Helm/Ansible 모드 |
| Kopf (Python) | Python 으로 가볍게 |
| Metacontroller | webhook 으로 controller 외주 |

상세: [[operator-pattern]].

### 9.3 finalizer

resource 가 삭제 요청을 받았을 때 외부 정리가 필요하면 finalizer 등록. finalizer 가 모두 제거될 때까지 etcd 에서 실제 삭제되지 않음.

```yaml
metadata:
  finalizers:
    - mycorp.io/cleanup-cloud-resource
```

삭제 흐름:

| 단계 | 동작 |
| --- | --- |
| 1 | `kubectl delete` → `deletionTimestamp` 설정 |
| 2 | controller 가 finalizer 보고 cleanup (LB 해제 / DNS record 삭제 / volume detach) |
| 3 | cleanup 성공 시 controller 가 finalizer 제거 |
| 4 | finalizer 0 개 → etcd 에서 삭제 |

→ finalizer 가 남아있으면 resource 가 영원히 `Terminating` 상태. operator 미배포 / bug 시 발생.

---

## 10. 모델 7 — admission control

### 10.1 정의

API server 가 etcd 에 쓰기 전 통과해야 하는 검증 / 변경 단계.

```
client → authentication → authorization → mutating admission → validation → schema → mutating admission (built-in)
                                                                                          → validating admission
                                                                                                → persistence (etcd)
```

### 10.2 admission 종류

| 종류 | 시점 | 예 |
| --- | --- | --- |
| built-in mutating | spec 자동 보강 | ServiceAccount 의 token 주입 |
| built-in validating | 표준 정책 검증 | resource quota / PodSecurity |
| mutating webhook | 외부 서비스 변경 | Istio sidecar 주입, OPA Gatekeeper 의 변경 |
| validating webhook | 외부 서비스 검증 | image scan 통과 여부, label 강제 |

### 10.3 이로부터 도출되는 행동

| 행동 | 결과 |
| --- | --- |
| webhook 서비스 죽으면 apiserver 가 모든 쓰기 reject | `failurePolicy: Fail` 의 효과 |
| webhook 가 namespace 제한 안 두면 kube-system 까지 영향 | `namespaceSelector` 필수 |
| Pod 의 sidecar 가 자동 주입 | mutating webhook 의 효과 |
| OPA / Kyverno 의 policy 가 적용됨 | validating webhook |

상세: [[admission-webhook-deep]].

---

## 11. 모델 간 상호작용 — Pod 생성의 end-to-end

다음은 `kubectl apply -f deployment.yaml` 후 실제 Pod 가 노드에서 실행되기까지의 전체 흐름이다.

```
1. kubectl → apiserver (REST POST)
2. apiserver
   ├─ authentication / authorization
   ├─ mutating admission (built-in + webhook)
   ├─ validating admission
   └─ etcd write (Deployment resource)
3. Deployment controller (kcm)
   ├─ watch event: ADDED Deployment
   ├─ reconcile: create ReplicaSet (etcd write)
4. ReplicaSet controller (kcm)
   ├─ watch event: ADDED ReplicaSet
   ├─ reconcile: create N Pods (etcd write)
5. scheduler
   ├─ watch event: ADDED Pod (spec.nodeName == "")
   ├─ filter + score → node 선택
   └─ apiserver: PATCH Pod (spec.nodeName = "node-X") (etcd write)
6. kubelet (on node-X)
   ├─ watch event: ADDED Pod with spec.nodeName == "node-X"
   ├─ CRI: containerd 에게 sandbox + container 생성 요청
   ├─ CNI plugin: pod IP 할당, namespace 구성
   ├─ CSI plugin: volume mount
   ├─ container start
   └─ apiserver: PATCH Pod status (Running)
7. Endpoint controller (kcm)
   ├─ watch event: Pod Ready
   └─ Endpoint/EndpointSlice 갱신 (etcd write)
8. kube-proxy (on all nodes)
   ├─ watch event: EndpointSlice 변경
   └─ iptables / IPVS 규칙 갱신
9. (optional) ExternalDNS / ingress controller
   └─ external DNS / LB 갱신
```

→ 각 단계는 watch-based 비동기이며 controller 간 직접 호출 없음. 전체가 apiserver + etcd 를 중심으로 한 reconcile graph.

---

## 12. 흔한 실패 모드

| 실패 | 원인 | 모델로부터의 설명 |
| --- | --- | --- |
| `kubectl delete pod` 후 즉시 같은 Pod 가 다시 생김 | ReplicaSet 의 desired 가 그대로 | §4.3 |
| etcd 직접 수정 후 cluster 이상 | controller 가 즉시 다시 reconcile | §4.3 |
| controller 가 무한 reconcile 루프 | spec 과 status 가 서로를 변경 | §6.3 idempotency 위반 |
| resource 가 `Terminating` 영원히 남음 | finalizer 미제거 | §9.3 |
| mutating webhook 죽고 cluster 가 쓰기 못함 | `failurePolicy: Fail` | §10.3 |
| Pod 가 `Pending` 계속 | scheduler filter 모두 fail | §8.4 |
| ConfigMap 변경했는데 Pod 갱신 안 됨 | Deployment 의 template hash 미변경 | §4.3 |
| HPA scale-up 했는데 Pod 못 띄움 | 노드 가용 request 부족 | §8.4 |
| controller pod 3 개인데 1 개만 일함 | leader election 정상 동작 | §7.3 |
| operator restart 후 5-10분 멈춤 | informer initial list + cache 채우기 | §6.2 |
| webhook 가 cluster 전체 영향 | namespaceSelector 미설정 | §10.3 |

---

## 13. 참고

- [[kubernetes|↑ kubernetes]]
- [[concepts]]
- [[deployments]]
- [[services-networking]]
- [[ingress-controllers]]
- [[operator-pattern]]
- [[admission-webhook-deep]]
- [[advanced-scheduling]]
- [[debugging]]
- [[pitfalls]]
- [[../docker/docker-mental-models]]
- [[../linux/linux-mental-models-for-devops]]
- [[../../computer-science/network/topics/container-networking|↗ CS — container networking]] — pod 네트워크의 OS 기반
- [[../../computer-science/network/topics/service-discovery|↗ CS — service discovery]] — Service / kube-proxy / CoreDNS 의 위치
- [[../../computer-science/network/tools/iptables-netfilter|↗ CS — iptables / netfilter]] — kube-proxy iptables mode 의 실제 룰
- Kelsey Hightower — kubernetes-the-hard-way
- 공식 docs — https://kubernetes.io/docs/concepts/architecture/
- controller-runtime — https://github.com/kubernetes-sigs/controller-runtime
