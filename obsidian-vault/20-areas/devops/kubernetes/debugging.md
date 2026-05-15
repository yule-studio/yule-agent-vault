---
title: "Kubernetes debugging — pod / network / DNS"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T11:22:00+09:00
tags: [devops, kubernetes, debugging, troubleshooting]
---

# Kubernetes debugging — pod / network / DNS

**[[kubernetes|↑ kubernetes]]**

---

## 1. 60초 진단

```bash
kubectl get pod -A | grep -v Running          # 비정상 pod
kubectl top nodes                              # node 자원
kubectl top pods -A                            # pod 자원
kubectl get events --sort-by='.lastTimestamp' -A | tail
kubectl describe node <node>                   # node 상태
```

---

## 2. pod 진단

```bash
# 상태
kubectl get pod <name> -o wide
kubectl describe pod <name>                    # event + status detail

# log
kubectl logs <name> -c <container>             # 한 container
kubectl logs <name> --previous                 # 이전 instance (crash 분석)
kubectl logs -f <name>                         # follow
kubectl logs <name> --since=1h
kubectl logs <name> --tail=100

# exec
kubectl exec -it <name> -- /bin/sh
kubectl exec <name> -- ps aux
kubectl exec <name> -- env

# debug container (★ k8s 1.18+)
kubectl debug <name> -it --image=busybox --target=<container>
# → 같은 process namespace 안에서 debug tool 사용
```

---

## 3. pod 상태별 진단

### Pending

```bash
kubectl describe pod <name>

# 흔한 원인:
# - "0/3 nodes are available: insufficient cpu/memory"
#   → resource request 작게 또는 node 늘림
# - "no nodes match node selector"
#   → nodeSelector / affinity 검토
# - "no taint match"
#   → toleration 추가
# - "no PersistentVolume available"
#   → PVC 의 StorageClass 또는 PV 부족
# - "ImagePullBackOff"
#   → image name / credentials
```

### CrashLoopBackOff

```bash
kubectl logs <name> --previous
# → 죽기 직전 log

kubectl describe pod <name>
# → exit code 분석:
# 0    = 정상 종료 (왜?)
# 1    = 일반 error
# 137  = SIGKILL (OOM)
# 139  = SIGSEGV (segfault)
# 143  = SIGTERM (외부 종료)

kubectl get events --field-selector involvedObject.name=<pod>
```

### ImagePullBackOff

```bash
kubectl describe pod <name> | grep -A5 "Events"

# 원인:
# - image name 오타
# - tag 없음
# - private registry credentials 없음
#   → imagePullSecrets / ServiceAccount

# 디버그
kubectl get secret <pull-secret> -o yaml
docker pull <image>   # 같은 image manually
```

### Running but not Ready

```bash
kubectl describe pod <name>
# → readiness probe fail

kubectl logs <name>
# → app 의 startup 문제

# exec 로 직접 check
kubectl exec <name> -- curl localhost:8080/actuator/health
```

---

## 4. OOMKilled

```bash
kubectl describe pod <name> | grep -A5 "Last State"
# State:       Terminated
# Reason:      OOMKilled
# Exit Code:   137

# 분석:
# 1. resource limit 확인
kubectl get pod <name> -o yaml | grep -A5 limits

# 2. 실제 memory 사용 history
kubectl top pod <name> --containers

# 3. JVM 의 경우
kubectl exec <name> -- jcmd 1 GC.heap_info

# 해결:
# - limit 늘림
# - JVM heap 조정 (Xmx)
# - memory leak 분석 (heap dump)
```

---

## 5. network 진단

```bash
# pod → service
kubectl exec <pod> -- nslookup <service-name>
kubectl exec <pod> -- curl http://<service>:<port>/health

# service endpoint
kubectl get endpoints <service-name>
# → pod IP list (없으면 selector 매칭 안 됨)

# pod IP 직접
kubectl exec <pod> -- curl http://<pod-ip>:<port>

# DNS
kubectl exec <pod> -- nslookup kubernetes.default.svc.cluster.local
kubectl exec <pod> -- cat /etc/resolv.conf

# network 도구 pod 띄우기 (★)
kubectl run -it --rm debug --image=nicolaka/netshoot -- bash
# 안에서: dig / curl / tcpdump / nslookup / nc / iperf3
```

---

## 6. DNS 문제

```bash
# CoreDNS 상태
kubectl get pod -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns

# pod 의 DNS 설정
kubectl exec <pod> -- cat /etc/resolv.conf
# nameserver 10.96.0.10
# search default.svc.cluster.local svc.cluster.local cluster.local
# options ndots:5

# DNS resolve 시도
kubectl exec <pod> -- nslookup my-service
kubectl exec <pod> -- nslookup my-service.default.svc.cluster.local

# 자주 본 문제:
# - service name 오타
# - namespace 다름 (FQDN 사용)
# - CoreDNS pod crash
# - NetworkPolicy 가 DNS (UDP 53) 차단
```

---

## 7. NetworkPolicy 디버그

```bash
# 적용 중인 NetworkPolicy
kubectl get networkpolicy -A

# 특정 pod 가 어느 NP 영향?
# → Cilium hubble
hubble observe --from-pod default/my-pod --verdict DROPPED

# 또는 Calico
calicoctl get networkpolicy

# 임시 test
kubectl exec <source-pod> -- nc -zv <target-svc> <port>
# 막힘 = NetworkPolicy 의심
```

---

## 8. resource / scheduling

```bash
# node 상태
kubectl describe node <node>
# → conditions / allocatable / non-terminated pod

# pending pod 의 이유
kubectl get events --field-selector reason=FailedScheduling

# resource quota
kubectl describe resourcequota -n <namespace>

# LimitRange
kubectl get limitrange -n <namespace>

# 정렬 (most used)
kubectl top pod -A --sort-by=cpu | head
kubectl top pod -A --sort-by=memory | head
```

---

## 9. PV / PVC 문제

```bash
# PVC pending
kubectl describe pvc <name>
# → "no PV available" 또는 "WaitForFirstConsumer"

# PV 확인
kubectl get pv
kubectl describe pv <name>

# StorageClass
kubectl get storageclass
kubectl describe sc <name>

# 흔한 문제:
# - StorageClass 없음 (default)
# - PV / PVC AZ mismatch
# - reclaim policy Retain → PV released 상태로 남음
```

---

## 10. control plane 문제

```bash
# component status
kubectl get componentstatuses    # legacy

# 또는
kubectl get pod -n kube-system

# api server log
kubectl logs -n kube-system kube-apiserver-<node>

# etcd
kubectl exec -n kube-system etcd-<node> -- etcdctl endpoint health

# 또는 control plane node 에서
journalctl -u kubelet -f
crictl ps -a    # containerd 의 container
```

---

## 11. ephemeral debug container (★ 1.18+)

```bash
# image 안에 debug tool 없는 경우
kubectl debug -it <pod> --image=nicolaka/netshoot --target=<container>

# 또는 새 pod
kubectl debug node/<node> -it --image=ubuntu

# attach to running container
kubectl debug -it <pod> --share-processes --copy-to=debug-pod
```

---

## 12. kubectl plugin (★)

```bash
brew install krew                 # plugin manager

# 자주 쓰는 plugin
kubectl krew install neat         # output 정리
kubectl krew install ctx          # context switch
kubectl krew install ns           # namespace switch
kubectl krew install resource-capacity   # capacity 분석
kubectl krew install tail          # multi-pod log
kubectl krew install whoami        # 현재 user
kubectl krew install tree          # ownership tree
kubectl krew install access-matrix # RBAC 분석

kubectl ctx                       # context list
kubectl ns prod                   # namespace 전환
kubectl tail -l app=web           # pod 들의 log tail
```

---

## 13. 외부 도구

```
k9s            ★ TUI dashboard
Lens           desktop GUI
Headlamp       desktop GUI (CNCF)
Octant         legacy GUI

stern          multi-pod log
kail           log stream
ksniff         tcpdump in pod
kubectl-trace  bpftrace
```

---

## 14. application level

```
- jstack (Java thread dump)
- jmap (Java heap)
- py-spy (Python)
- delve (Go)
- gdb (C/C++)

→ pod 안에 도구 없으면 ephemeral debug container.
```

---

## 15. 흔한 실수 (★)

1. **labels mismatch** — Service selector 와 Pod label.
2. **namespace 안 명시** — 다른 namespace 의 resource 못 봄.
3. **resource limit = request** — overcommit 불가.
4. **probe 너무 짧음** — startup 시간 못 따라잡음.
5. **DNS UDP block** — NetworkPolicy.
6. **OOMKilled 무시** — log 만 봄, exit code 137 못 봄.
7. **previous log** 안 봄 — crash 시 이전 인스턴스.
8. **etcd disk slow** — control plane 느림.

---

## 16. 관련

- [[kubernetes|↑ kubernetes]]
- [[concepts]]
- [[pitfalls]]
- [[../linux/performance-troubleshooting|↗ linux perf]]
