---
title: "VM / Hypervisor — Type 1 / Type 2 / Light VM"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T14:55:00+09:00
tags:
  - operating-system
  - virtualization
  - vm
---

# VM / Hypervisor

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | Type 1 / 2 / Light VM |

**[[virtualization|↑ Virt hub]]**

---

## 1. Type 1 (Bare-metal) vs Type 2 (Hosted)

```
Type 1 (Hypervisor as OS):
  Hardware → Hypervisor → Guest OS
  예: VMware ESXi, Xen, Hyper-V, Proxmox

Type 2 (Hypervisor as Process):
  Hardware → Host OS → Hypervisor → Guest OS
  예: VirtualBox, VMware Workstation, Parallels
```

KVM 은 모호 — Linux 커널 모듈이지만 사실상 Type 1 (Linux 가 hypervisor 역할).

---

## 2. 가상화 종류

| 종류 | 의미 |
| --- | --- |
| **Full virtualization** | 게스트 OS 그대로 (수정 X) |
| **Paravirtualization** | 게스트가 hypervisor 인식 (특수 driver) |
| **Hardware-assisted** | Intel VT-x / AMD-V — CPU 지원 |
| **OS-level** | 컨테이너 (다른 OS X) |

---

## 3. Intel VT-x / AMD-V

CPU 의 추가 mode:
```
Root (Host)     ← hypervisor
Non-root (Guest) ← VM
VMX transition (VMEXIT / VMENTRY)
```

→ 게스트의 sensitive 명령 (CR / I/O / interrupt) 이 VMEXIT 트리거 → hypervisor 처리.

EPT (Extended Page Table, Intel) / NPT (Nested Page Table, AMD):
- guest virtual → guest physical → host physical
- 2-level paging (HW 가속)

---

## 4. 주요 Hypervisor

| Hypervisor | 종류 | 비고 |
| --- | --- | --- |
| **KVM** | Linux kernel module | qemu 와 결합 — 가장 표준 |
| **Xen** | Type 1 | dom0 + domU |
| **VMware ESXi** | Type 1 | 상용 |
| **Hyper-V** | Type 1 | Windows |
| **Proxmox VE** | KVM + LXC | 무료 |
| **VirtualBox** | Type 2 | 데스크탑 |
| **VMware Workstation** | Type 2 | 데스크탑 상용 |
| **QEMU** | emulator | KVM 없으면 software emul |
| **Firecracker** | Type 1 (lightweight) | AWS Lambda |

자세히 → [[kvm]]

---

## 5. QEMU + KVM

```bash
# Linux 의 사실상 표준
sudo apt install qemu-kvm libvirt-daemon-system

qemu-system-x86_64 \
  -enable-kvm \
  -m 4G \
  -smp 4 \
  -cpu host \
  -drive file=disk.qcow2 \
  -netdev user,id=net0 -device virtio-net,netdev=net0
```

GUI: virt-manager / virsh.

---

## 6. 디바이스 가상화

### 6.1 Full emulation
실제 디바이스 (e1000 NIC, IDE disk) 흉내 — 게스트 driver 그대로. 느림.

### 6.2 Paravirtualized (virtio)
```
guest 의 virtio driver ↔ host 의 vhost
ring buffer 공유 → 빠름
```

- virtio-net (NIC)
- virtio-blk (디스크)
- virtio-scsi
- virtio-fs (파일 시스템)
- virtio-gpu

→ KVM / Xen 의 표준.

### 6.3 SR-IOV
NIC 가 자체로 virtual function 제공 → 게스트가 직접 접근.
HPC / NFV 에서 사용.

---

## 7. Light VM

VM 의 격리 + 컨테이너의 가벼움:

| 도구 | 특징 |
| --- | --- |
| **Firecracker** (AWS, Rust) | < 125 ms boot, 5 MB memory overhead — Lambda/Fargate |
| **Kata Containers** | OCI 호환 + 게스트 커널 — K8s 통합 |
| **gVisor** | 사용자 공간 syscall 인터셉터 (커널 흉내) — 격리 ↑ but overhead |
| **Cloud Hypervisor** | Rust, modern, K8s |

→ multi-tenant SaaS / FaaS.

---

## 8. Live Migration

```
running VM 을 다른 host 로 이동 (downtime 거의 0)
1. memory 페이지 복사 (pre-copy)
2. dirty page 재복사 반복 (iterations)
3. 짧은 stop + 최종 동기화 + 새 host 에서 시작
```

VMware vMotion / KVM `virsh migrate --live`.

---

## 9. 메모리 절약 기법

- **KSM** (Kernel Same-page Merging) — 같은 페이지 dedup
- **Ballooning** — guest 가 자기 안의 free page 를 host 에 반환
- **Memory overcommit**

---

## 10. CPU 가상화 모드

```
# qemu cpu 옵션
-cpu host           # host 그대로 — 같은 host 만 호환
-cpu kvm64          # 보수 — 마이그 호환
-cpu Skylake-Server # 특정 모델 emulate
```

→ 마이그레이션 / 호환성 trade-off.

---

## 11. Nested Virtualization

```
host KVM
  guest KVM (nested)
    guest-of-guest VM
```

Intel/AMD CPU 의 지원 + flag (`kvm-intel.nested=1`).
CI / 개발 / 학습용. 성능 ↓.

---

## 12. Cloud 의 VM

| Cloud | VM |
| --- | --- |
| AWS EC2 | Xen (옛) → Nitro KVM (현재) |
| Google Cloud | KVM |
| Azure | Hyper-V |
| Oracle Cloud | KVM + Xen |
| 옛 OpenStack | KVM + Xen |

거의 모두 KVM 기반.

---

## 13. 보안

VM 의 격리는 강하지만:
- **Side channel** — Spectre / Meltdown / L1TF / MDS
- **VMEscape** — guest → host (CVE 있었음 — Venom, ...)
- **Hypervisor 자체 취약점** — 위험

→ 매우 민감한 워크로드 = baremetal 또는 dedicated host.

---

## 14. 컨테이너 안의 VM

- Kata Containers 는 K8s 안에서 light VM
- Docker on Mac/Windows = Linux VM + 컨테이너

---

## 15. 함정

### 15.1 -cpu host + 마이그
다른 CPU host 로 이동 X. baseline CPU 모델 권장.

### 15.2 nested 성능
~ 20-50% 손해.

### 15.3 KSM 의 side channel
같은 페이지 dedup → timing attack 위험. multi-tenant 비활성.

### 15.4 disk image 의 fsync
qcow2 의 `cache=writeback` = 호스트 crash 시 손실.

### 15.5 SR-IOV + live migration
지원 X (옛). 최근 일부 가능.

### 15.6 ballooning 의 OOM
guest 가 너무 reclaim → OOM. 모니터링.

---

## 16. 학습 자료

- **KVM** kernel.org docs
- **Firecracker** github
- **Hardware-Assisted Virtualization** — Intel SDM Vol. 3 Ch. 24-30
- **Xen Hypervisor** docs
- **Brendan Gregg — Virtualization Performance**

---

## 17. 관련

- [[kvm]]
- [[vm-vs-container]]
- [[virtualization]] — Virt hub
