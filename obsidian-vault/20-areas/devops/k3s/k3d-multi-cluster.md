---
title: "k3d — k3s-in-docker / 다중 cluster dev"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T09:21:00+09:00
tags: [devops, k3s, k3d, kind]
---

# k3d — k3s-in-docker / 다중 cluster dev

**[[k3s|↑ k3s]]**

---

## 1. 무엇

- k3s 를 Docker container 안에 띄움.
- laptop 에 수십 cluster 가능 (각 cluster 가 container).
- 가볍고 빠른 (수초 안에 cluster 생성).
- CI / e2e test 표준 (kind 와 경쟁).

→ "kind 의 k3s 버전" + 더 빠르고 가벼움.

---

## 2. 설치

```bash
brew install k3d
# 또는
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
```

---

## 3. 기본 사용

```bash
# 만들기
k3d cluster create dev
# → docker 안에 k3s server + agent (default 1+1) 생성
# → kubeconfig 자동 설정

# 검증
kubectl get nodes
# NAME                STATUS   ROLES
# k3d-dev-server-0    Ready    control-plane
# k3d-dev-agent-0     Ready    <none>

# 클러스터 목록
k3d cluster list

# 삭제
k3d cluster delete dev
```

---

## 4. 고급 옵션

```bash
k3d cluster create my-cluster \
    --servers 3 \
    --agents 2 \
    --image rancher/k3s:v1.29.4-k3s1 \
    --port "8080:80@loadbalancer" \
    --port "8443:443@loadbalancer" \
    --volume /tmp/data:/data@all \
    --registry-create my-registry:5000 \
    --k3s-arg "--disable=traefik@server:*" \
    --network my-net
```

기능:
- 3 server HA (embedded etcd 자동)
- 2 agent
- host port mapping (LB)
- volume mount
- 자체 registry container 자동 생성
- k3s arg pass-through
- docker network

---

## 5. 자체 registry 통합 (★)

```bash
# registry 생성
k3d registry create my-registry --port 5000

# cluster + registry 연결
k3d cluster create dev \
    --registry-use k3d-my-registry:5000

# image push
docker tag my-app:latest k3d-my-registry:5000/my-app:latest
docker push k3d-my-registry:5000/my-app:latest

# k8s 에서 사용
image: k3d-my-registry:5000/my-app:latest
```

→ Docker Hub 의존성 X. dev 빠름.

---

## 6. CI 통합 (★)

```yaml
# GitHub Actions
jobs:
  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: AbsaOSS/k3d-action@v2
        with:
          cluster-name: e2e
          args: >-
            --agents 1
            --image rancher/k3s:v1.29.4-k3s1
            --no-lb

      - run: kubectl apply -f manifests/
      - run: ./test/e2e.sh
```

→ 1분 안에 cluster, e2e 후 자동 정리.

---

## 7. config file (k3d.yaml)

```yaml
# k3d.yaml
apiVersion: k3d.io/v1alpha5
kind: Simple
metadata: {name: dev}
servers: 3
agents: 2
image: rancher/k3s:v1.29.4-k3s1
ports:
  - port: 8080:80
    nodeFilters: [loadbalancer]
  - port: 8443:443
    nodeFilters: [loadbalancer]
options:
  k3s:
    extraArgs:
      - arg: "--disable=traefik"
        nodeFilters: [server:*]
registries:
  create:
    name: registry.local
    host: "0.0.0.0"
    hostPort: "5000"
  config: |
    mirrors:
      "docker.io":
        endpoint:
          - http://registry.local:5000
```

```bash
k3d cluster create --config k3d.yaml
```

→ 재현 가능 / git commit.

---

## 8. 다중 cluster (★ multi-env dev)

```bash
# 한 laptop 에 여러 cluster
k3d cluster create dev --servers 1 --agents 1
k3d cluster create staging --servers 1 --agents 2
k3d cluster create perf --servers 1 --agents 4

# 자유롭게 전환
kubectl config get-contexts
kubectl config use-context k3d-dev
kubectl config use-context k3d-staging
```

→ "각 환경 = 다른 cluster" 시뮬레이션.

---

## 9. resource 제한 (laptop 친화)

```bash
k3d cluster create small \
    --servers 1 --agents 0 \
    --memory 1G \
    --cpus 1
```

→ docker 의 resource limit. heavy cluster 도 noisy neighbor 안 됨.

---

## 10. k3d vs kind vs minikube

| | k3d | kind | minikube |
| --- | --- | --- | --- |
| 기반 | k3s | k8s | k8s |
| 속도 | 가장 빠름 (수초) | 빠름 (10초) | 느림 (30초+) |
| 메모리 | 가장 가벼움 | 가벼움 | 무거움 |
| HA | ✓ | ✓ | ✗ |
| LB | klipper-lb | metallb 별도 | minikube tunnel |
| multi-cluster | 쉬움 | 쉬움 | 어색함 |
| ingress | Traefik 자동 | manual | addon |
| Docker desktop 통합 | ✓ | ✓ | ✓ |
| CI 사용 | ✓ (빠름) | ✓ (표준) | ✗ |

→ **dev = k3d 또는 kind**, **prod-like = minikube**.

---

## 11. 실용 명령

```bash
# cluster stop / start (재시작)
k3d cluster stop dev
k3d cluster start dev

# kubeconfig export
k3d kubeconfig get dev > ~/.kube/dev.yaml

# image import (registry 거치지 않고)
docker build -t my-app:dev .
k3d image import my-app:dev -c dev

# port forward 안 해도 됨 (cluster 생성 시 --port)
curl http://localhost:8080
```

---

## 12. 함정

1. **Docker memory limit** — Mac Docker Desktop 의 RAM 부족.
2. **registry-create 후 cluster 별도 사용** — connection refused. `registry-use` 필요.
3. **WSL2** — DNS 충돌. `/etc/docker/daemon.json` 점검.
4. **너무 많은 cluster** — laptop 가 죽음.
5. **k3d 의 LB port — 동시 사용 시 충돌** — cluster 마다 다른 port.
6. **kubeconfig context 혼동** — `kubectx` / `kubie` 활용.

---

## 13. 관련

- [[k3s|↑ k3s]]
- [[installation]]
- [[../kubernetes/concepts|↗ k8s concepts]]
- [[../cicd/cicd|↗ CI/CD]]
