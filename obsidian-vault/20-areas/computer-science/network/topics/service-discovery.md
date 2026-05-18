---
title: "service discovery — DNS / sidecar / config 기반 패턴"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-18T15:00:00+09:00
tags: [network, topics, service-discovery, dns, microservices]
home_hub: network
related:
  - "[[topics]]"
  - "[[../network]]"
  - "[[../dns/dns]]"
  - "[[../load-balancing/load-balancing]]"
  - "[[container-networking]]"
  - "[[../../../devops/kubernetes/kubernetes-mental-models]]"
  - "[[../../../devops/networking-ops/service-mesh]]"
---

# service discovery — DNS / sidecar / config 기반 패턴

**[[topics|↑ topics]]**

---

## 1. 목적

본 문서는 분산 시스템에서 클라이언트가 서비스 인스턴스의 위치를 찾는 패턴 (service discovery) 을 정의한다.

본 문서가 정의하는 것:
- service discovery 가 필요한 이유 (동적 IP / 가변 인스턴스 수)
- 4 가지 구현 패턴 (정적 config / DNS / client-side registry / sidecar)
- 각 패턴의 트레이드오프 (지연 / 일관성 / 운영 부담)
- k8s / Consul / Eureka / Istio 가 각 패턴 중 어디에 해당하는지
- 운영 결정의 기준

본 문서가 정의하지 않는 것:
- 특정 도구의 사용법 — 각 도구 docs
- DNS 프로토콜 세부 — [[../dns/dns]]
- 서비스 mesh 의 mTLS / observability — [[../../../devops/networking-ops/service-mesh]]

---

## 2. 범위

| 구분 | 포함 |
| --- | --- |
| 대상 환경 | 분산 마이크로서비스, container orchestration |
| 대상 도구 | Kubernetes (CoreDNS / kube-proxy), Consul, etcd, ZooKeeper, Eureka, Istio, Linkerd, AWS Cloud Map |
| 제외 | monolith 의 hardcoded endpoint |
| 제외 | message broker (RabbitMQ / Kafka) 의 broker discovery — 별도 |

---

## 3. 용어

| 용어 | 정의 |
| --- | --- |
| **service** | 동일 기능을 제공하는 인스턴스 집합의 논리 이름. |
| **instance** | service 의 한 실행 단위 (process / container / pod). |
| **endpoint** | instance 의 (IP, port) 쌍. |
| **registry** | service 와 endpoint 의 매핑을 저장하는 데이터 저장소. |
| **registration** | instance 가 시작 시 registry 에 자신을 등록하는 행위. |
| **deregistration** | instance 가 종료 시 registry 에서 제거되는 행위. |
| **health check** | instance 의 가용성을 주기적으로 확인하는 절차. |
| **TTL** | registry entry 가 유효한 시간. 만료 시 자동 제거. |

---

## 4. 왜 필요한가

| 환경 | 상황 |
| --- | --- |
| static IaaS | 인스턴스 IP 가 변하지 않음 → hosts file 또는 정적 DNS 로 충분 |
| auto-scaling group | 인스턴스 수가 동적 → 시작 / 종료마다 endpoint 변화 |
| container orchestration | 인스턴스 수명이 짧고 IP 가 매번 새로 할당 → 정적 config 불가능 |
| multi-region | 사용자 위치에 따라 다른 region 의 endpoint 로 라우팅 |

→ 동적 환경에서 클라이언트가 endpoint 를 알아내는 메커니즘이 service discovery.

---

## 5. 4 가지 구현 패턴

### 5.1 패턴 A — 정적 config

```yaml
# 예: docker-compose
services:
  api:
    environment:
      DB_HOST: db.example.com
      DB_PORT: 5432
```

| 특성 | 평가 |
| --- | --- |
| 구현 복잡도 | 최저 |
| 변경 시 절차 | 수동 reload / restart |
| 인스턴스 추가 | 불가능 (또는 LB 뒤로) |
| 사용 | 단일 인스턴스 dev / static infrastructure |

### 5.2 패턴 B — DNS 기반

```
client → DNS query (api.example.com) → resolver → A/AAAA records → endpoint(s)
```

| 변형 | 특성 |
| --- | --- |
| **DNS round-robin** | 동일 이름 → 여러 A record → 클라이언트가 round-robin |
| **DNS SRV** | service + protocol + priority + weight + port + target — 가장 표현력 높음 |
| **headless service (k8s)** | ClusterIP 없이 pod IP 직접 응답 — stateful 워크로드 |
| **CoreDNS / k8s DNS** | `<svc>.<ns>.svc.cluster.local` 로 ClusterIP 응답 |
| **External DNS** | k8s Service / Ingress → 외부 DNS (Cloudflare / Route53) record 자동 동기 |

| 특성 | 평가 |
| --- | --- |
| 구현 복잡도 | 낮음 (표준 프로토콜) |
| 클라이언트 변경 | 불필요 (모든 OS 가 DNS resolver 보유) |
| 변경 전파 | TTL 만료까지 지연 (보통 30s ~ 5min) |
| health check | DNS 자체로는 부족 — 외부 health checker 와 결합 필요 |
| 부하 분산 | round-robin 만 (가중치는 SRV 또는 GeoDNS) |
| 사용 | k8s 내부 (CoreDNS), 단순 multi-instance |

### 5.3 패턴 C — client-side registry (lookup API)

```
[instance] → 시작 시 → registry (Consul / etcd / Eureka)
              ↘ 종료 시 → registry 제거

[client] → registry API 조회 → endpoint 목록 → client-side LB 로 선택
```

| 도구 | 특성 |
| --- | --- |
| **Consul** | KV + service registry + health check + DNS interface |
| **etcd** | 일반 KV (Raft 합의). k8s 내부에서도 사용 |
| **ZooKeeper** | 표준 분산 coordination, Kafka 등에서 사용 |
| **Eureka (Netflix)** | Spring Cloud 의 표준. eventually consistent. |
| **AWS Cloud Map** | AWS 매니지드 service registry |

| 특성 | 평가 |
| --- | --- |
| 구현 복잡도 | 중 (클라이언트가 registry 알아야) |
| 변경 전파 | 빠름 (push 또는 short poll) |
| health check | registry 가 주기적으로 check + TTL 기반 자동 deregister |
| 부하 분산 | client-side LB (Ribbon / gRPC client / Linkerd-proxy) |
| 일관성 | 도구에 따라 strong (etcd) ~ eventually (Eureka) |
| 사용 | Spring Cloud / Consul 기반 마이크로서비스 |

### 5.4 패턴 D — sidecar / service mesh

```
[instance] ── localhost → [sidecar proxy] → [mesh control plane (Istio / Linkerd)]
                                                ↑
                                          registry 통합 + 라우팅 정책
```

| 도구 | 특성 |
| --- | --- |
| **Istio** | Envoy sidecar + istiod control plane |
| **Linkerd** | linkerd-proxy sidecar + control plane |
| **Consul Connect** | Envoy sidecar + Consul registry |
| **Cilium Service Mesh** | sidecar-less (eBPF) — node 별 mesh |

| 특성 | 평가 |
| --- | --- |
| 구현 복잡도 | 높음 (mesh 운영) |
| 클라이언트 변경 | 거의 없음 (sidecar 가 transparent intercept) |
| 변경 전파 | 빠름 (xDS gRPC) |
| health check | mesh 가 active check + outlier detection |
| 부하 분산 | sidecar 가 L7 LB (weighted / consistent hash / locality-aware) |
| 추가 기능 | mTLS / retry / circuit break / observability |
| 사용 | 100+ 서비스의 마이크로서비스 / zero-trust 요구 / multi-cluster |

---

## 6. 비교 매트릭스

| 항목 | static config | DNS | client registry | sidecar mesh |
| --- | --- | --- | --- | --- |
| 동적 변경 | ✗ | △ (TTL) | ✓ | ✓ |
| 클라이언트 변경 | (불필요) | (불필요) | 필요 | 거의 없음 |
| health check | 없음 | 외부 | 통합 | 통합 (active + outlier) |
| L7 라우팅 | ✗ | ✗ | (클라이언트 구현) | ✓ |
| mTLS | ✗ | ✗ | (별도) | ✓ |
| 운영 부담 | 최저 | 낮음 | 중 | 높음 |
| 적합 규모 | < 10 서비스 | 10 ~ 100 | 50 ~ 200 | 100+ |

---

## 7. k8s 의 service discovery 모델

k8s 는 **DNS 기반 + iptables/IPVS/eBPF 의 ClusterIP DNAT** 의 조합이다.

### 7.1 흐름

```
1. 클라이언트 (pod) 가 "api.default.svc.cluster.local" 조회
2. CoreDNS → Service 의 ClusterIP (10.96.0.5) 응답
3. 클라이언트가 10.96.0.5:80 으로 connect
4. kube-proxy 의 iptables 룰이 PREROUTING 에서 random pod endpoint 로 DNAT
5. pod 응답
```

### 7.2 추가 모델

| 종류 | 의미 |
| --- | --- |
| **ClusterIP Service** | virtual IP + DNS 매핑. 위 표준 흐름 |
| **Headless Service** (`clusterIP: None`) | DNS 가 pod IP 들 직접 응답. StatefulSet 등 |
| **ExternalName Service** | DNS CNAME 매핑 (외부 도메인) |
| **Endpoint / EndpointSlice** | Service 의 실제 백엔드 IP 목록 (Endpoint controller 가 관리) |
| **External DNS** | Service / Ingress → 외부 DNS (Cloudflare / Route53) 자동 동기 |

상세: [[../../../devops/kubernetes/kubernetes-mental-models]] §11, [[container-networking]] §9.

---

## 8. 선택 기준

다음 질문 순서로 패턴 선택.

| 질문 | 답 | 패턴 |
| --- | --- | --- |
| 인스턴스 수 / IP 가 변하는가? | 아니오 | static config |
| 클라이언트 변경 비용이 큰가? | 그렇다 | DNS 기반 |
| L7 라우팅 / mTLS / retry 필요? | 아니오 | DNS 기반 |
| 100+ 마이크로서비스 + zero-trust? | 그렇다 | sidecar mesh |
| Spring / gRPC 클라이언트 측에서 LB 가능? | 그렇다 | client-side registry |
| k8s 안에서만? | 그렇다 | k8s Service + CoreDNS (DNS 기반) |
| k8s + 외부 클라이언트? | 그렇다 | DNS 기반 + ExternalDNS |

---

## 9. 흔한 실패 모드

| 실패 | 원인 | 모델 설명 |
| --- | --- | --- |
| 인스턴스 down 후에도 클라이언트가 한참 옛 IP 사용 | DNS TTL 길다 + 클라이언트가 추가 cache | §5.2 — TTL 한계 |
| Eureka 의 endpoint 가 죽은 인스턴스 포함 | self-preservation mode | Eureka 의 의도 — 짧은 네트워크 단절에서 deregister 폭주 방지 |
| k8s Service IP 가 응답 없음 | kube-proxy 죽음 / iptables 룰 부재 | [[../tools/iptables-netfilter]] §8 |
| Headless Service 에서 한 pod 만 응답 | 클라이언트가 첫 A record 만 사용 (round-robin 미구현) | §5.2 |
| sidecar 가 startup 늦어 첫 request 실패 | application 이 sidecar 보다 먼저 ready | sidecar holdApplicationUntilProxyStarts |
| Consul agent 가 죽고 모든 service unhealthy | agent 가 deregister 자동 트리거 | health check graceful interval 조정 |
| DNS lookup 폭주로 CoreDNS overload | 클라이언트 측 DNS cache 부재 | NodeLocalDNS / 클라이언트 cache |
| ExternalDNS 가 Cloudflare 변경 못함 | API token 권한 부족 또는 zone 미발견 | ExternalDNS log 확인 |

---

## 10. 참고

- [[topics|↑ topics]]
- [[../network|↑ network hub]]
- [[../dns/dns]] — DNS 프로토콜
- [[../dns/dns-record-types]] — A / SRV / CNAME
- [[../load-balancing/load-balancing]] — client-side vs server-side LB
- [[container-networking]] — k8s Service / ClusterIP 의 packet 흐름
- [[../tools/iptables-netfilter]] — kube-proxy 의 iptables 룰
- [[../../../devops/kubernetes/kubernetes-mental-models]] — Service / Endpoint / kube-proxy
- [[../../../devops/networking-ops/service-mesh]] — Istio / Linkerd / Cilium SM
- [[../../../devops/networking-ops/dns-provider-architecture]] — 외부 DNS 의 cascade
- Consul docs — https://www.consul.io
- CoreDNS docs — https://coredns.io
- ExternalDNS — https://github.com/kubernetes-sigs/external-dns
