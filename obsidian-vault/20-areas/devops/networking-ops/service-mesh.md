---
title: "Service Mesh — Istio / Linkerd / Consul"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T07:48:00+09:00
tags: [devops, networking-ops, service-mesh, istio]
---

# Service Mesh — Istio / Linkerd / Consul

**[[networking-ops|↑ networking-ops]]**

---

## 1. 무엇

```
[service A] ──┐                        ┌── [service B]
              │ (Pod 옆에 sidecar)      │
        [Envoy]  ──── mTLS / retry / LB ──── [Envoy]
              │                        │
              └── 통제 plane (Istiod) ──┘
                  (config, cert, observability)
```

- 각 Pod 에 sidecar proxy (Envoy) 자동 주입.
- service ↔ service 트래픽이 mesh 를 거침.
- 횡단 관심사 (mTLS / retry / circuit / observability) 를 코드 변경 X.

---

## 2. 왜 필요

### 왜 필요
- MSA 의 N×N 통신.
- mTLS / retry / circuit break / observability 를 각 service 마다 라이브러리 → 언어별 중복.
- traffic 관찰 (어디서 느린지) 어려움.

### 안 하면 문제
- service 간 평문 통신 (보안).
- 한 service 의 fail 이 cascade.
- distributed tracing 누락.

### 대안
- 라이브러리 (Resilience4j, Hystrix) — 언어별 재구현.
- API Gateway — service ↔ service 안 다룸.
- DAPR — sidecar 의 다른 방식 (mesh 아님).

### 트레이드오프
- complexity 폭증.
- latency 추가 (sidecar hop).
- 운영 도구 학습 (Istio 의 CRD 들).

---

## 3. 도구 비교

| | proxy | 설치 | 강점 | 약점 |
| --- | --- | --- | --- | --- |
| **Istio** | Envoy | 무거움 | feature 가장 풍부 | 학습 곡선 ↑ |
| **Linkerd** | linkerd2-proxy | 가벼움 | 단순 / 빠름 | feature 적음 |
| **Consul Connect** | Envoy | 중간 | HashiCorp 통합 | k8s only X |
| **Kuma** | Envoy | 중간 | multi-cluster | 작은 community |
| **Cilium Service Mesh** | eBPF (sidecar X) | 가벼움 | kernel-level | 성숙도 |
| **AWS App Mesh** | Envoy | 중간 | AWS 통합 | AWS lock |

→ **첫 도입 = Linkerd**. **enterprise / 풍부한 정책 = Istio**.

---

## 4. Istio 설치

```bash
# istioctl
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.20*

./bin/istioctl install --set profile=demo -y
kubectl label namespace default istio-injection=enabled
```

→ pod 재시작 시 sidecar 자동 주입.

---

## 5. 핵심 CRD (Istio)

```yaml
# VirtualService — routing
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata: {name: reviews}
spec:
  hosts: [reviews]
  http:
    - match: [{headers: {x-user-tier: {exact: premium}}}]
      route: [{destination: {host: reviews, subset: v2}}]
    - route: [{destination: {host: reviews, subset: v1}}]

---
# DestinationRule — subset 정의
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata: {name: reviews}
spec:
  host: reviews
  subsets:
    - name: v1
      labels: {version: v1}
    - name: v2
      labels: {version: v2}
  trafficPolicy:
    connectionPool:
      tcp: {maxConnections: 100}
      http: {http1MaxPendingRequests: 100}
    outlierDetection:
      consecutiveErrors: 5
      interval: 30s
      baseEjectionTime: 60s
```

---

## 6. mTLS

```yaml
# 전체 cluster strict mTLS
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata: {name: default, namespace: istio-system}
spec:
  mtls: {mode: STRICT}
```

→ service ↔ service 가 자동 암호화. cert 발급 / rotation 자동.

---

## 7. canary / traffic split

```yaml
spec:
  http:
    - route:
        - destination: {host: reviews, subset: v1}
          weight: 90
        - destination: {host: reviews, subset: v2}
          weight: 10
```

→ 10% 만 v2 로. 점진 ramp-up.

---

## 8. circuit break / retry

```yaml
trafficPolicy:
  outlierDetection:
    consecutiveErrors: 5         # 5번 연속 error → pod eject
    interval: 30s
    baseEjectionTime: 60s
  retries:
    attempts: 3
    perTryTimeout: 2s
    retryOn: 5xx,gateway-error,connect-failure
```

---

## 9. observability

자동 metric / trace / log:
- Prometheus (`istio_requests_total`, `istio_request_duration_milliseconds`)
- Jaeger / Tempo (trace_id 자동 전파)
- Kiali (mesh 시각화)
- Grafana dashboard 내장

→ 코드 변경 0 으로 observability 획득.

---

## 10. Linkerd (간단 대안)

```bash
linkerd install | kubectl apply -f -
kubectl annotate ns default linkerd.io/inject=enabled
```

```bash
# 같은 효과 (mTLS / retry / observability) — config 없이 default 적용
linkerd viz install | kubectl apply -f -
linkerd viz dashboard
```

→ 학습 부담 ↓, feature ↓.

---

## 11. ambient mesh (★ 최신 Istio)

```
sidecar 없음 — node 마다 ztunnel (mTLS) + waypoint proxy (L7).
```

→ sidecar overhead 제거. 2024-25 신규 도입 적합.

---

## 12. 함정

1. **mesh 부재로 충분한데 도입** — over-engineering.
2. **sidecar overhead** — 메모리 50-100MB × pod 수.
3. **mTLS PERMISSIVE → STRICT** — 마이그레이션 신중.
4. **istioctl version mismatch** — control plane vs data plane.
5. **VirtualService 우선순위** — 첫 매칭 rule.
6. **외부 → mesh** — IngressGateway 필요 (LB → 일반 svc 다른 flow).

---

## 13. 관련

- [[networking-ops|↑ networking-ops]]
- [[api-gateway]]
- [[../monitoring/opentelemetry|↗ OTel]]
- [[../kubernetes/kubernetes|↗ k8s]]
