---
title: "KVM — Kernel-based Virtual Machine"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T15:00:00+09:00
tags:
  - operating-system
  - virtualization
  - kvm
---

# KVM

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | KVM 구조 / 운영 |

**[[virtualization|↑ Virt hub]]**

---

## 1. 한 줄

Linux 커널이 **hypervisor 가 되는 모듈** (`kvm.ko`). 게스트 VM 은 Linux 의 process 처럼 보임 + Intel VT-x / AMD-V 활용.

Avi Kivity, 2007. 2.6.20 부터 mainline.

---

## 2. 구조

```
QEMU (user space)
  ↓ ioctl(/dev/kvm)
KVM (kernel module)
  ↓
VT-x / AMD-V (hardware)
```

- KVM = CPU / memory 가상화 (커널)
- QEMU = device emulation (user)
- libvirt / virt-manager = 관리 layer

---

## 3. 한 VM = 한 QEMU 프로세스

```bash
ps aux | grep qemu
# qemu-system-x86_64 -enable-kvm -m 4G -smp 4 ...
```

- 각 VM = 호스트 OS 의 1 process
- vCPU = qemu 의 thread
- 호스트 scheduler 가 vCPU 스케줄

→ `top` / `pidstat` 으로 vCPU 부하 보임.

---

## 4. /dev/kvm

```bash
ls -l /dev/kvm
# crw-rw---- 1 root kvm 10, 232

# 사용자 권한
sudo usermod -aG kvm $USER

# 가상화 가능 여부
egrep -c '(vmx|svm)' /proc/cpuinfo
# > 0 면 VT-x / AMD-V 지원
```

---

## 5. virsh / libvirt

```bash
# domain (VM) 관리
virsh list --all
virsh start myvm
virsh shutdown myvm
virsh destroy myvm           # 강제
virsh dumpxml myvm > myvm.xml
virsh define myvm.xml
virsh edit myvm

# snapshot
virsh snapshot-create-as myvm snap1
virsh snapshot-list myvm
virsh snapshot-revert myvm snap1

# live migration
virsh migrate --live myvm qemu+ssh://target/system
```

XML 정의 — CPU / memory / disk / network 명시.

---

## 6. 디스크 포맷

| 포맷 | 의미 |
| --- | --- |
| `raw` | 그대로 — 빠름, snapshot X |
| `qcow2` | QEMU Copy-on-Write 2 — snapshot + thin |
| `vmdk` | VMware |
| `vdi` | VirtualBox |

```bash
qemu-img create -f qcow2 disk.qcow2 20G
qemu-img info disk.qcow2
qemu-img convert -O qcow2 src.vmdk dst.qcow2
qemu-img resize disk.qcow2 +10G
qemu-img snapshot -c snap1 disk.qcow2
```

---

## 7. 네트워크

### 7.1 user mode (slirp)
```
-netdev user,id=net0 -device virtio-net,netdev=net0
```
가장 단순, NAT, 외부 → guest 접근 X.

### 7.2 bridge
```
host 의 bridge (br0) 에 tap → guest
guest 가 LAN 의 1 호스트처럼
```

### 7.3 macvtap / VLAN / VXLAN
복잡 — 운영 / 컨테이너 비슷.

자세히 → [[../../network/network]]

---

## 8. virtio

paravirtualized 드라이버 — 게스트가 virtio 인식 → host 와 ring buffer 공유.

| 디바이스 | 의미 |
| --- | --- |
| virtio-net | NIC |
| virtio-blk | block 디바이스 |
| virtio-scsi | SCSI 디스크 |
| virtio-fs | host fs 공유 |
| virtio-gpu | GPU |
| virtio-rng | 난수 |
| virtio-balloon | 메모리 ballooning |

→ full emulation 보다 훨씬 빠름.

---

## 9. CPU pinning / NUMA

```xml
<vcpu placement='static' cpuset='0-3'>4</vcpu>
<numatune>
  <memory mode='strict' nodeset='0'/>
</numatune>
<cputune>
  <vcpupin vcpu='0' cpuset='0'/>
  <vcpupin vcpu='1' cpuset='1'/>
</cputune>
```

→ guest vCPU 를 host 의 같은 NUMA 노드에 pin → 캐시 / latency ↑.

자세히 → [[../memory/numa]]

---

## 10. SR-IOV / PCI passthrough

```
호스트의 GPU / NIC 를 guest 에 직접 부여
guest 가 native 성능
단, 그 동안 host 가 사용 X
```

VFIO 드라이버 사용. GPU passthrough for gaming / AI VM.

---

## 11. KSM (Kernel Same-page Merging)

```bash
cat /sys/kernel/mm/ksm/run            # 1 = on
cat /sys/kernel/mm/ksm/pages_shared    # 합쳐진 페이지 수
```

같은 내용의 페이지 dedup → memory overcommit 가능.

⚠️ side channel 위험 (multi-tenant).

---

## 12. Hugepages

```bash
echo 1024 > /proc/sys/vm/nr_hugepages
# qemu
-mem-path /dev/hugepages
```

VM 의 TLB miss ↓.

자세히 → [[../memory/huge-pages]]

---

## 13. 모니터링

```bash
# host 측
virsh dominfo myvm
virt-top
perf kvm stat

# guest 안
top / vmstat 일반과 같음

# /sys/kernel/debug/kvm
cat /sys/kernel/debug/kvm/halt_*
```

---

## 14. cgroup + KVM

VM 자원 제한:
```
libvirt 는 자동으로 systemd-machined / scope cgroup
machinectl list
systemctl status machine-qemu-...
```

cgroup memory.max, cpu.max 등 설정 가능.

---

## 15. 클라우드 — Nitro / Compute Engine

| Cloud | 변형 |
| --- | --- |
| AWS Nitro | KVM 기반 + 전용 카드 (network / EBS offload) |
| GCP | KVM + virtio |
| Linode / DigitalOcean | KVM |
| Azure | Hyper-V (KVM 아님) |

AWS Nitro 의 Nitro Card = NIC + EBS + 보안을 KVM 외부 카드로 → host CPU 부담 ↓.

---

## 16. AWS Firecracker

KVM 기반 minimal VMM:
- < 5 MB binary
- < 125 ms boot
- 5 MB overhead per VM
- AWS Lambda / Fargate

자세히 → [[vm-hypervisor#7-light-vm]]

---

## 17. 함정

### 17.1 -enable-kvm 누락
software emulation — 10-100x 느림.

### 17.2 -cpu host + 마이그
다른 CPU 호스트로 이동 X.

### 17.3 disk cache mode
`cache=writeback` = 빠름, crash 시 손실. `none` (O_DIRECT) 또는 `writethrough`.

### 17.4 KSM + side channel
multi-tenant 에선 비활성.

### 17.5 큰 memory + ballooning
guest OOM 위험.

### 17.6 nested KVM 성능
20-50% 손해.

### 17.7 GPU passthrough + reboot
guest 가 GPU reset 안 함 → host 도 hang. reset patch / mdev 검토.

### 17.8 libvirt + 직접 qemu 혼용
누가 관리하는지 혼동.

---

## 18. 학습 자료

- **kernel.org** `Documentation/virt/kvm/`
- **QEMU docs** — wiki.qemu.org
- **libvirt docs** — libvirt.org
- **Performance Tuning Guide** — Red Hat
- **AWS Nitro 시스템** 발표

---

## 19. 관련

- [[vm-hypervisor]]
- [[namespace]]
- [[../memory/numa]]
- [[../memory/huge-pages]]
- [[virtualization]] — Virt hub
