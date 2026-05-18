---
title: "CNI deep — Calico / Cilium / Flannel / eBPF"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T11:47:00+09:00
tags: [devops, networking-ops, cni, cilium, calico, ebpf]
---

# CNI deep — Calico / Cilium / Flannel / eBPF

**[[networking-ops|↑ networking-ops]]**

---

## 1. CNI 란

```
Container Network Interface (CNCF spec).

역할:
  - pod 에 IP 할당
  - pod ↔ pod traffic 라우팅
  - NetworkPolicy enforcement
  - service / LB 통합

구현체 (plugin):
  k8s 가 cluster 시작 시 한 plugin 선택.
```

---

## 2. 주요 CNI 비교

| | base | NetworkPolicy | encryption | scale | 사용 |
| --- | --- | --- | --- | --- | --- |
| **flannel** | VXLAN | ✗ | WireGuard | small | k3s default |
| **Calico** | BGP / VXLAN | ✓ (★) | WireGuard | large | enterprise 표준 |
| **Cilium** | eBPF | ✓ + L7 | WireGuard / IPsec | massive | ★ modern |
| **Weave Net** | VXLAN | ✓ | ✓ | small-mid | 간단 |
| **AWS VPC CNI** | AWS routing | ✓ (별도) | TLS | EKS native | EKS |
| **Azure CNI** | Azure native | ✓ | TLS | AKS native | AKS |
| **GKE CNI** | Google VPC | ✓ (✓ Dataplane V2 = Cilium) | TLS | GKE native | GKE |
| **Antrea** | OVS | ✓ | ✓ | mid-large | VMware |
| **Multus** | meta | (plugin) | ✓ | special | multi-CNI (NFV) |
| **Kube-OVN** | OVN | ✓ | ✓ | large | 통신/네트워크 회사 |

→ **modern = Cilium 추세** (eBPF + L7 + service mesh).

---

## 3. flannel (k3s default)

```yaml
# config
flannel-backend: vxlan         # default (overlay)
flannel-backend: host-gw        # 같은 L2 (overhead ↓)
flannel-backend: wireguard-native    # encryption
```

장단:
- 가장 단순
- VXLAN overhead 약간
- NetworkPolicy X (Calico / Canal 같이)

---

## 4. Calico (★ 대규모 표준)

```bash
# 설치
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml

# 또는 operator
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml
```

```yaml
# Installation
apiVersion: operator.tigera.io/v1
kind: Installation
spec:
  calicoNetwork:
    bgp: Enabled                       # ★ 또는 Disabled (VXLAN)
    ipPools:
      - cidr: 10.42.0.0/16
        encapsulation: VXLANCrossSubnet
        natOutgoing: Enabled
```

기능:
- BGP (node 간 routing, NAT 없음)
- VXLAN fallback (BGP 안 되는 환경)
- NetworkPolicy + GlobalNetworkPolicy
- WireGuard encryption
- pod IP 가 routable (외부 network 에서 직접 도달)

```yaml
# Global Policy (전체 cluster)
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata: {name: deny-all-egress-internet}
spec:
  selector: tier == 'internal'
  egress:
    - action: Deny
      destination:
        notNets: [10.0.0.0/8, 192.168.0.0/16]
```

---

## 5. Cilium (★ modern)

```bash
helm install cilium cilium/cilium \
    -n kube-system \
    --set kubeProxyReplacement=true \      # ★ kube-proxy 교체
    --set hostServices.enabled=true \
    --set externalIPs.enabled=true \
    --set nodePort.enabled=true \
    --set hubble.enabled=true \             # observability
    --set hubble.ui.enabled=true
```

→ "kubeProxyReplacement" 가 핵심: iptables / IPVS 의 kube-proxy 를 eBPF 로 교체. 더 빠름.

---

## 6. eBPF 란 (★)

```
extended Berkeley Packet Filter:
  - Linux kernel 안에서 sandbox 코드 실행
  - kernel module 없이 동적 kernel 변경
  - 매우 빠름 (kernel 단)
  - networking / security / observability / tracing

→ Cilium / Tetragon / Pixie / Falco 의 기반.
```

```
일반 packet flow:
  pod → veth → iptables (CPU 부하) → host eth0
  
Cilium:
  pod → veth → eBPF (kernel) → host eth0
  → bypass iptables. 10x 빠름.
```

---

## 7. Cilium 의 기능

```
1. CNI (pod networking)
   - VXLAN or native routing or BGP

2. kube-proxy replacement
   - eBPF 가 service / NodePort 처리

3. NetworkPolicy
   - L3/L4 + L7 (HTTP method/path)
   - 표준 k8s + CiliumNetworkPolicy

4. observability (Hubble)
   - flow log (실시간)
   - service map
   - latency histogram

5. service mesh (Cilium Service Mesh)
   - mTLS (eBPF, sidecar 없음)
   - L7 routing
   - retry / circuit break

6. cluster mesh (multi-cluster)
   - 여러 cluster 의 service 통신
   
7. BGP control plane
   - LB IP / pod CIDR advertise

8. encryption
   - WireGuard or IPsec (transparent)
```

---

## 8. CiliumNetworkPolicy L7 (★)

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata: {name: api-l7}
spec:
  endpointSelector:
    matchLabels: {app: api}
  ingress:
    - fromEndpoints:
        - matchLabels: {app: web}
      toPorts:
        - ports:
            - {port: "8080", protocol: TCP}
          rules:
            http:
              - {method: "GET", path: "/api/v1/.*"}
              - {method: "POST", path: "/api/v1/login"}
              - method: "POST"
                path: "/api/v1/orders"
                headers:
                  - "X-User-Tier: premium"
```

→ "web pod 는 api 의 특정 endpoint 만 + 특정 header 면 OK".

→ k8s 표준 NetworkPolicy 는 L3/L4 만. Cilium 가 L7.

---

## 9. Hubble (★ observability)

```bash
# UI
cilium hubble ui

# CLI
hubble observe                          # 실시간 flow
hubble observe --pod web                # 특정 pod
hubble observe --verdict DROPPED         # 거부된 traffic 만
hubble observe --to-service redis        # service 별

# service map (Grafana)
```

```
일반 monitoring 과 다름:
  network flow / NetworkPolicy 의 효과 / DNS query / HTTP path
  → 모두 pod 안의 application 코드 변경 0.
```

---

## 10. cilium service mesh

```
sidecar-less mesh:
  Istio: 각 pod 에 Envoy sidecar (메모리 50MB+ × N pod)
  Cilium SM: eBPF + node 별 Envoy (sidecar 없음)
  → memory / latency 우수
```

```yaml
# mTLS 자동
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
spec:
  endpointSelector:
    matchLabels: {app: api}
  ingress:
    - fromEndpoints:
        - matchLabels: {app: web}
      authentication:
        mode: required        # mTLS 강제
```

---

## 11. AWS VPC CNI

```
EKS default:
  - pod IP = VPC subnet 의 IP (★)
  - 외부 network 에서 직접 도달
  - security group per pod (★)
  
trade-off:
  + flat / fast
  + AWS native (SG / Flow Logs)
  - subnet IP 부족 (max pod = ENI 의 IP 수)
  - VPC peering / Transit Gateway complexity

ENI 한계:
  - instance type 별 max ENI / IP
  - prefix delegation 옵션 (1 ENI → 16 IP)
```

---

## 12. CNI 결정 매트릭스

| 시나리오 | CNI |
| --- | --- |
| k3s edge / 작은 | flannel (default) |
| general production | Calico |
| modern / observability / L7 policy | ★ Cilium |
| EKS, security group per pod 필요 | AWS VPC CNI |
| GKE | GKE Dataplane V2 (= Cilium) |
| service mesh integration | Cilium 또는 Istio + Calico |
| NFV / multi-network | Multus |

---

## 13. 성능 비교 (대략)

```
iptables-based (default kube-proxy):
  10k service: 룰 폭주, latency 큼
  
IPVS-based:
  hash table, scale 좋음

Cilium eBPF kube-proxy replacement:
  최고 성능 (kernel 단)
  large cluster (1000+ node) 권장
```

---

## 14. troubleshoot

```bash
# pod connectivity
kubectl exec <pod> -- curl <service>
kubectl exec <pod> -- nc -zv <pod-ip> <port>

# NetworkPolicy 영향
hubble observe --from-pod default/<pod> --verdict DROPPED
# 또는 Calico
calicoctl get networkpolicy

# CNI plugin log
journalctl -u kubelet | grep -i cni
cat /etc/cni/net.d/*

# CNI 별 도구
cilium status
cilium connectivity test
calicoctl node status
```

---

## 15. 함정

1. **flannel + NetworkPolicy** — flannel 자체 미지원.
2. **CNI 변경** — cluster 재생성 필요 (대부분).
3. **AWS VPC CNI 의 IP 부족** — prefix delegation 또는 작은 instance.
4. **iptables 룰 폭주** — Cilium 으로 교체.
5. **Cilium eBPF 의 kernel version** — 5.4+ 권장.
6. **multi-cluster network** — 잘못 구성 시 IP 충돌.
7. **encryption overhead** — WireGuard / IPsec 약 5-10% throughput ↓.

---

## 16. 관련

- [[networking-ops|↑ networking-ops]]
- [[service-mesh]]
- [[network-policy]]
- [[../kubernetes/services-networking|↗ k8s services]]
- [[../../computer-science/network/topics/container-networking|↗ CS — container networking (veth / bridge / VXLAN)]] — CNI 가 구현하는 OS 기본 단위
- [[../../computer-science/network/tools/iptables-netfilter|↗ CS — iptables / netfilter]] — CNI 의 packet 처리 backend
