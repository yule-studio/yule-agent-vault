---
title: "k3s networking — flannel / Traefik / klipper-lb"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T09:07:00+09:00
tags: [devops, k3s, networking, cni]
---

# k3s networking — flannel / Traefik / klipper-lb

**[[k3s|↑ k3s]]**

---

## 1. 구조

```
Pod ↔ Pod (cluster-wide)
   ↑ CNI (flannel default)
   
Service (ClusterIP)
   ↑ kube-proxy (iptables)

Service (LoadBalancer)
   ↑ klipper-lb (default) 또는 MetalLB
   
HTTP / HTTPS Ingress
   ↑ Traefik (default) 또는 nginx-ingress
```

---

## 2. flannel (CNI)

```
default backend: VXLAN
대안: host-gw, wireguard-native, wireguard-go, ipsec

# config
flannel-backend: "host-gw"   # 같은 L2 라면 (latency ↓)
flannel-backend: "wireguard-native"  # WAN over WireGuard (encryption)

# CIDR
cluster-cidr: "10.42.0.0/16"
service-cidr: "10.43.0.0/16"
```

→ host-gw = VXLAN overhead 없음. 단 router 가 routing 알아야.

---

## 3. flannel 끄고 다른 CNI

```yaml
# /etc/rancher/k3s/config.yaml
flannel-backend: "none"
disable-network-policy: true   # flannel 의 network policy 끔
```

```bash
# 그 후 직접 CNI 설치
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
# 또는
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/main/install/kubernetes/quick-install.yaml
```

→ Calico / Cilium 가 NetworkPolicy + 추가 기능 (BGP / eBPF L7).

---

## 4. Traefik (default Ingress)

```yaml
# 자동 배포 (kube-system/traefik)
# manifest: /var/lib/rancher/k3s/server/manifests/traefik.yaml

# 비활성:
disable:
  - traefik
```

```yaml
# Traefik IngressRoute (CRD)
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata: {name: web}
spec:
  entryPoints: [websecure]
  routes:
    - match: Host(`example.com`)
      kind: Rule
      services:
        - {name: web, port: 80}
  tls:
    certResolver: letsencrypt
```

```yaml
# 일반 Ingress (k8s 표준) 도 OK
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
```

→ 작은 cluster = Traefik 그대로. 큰 cluster = nginx-ingress / Cilium 으로 교체.

---

## 5. Traefik HelmChart customize

```yaml
# /var/lib/rancher/k3s/server/manifests/traefik-config.yaml
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: traefik
  namespace: kube-system
spec:
  valuesContent: |
    additionalArguments:
      - "--certificatesresolvers.letsencrypt.acme.tlschallenge=true"
      - "--certificatesresolvers.letsencrypt.acme.email=ops@example.com"
      - "--certificatesresolvers.letsencrypt.acme.storage=/data/acme.json"
    persistence:
      enabled: true
      path: /data
    ports:
      web:
        redirectTo: websecure
```

→ k3s 의 **HelmChart CRD** 가 자동 reconcile.

---

## 6. klipper-lb (Service LoadBalancer)

```
Service type: LoadBalancer
   ↓
klipper-lb 가 hostPort 로 노드의 :80, :443 등 bind
   ↓
external IP = node IP
```

```bash
kubectl get svc
# NAME   TYPE           EXTERNAL-IP    PORT(S)
# web    LoadBalancer   <node-ips>     80:32000/TCP
```

→ 단순. on-prem / homelab 친화. cloud LB 비용 X.

→ 한계: 한 port 에 한 service. 여러 service 면 MetalLB.

---

## 7. MetalLB (대안 LB)

```yaml
disable:
  - servicelb   # klipper-lb 끄기
```

```bash
helm install metallb metallb/metallb -n metallb-system --create-namespace
```

```yaml
# IP pool
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata: {name: pool}
spec:
  addresses: [192.168.1.240-192.168.1.250]

---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata: {name: l2}
spec:
  ipAddressPools: [pool]
```

→ 진짜 LB IP (각 service 마다). bare-metal 표준.

---

## 8. CoreDNS

```bash
kubectl -n kube-system get pod -l k8s-app=kube-dns
kubectl -n kube-system get cm coredns -o yaml
```

→ k3s 자동. cluster.local 의 service / pod 자동 resolve.

```yaml
# customize (kube-system/coredns-custom)
apiVersion: v1
kind: ConfigMap
metadata: {name: coredns-custom, namespace: kube-system}
data:
  example.server: |
    example.com {
        forward . 10.0.0.10
    }
```

---

## 9. dual-stack (IPv4 + IPv6)

```yaml
cluster-cidr: "10.42.0.0/16,fd00:42::/48"
service-cidr: "10.43.0.0/16,fd00:43::/112"
```

→ k3s 1.21+ 지원.

---

## 10. WireGuard (encrypted CNI)

```yaml
flannel-backend: "wireguard-native"
```

→ 노드 간 traffic 자동 encryption. WAN / 신뢰 안 되는 network.

→ overhead 약간 (5-10% throughput ↓).

---

## 11. firewall (★)

```
TCP:
  6443     kube-apiserver (server)
  10250    kubelet
  2379     etcd client (HA)
  2380     etcd peer (HA)

UDP:
  8472     flannel VXLAN
  51820    flannel wireguard (옵션)
  51821    flannel wireguard v6

ingress:
  80, 443  Traefik / nginx
```

```bash
# UFW (Ubuntu)
sudo ufw allow 6443/tcp
sudo ufw allow from 10.42.0.0/16 to any   # pod CIDR
sudo ufw allow from 10.43.0.0/16 to any   # svc CIDR
sudo ufw allow 8472/udp
```

---

## 12. 함정

1. **firewall 으로 6443 / 8472 막힘** — agent join 실패 / pod 통신 안 됨.
2. **flannel + Calico 같이** — 충돌. 한 CNI 만.
3. **Traefik 안 끄고 nginx-ingress** — port 충돌.
4. **klipper-lb hostPort 충돌** — 한 node 의 :80 에 두 LB svc.
5. **cluster-cidr 회사 network 와 중복** — 10.42.x.x 가 회사 LAN 이면 충돌.
6. **CoreDNS upstream 너무 느림** — `/etc/resolv.conf` 의 DNS 검토.
7. **dual-stack 미설정 시 IPv6 ingress 안 됨**.

---

## 13. 관련

- [[k3s|↑ k3s]]
- [[installation]]
- [[../networking-ops/networking-ops|↗ networking-ops]]
- [[../kubernetes/services-networking|↗ k8s services]]
