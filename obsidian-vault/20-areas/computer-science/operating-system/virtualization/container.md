---
title: "Container — Docker / runc / containerd / OCI"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T15:15:00+09:00
tags:
  - operating-system
  - virtualization
  - container
  - docker
---

# Container

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | Container 구조 + runtime |

**[[virtualization|↑ Virt hub]]**

---

## 1. 한 줄

호스트 커널을 공유하면서 **namespace + cgroups + 보안 layer** 로 격리된 process.

= 가벼운 VM 의 환상 + 빠른 부팅 + 작은 메모리.

---

## 2. 구성 요소

```
1. Namespace      — 격리
2. cgroups        — 자원 제한
3. rootfs / image — 자기 파일 시스템
4. capabilities   — root 의 일부만
5. seccomp        — syscall 필터
6. AppArmor/SELinux — MAC
7. read-only      — 무결성
```

---

## 3. OCI 표준

| Spec | 의미 |
| --- | --- |
| **OCI Runtime Spec** | 컨테이너 구성 (config.json) + rootfs |
| **OCI Image Spec** | image 포맷 (manifest, layers) |
| **OCI Distribution Spec** | registry API (v2) |

→ runc, podman, containerd 등 표준 준수.

---

## 4. Runtime 계층

```
Docker CLI / Kubernetes
       ↓
containerd (high-level)
       ↓
runc / crun (low-level, OCI)
       ↓
namespace + cgroup + seccomp + ...
       ↓
Linux Kernel
```

| 레이어 | 도구 |
| --- | --- |
| Orchestration | Kubernetes / Nomad |
| High-level runtime | containerd, CRI-O |
| Low-level runtime | runc (Go), crun (C, 빠름) |
| Image build | docker buildx / buildah / kaniko |
| CLI | docker / podman / nerdctl |

---

## 5. Image — Layered FS

```
Image = N 개의 read-only layer
Container = image + 1 writable layer (top)

OverlayFS / Btrfs / ZFS 의 union mount
```

```
Layer 4 (top, writable):   /tmp/foo
Layer 3 (image):           /app/code
Layer 2 (image):           /usr/lib/python
Layer 1 (image):           debian rootfs
```

→ 같은 base 사용 시 dedup → 디스크 / 다운로드 절약.

```bash
docker history myimage
docker inspect myimage | jq '.[0].RootFS.Layers'
```

---

## 6. Dockerfile

```dockerfile
FROM python:3.12-slim                  # base image
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["python", "app.py"]
```

각 `RUN/COPY/ADD` = 새 layer.

### 6.1 Best practice
- 작은 base (alpine, distroless)
- multi-stage build (build artifact 만 final 에)
- `.dockerignore`
- 변경 잦은 layer 를 뒤로
- pin version

---

## 7. Docker 의 일상 명령

```bash
docker build -t myapp:v1 .
docker images
docker run -d --name a -p 8000:8000 myapp:v1
docker ps
docker logs -f a
docker exec -it a bash
docker stop a
docker rm a
docker rmi myapp:v1

# 자원 제한
docker run --cpus=2 --memory=4g --pids-limit=1024 ...

# volume
docker run -v ./data:/data ...
docker run -v myvol:/data ...
docker volume ls

# network
docker network create mynet
docker run --network=mynet ...
```

---

## 8. 컨테이너 안의 PID 1

PID 1 = init. 책임:
- 자식 reap (SIGCHLD)
- SIGTERM 처리 → graceful shutdown
- signal forwarding

일반 응용은 PID 1 책임 모름 → 좀비 누적 / shutdown 무시.

해결:
- `docker run --init` (tini 자동)
- `dumb-init` / `tini` 명시
- 응용이 직접 처리

자세히 → [[../process/orphan-zombie#10-컨테이너의-pid-1-함정]]

---

## 9. OverlayFS

```bash
mount | grep overlay
# overlay on /var/lib/docker/.../merged type overlay
#   (lowerdir=L1:L2:L3, upperdir=upper, workdir=work)
```

- lower (read-only) + upper (writable) = merged
- write 시 copy-up 으로 upper 에 복사
- 효율적 layer 공유

---

## 10. Volume

| 종류 | 의미 |
| --- | --- |
| `-v /host:/container` | bind mount |
| `-v name:/container` | named volume (docker 관리) |
| `--mount type=tmpfs,...` | tmpfs |
| named volume + driver | nfs / iscsi / ... |

데이터 영속 / 공유.

---

## 11. Network

| 모드 | 의미 |
| --- | --- |
| `bridge` (기본) | virtual bridge + NAT |
| `host` | host network namespace 공유 — 격리 X |
| `none` | network X |
| `container:<id>` | 다른 container 의 net ns 공유 |
| custom bridge | user-defined + DNS |

K8s = CNI plugin (Calico, Flannel, Cilium).

---

## 12. K8s Pod

```
Pod = 같은 namespace (net, ipc, uts, pid optional) 공유 컨테이너 그룹
```

- 같은 IP / port 공유
- localhost 로 통신
- volume 공유

→ "sidecar" 패턴의 토대.

---

## 13. rootless container

user namespace 활용 → 호스트 root 없이 컨테이너 안 root.

```bash
# podman 은 기본 rootless
podman run -it ubuntu

# docker 도 rootless mode 지원
dockerd-rootless-setuptool.sh install
```

장점:
- 보안 ↑
- CI / 개발 환경
- HPC

단점:
- 일부 기능 제한 (network, port < 1024, ...)
- 성능 약간 ↓

---

## 14. 보안 layer

### 14.1 capability drop

```bash
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE ...
```

### 14.2 seccomp

```bash
docker run --security-opt seccomp=profile.json ...
```

→ 일부 syscall 만 허용.

자세히 → [[../security/seccomp]] (추후)

### 14.3 read-only rootfs

```bash
docker run --read-only --tmpfs /tmp ...
```

### 14.4 no-new-privileges

```bash
docker run --security-opt=no-new-privileges ...
```

setuid binary 의 권한 escalation 차단.

### 14.5 AppArmor / SELinux

```bash
docker run --security-opt apparmor=docker-default ...
```

---

## 15. Image 보안

- 작은 base (distroless / alpine)
- pinned version
- vulnerability scan (trivy, grype, snyk)
- signed image (cosign, notary)
- supply chain (SLSA)

---

## 16. Build 도구

| 도구 | 특징 |
| --- | --- |
| **docker buildx** | BuildKit (병렬, cache) |
| **buildah** | daemonless |
| **kaniko** | rootless, K8s 안 build |
| **img** | daemonless, snapshot |
| **podman build** | podman 의 |

---

## 17. Light VM 컨테이너

| | 일반 container | Light VM |
| --- | --- | --- |
| 격리 | namespace | + hypervisor |
| 부팅 | < 1s | < 1s |
| 보안 | 약 | 강 |
| 도구 | runc | Kata / Firecracker |

자세히 → [[vm-hypervisor#7-light-vm]]

---

## 18. WebAssembly Container

```
WASI binary → wasmtime / wasmer 안에서
극히 가벼움 + cross-platform
```

K8s 가 WASM workload 지원 진화 중 (containerd-wasm-shim).

---

## 19. 함정

### 19.1 PID 1 함정
tini / --init.

### 19.2 memory.max + JVM
heap + native + GC 합 < limit.

### 19.3 CPU throttling
burst 시 latency spike.

### 19.4 disk full
overlay upper dir → disk full → 컨테이너 fail.

### 19.5 swap in container
보통 비활성. memory.max hard hit.

### 19.6 host network 의존
격리 X — 보안 / portability ↓.

### 19.7 :latest tag
재현성 X. semver pin.

### 19.8 환경 변수의 secret
`-e` 의 secret 은 `ps` 에 노출. secret manager.

---

## 20. 학습 자료

- **OCI Spec** — opencontainers.org
- **Containers from Scratch** — Liz Rice
- **Docker Documentation**
- **Kubernetes Documentation**
- **Brendan Gregg — Container Perf**

---

## 21. 관련

- [[namespace]]
- [[cgroups]]
- [[vm-vs-container]]
- [[../security/security]]
- [[virtualization]] — Virt hub
