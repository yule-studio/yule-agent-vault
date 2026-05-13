---
title: "VM vs Container — 비교 / 결정"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T15:20:00+09:00
tags:
  - operating-system
  - virtualization
  - container
  - vm
---

# VM vs Container

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 비교표 + 결정 트리 |

**[[virtualization|↑ Virt hub]]**

---

## 1. 한 줄

VM = 강한 격리, 무거움.
Container = 빠르고 가벼움, 격리 약함 (커널 공유).

→ 보통 같이 — **VM 안의 container** (K8s 노드 = VM, pod = container).

---

## 2. 비교표

| 측면 | VM | Container |
| --- | --- | --- |
| **커널** | 게스트 자기 | 호스트 공유 |
| **격리** | 강 (HW virt) | 약 (namespace) |
| **부팅** | 30 s ~ 분 | < 1 s |
| **메모리** | GB (게스트 OS) | MB (process 만) |
| **디스크 image** | GB | MB |
| **밀도** | 10-50 / host | 100-1000 / host |
| **다른 OS** | ✅ (Windows on Linux 등) | ❌ (같은 kernel) |
| **보안** | 강 (hypervisor) | 보통 (kernel exploit 위험) |
| **신뢰 모델** | tenant 분리 OK | 같은 tenant 안 |
| **운영 부담** | 큼 (OS 관리) | 작음 |
| **CI / 개발** | 무거움 | 빠름 |
| **multi-process** | 자연 | 보통 1 process per container |
| **호환성** | OS 그대로 | Linux 한정 |

---

## 3. 격리 강도

```
강함  ─────────────────────────────  약함
 |              |              |              |
 VM (HW)     Light VM      Container      Process
 (KVM/ESXi) (Firecracker)   (runc)         (basic)
```

```
              + gVisor (user-space syscall)
              + Kata (per-pod VM)
```

---

## 4. 자원 오버헤드

### 4.1 VM
- OS 페이지 (커널 + userland) — 500 MB ~ GB
- vCPU emulation 1-5%
- 디스크 I/O virtio 우수
- 부팅 시간 → 응용 시작 지연

### 4.2 Container
- process 자체 만 (~ 100 MB JVM, 5 MB Go, ...)
- 거의 native CPU
- cgroup 미세 자원 제한
- 즉시 시작

→ 같은 hardware 에 container 10-50배 더 dense.

---

## 5. 보안 모델

### 5.1 VM
- 격리: hypervisor + VT-x EPT
- 위협: hypervisor 취약점 (CVE), side channel
- multi-tenant 표준 (AWS EC2)

### 5.2 Container
- 격리: namespace + cgroup + seccomp + cap + AppArmor
- 위협: kernel exploit (escape), shared kernel state
- 같은 tenant 안 / 신뢰 환경
- 다중 tenant = Light VM (Firecracker, gVisor)

---

## 6. 사용 시나리오

| 시나리오 | 추천 |
| --- | --- |
| Multi-tenant SaaS (제3자 코드) | Light VM / VM |
| 마이크로서비스 | Container |
| CI / build | Container |
| 다른 OS 실행 (Windows app on Linux) | VM |
| Database 운영 | 둘 다 (대형 = baremetal) |
| Lambda / FaaS | Light VM (Firecracker) |
| HPC / AI training | Container + GPU passthrough or baremetal |
| 개발 환경 | Container (devcontainer) |
| 레거시 OS | VM |
| ML inference | Container |

---

## 7. 결정 트리

```
다른 OS / 강한 격리 필요?
└── VM

같은 tenant + 가벼움?
└── Container

untrusted code (FaaS, online IDE)?
└── Light VM (Firecracker, gVisor)

거대 HPC / 진짜 native?
└── baremetal

K8s?
└── 노드 = VM, pod = container
```

---

## 8. 함께 — Stack 의 진실

```
Cloud:
  Bare-metal host
    Hypervisor (KVM)
      VM (Linux)
        Container (Pod)
          Process
```

거의 모든 공용 클라우드. AWS Fargate / GCP Cloud Run 은 Light VM 으로 한 단계 간소화.

---

## 9. 마이그레이션

### 9.1 VM → Container
- 12-factor app 원칙
- stateless 분리
- config 환경변수
- log → stdout
- multi-process → multi-container (sidecar)
- network = service mesh

### 9.2 Container → VM
- 강한 격리 필요 (regulation, multi-tenant)
- 다른 OS

### 9.3 Hybrid
- VM 안 Docker — 가장 흔함
- Light VM (Kata) — 격리 ↑ + container API

---

## 10. 운영 관점

### 10.1 VM
- OS patch / upgrade (운영 부담)
- snapshot / live migration
- 백업 = disk image
- 모니터링 = inside VM

### 10.2 Container
- image 만 patch
- 이동 (다른 호스트 즉시)
- 백업 = image + volume
- 모니터링 = cAdvisor / Prometheus

→ container 가 cloud-native 운영에 유리.

---

## 11. 성능 비교

| 측면 | VM | Container | baremetal |
| --- | --- | --- | --- |
| CPU | 99% | 100% | 100% |
| 메모리 | 95% | 100% | 100% |
| Network | 98% (virtio) | 100% | 100% |
| Disk | 95% (virtio) | 100% | 100% |
| Boot | 분 | < 1s | — |
| Density | 10-50 | 100-1000 | — |

→ container 가 매우 가까운 native. VM 도 modern paravirt 면 ~ 95%.

---

## 12. 함정

### 12.1 Container 의 "VM 같다" 가정
같은 커널 — 격리 X. 멀티 tenant 위험.

### 12.2 VM 의 가벼움 가정
GB 메모리 + 분 단위 부팅.

### 12.3 Container 안의 SSH
보통 잘못된 패턴. log + exec 으로.

### 12.4 VM image vs Container image 혼동
VM = OS + 데이터. Container = 응용 layer.

### 12.5 K8s 의 hostNetwork
격리 포기 — 의도적이어야.

### 12.6 Stateful workload + container
volume / PV / operator 사용.

### 12.7 Light VM 의 보안 환상
완전 격리는 baremetal. side channel 항상 위험.

---

## 13. 학습 자료

- **Container vs VM** — 많은 비교 글
- **Firecracker — Lightweight Virtualization for Serverless Applications** (paper)
- **gVisor** docs
- **Kata Containers** docs

---

## 14. 관련

- [[vm-hypervisor]]
- [[container]]
- [[namespace]]
- [[cgroups]]
- [[virtualization]] — Virt hub
