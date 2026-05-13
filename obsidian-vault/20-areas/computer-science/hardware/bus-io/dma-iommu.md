---
title: "DMA 와 IOMMU"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, bus, dma, iommu, vt-d, amd-vi, smmu, msi, msi-x, napi, sr-iov, ats, pasid]
---

# DMA 와 IOMMU

**[[bus-io|↑ 버스·I/O]]**

> CPU 없이 장치가 메모리에 직접 R/W — 현대 IO 처리량의 비밀. 그 권한을 격리하는 IOMMU 가 VM passthrough / 보안의 핵심.

## 1. DMA — CPU 없는 데이터 이동

### 옛 PIO (Programmed IO)
- CPU 가 `IN`/`OUT` 명령으로 한 word 씩 옮김.
- 1 MB 옮기는 데 CPU 가 수십만 사이클 점유.

### DMA
- 장치에 "이 메모리 region 에 N byte 옮겨" 명령.
- 장치의 DMA 엔진이 메모리 컨트롤러와 직접 통신.
- CPU 는 명령 후 다른 일.
- 완료 시 인터럽트.

```
   CPU ──cmd──► Disk Controller
                       │
                       ▼
                   DMA Engine
                       │
                  R/W   │   read/write
                       │
                       ▼
                 Main Memory ─────► (CPU 도 여기 캐시-일관성으로 보유)
                       │
                       ▼
                  완료 → CPU IRQ
```

### Bus Master DMA
- 장치가 직접 bus 의 master 가 되어 메모리에 read/write.
- 옛 ISA 는 별도 DMA controller (8237) 가 중재. PCIe 는 모든 장치가 bus master 가능.

## 2. DMA 의 종류

| 종류 | 의미 | 사용처 |
| --- | --- | --- |
| **Contiguous DMA** | 연속된 물리 메모리 한 chunk | 옛 / 단순한 장치 |
| **Scatter-Gather DMA** | 여러 chunk 의 리스트로 한 번에 전송 | 모든 현대 NIC/SSD |
| **Coherent DMA** | CPU 캐시와 자동 일관성 | x86 기본 |
| **Non-coherent DMA** | 드라이버가 직접 cache flush/invalidate | 일부 ARM 임베디드 |

### Scatter-Gather 의 중요성

- 사용자 공간의 큰 read/write 는 가상 메모리상 연속이지만 **물리 메모리는 분산** (페이지 단위).
- DMA 가 가상 → 물리 mapping 없이 처리하려면 N 개 chunk 로 분산 전송 필요.
- 네트워크 스택 / 디스크 IO 가 모두 scatter-gather.

## 3. DMA 의 위험

장치가 메모리에 무한 접근 = 심각한 보안 / 안정성 문제.

### 구체 위험
1. **Buggy/Malicious 펌웨어** — NIC 펌웨어가 임의 메모리 영역 dump.
2. **DMA Attack** (FireWire / Thunderbolt 등) — 외부 장치가 메모리 dump.
3. **VM passthrough 시 게스트가 호스트 메모리 접근**.
4. **32-bit 장치** — 4 GiB 이상 메모리 못 봄.
5. **버그가 다른 장치 buffer 덮어쓰기**.

→ 이를 막는 게 **IOMMU**.

## 4. IOMMU — I/O Memory Management Unit

장치의 메모리 접근을 가상 → 물리 매핑 + 권한 검사로 격리.

```
     Device ──(IOVA)──► IOMMU ──(check perm + translate)──► Physical Memory
                          │
                          ├── IOTLB (cache)
                          └── 결과: PA 또는 fault
```

- **IOVA (IO Virtual Address)**: 장치가 보는 가상 주소.
- **PA (Physical Address)**: 메모리 컨트롤러가 보는 실 주소.
- IOMMU 가 page table 로 매핑 + permission 확인.

| 제조사 | 이름 |
| --- | --- |
| Intel | VT-d (Virtualization Technology for Directed I/O) |
| AMD | AMD-Vi (구 IOMMU) |
| ARM | SMMU (System Memory Management Unit) |

### IOMMU 의 효용

1. **DMA 격리** — 장치 A 가 장치 B 의 buffer 못 봄.
2. **VM passthrough** — VM 에 PCIe 장치 직접 노출 (SR-IOV, GPU passthrough).
3. **DMA Remapping** — 32-bit 장치가 4 GiB+ 메모리 사용 가능.
4. **Page fault on device** — Shared Virtual Memory (ATS+PRI).
5. **Interrupt remapping** — 장치가 임의 인터럽트 발행 못 함.

## 5. IOMMU Page Table

- CPU MMU 의 page table 과 비슷한 구조 (4-level 또는 5-level).
- 가상 주소를 4 KB / 2 MB / 1 GB page 단위로 매핑.
- **IOTLB** (IO Translation Lookaside Buffer) 가 최근 매핑 캐시.

### IOMMU domain
- 한 IOMMU domain = 하나의 page table.
- 같은 domain 의 장치들은 같은 IOVA → PA 매핑 공유.
- 다른 domain 끼리는 서로 격리.

### Linux 의 IOMMU domain
- 보통 1 PCI 장치당 1 domain (full isolation).
- VM 의 모든 passthrough 장치는 그 VM 의 domain.
- 동시에 여러 GPU 를 passthrough 하면 한 domain 또는 multiple.

## 6. IOMMU 활성화 — Linux

```bash
# 1. BIOS / UEFI 에서 VT-d / AMD-Vi 활성
# 2. 부팅 옵션 (/etc/default/grub)
GRUB_CMDLINE_LINUX="intel_iommu=on iommu=pt"      # Intel
GRUB_CMDLINE_LINUX="amd_iommu=on iommu=pt"        # AMD

# 3. 확인
sudo dmesg | grep -i 'IOMMU\|DMAR'
# 예: DMAR: IOMMU enabled
#     DMAR: Translation was enabled for IOMMU 0
ls /sys/kernel/iommu_groups/                      # 0/, 1/, 2/, ...
ls /sys/kernel/iommu_groups/0/devices/            # 그 group 의 PCI 장치들
```

### `iommu=pt` (passthrough mode)
- "장치가 호스트 자체에서 쓰는 동안은 IOMMU translation off, VM 에 넘기는 순간 on".
- 호스트 IO 성능 영향 최소화.
- 보안 측면에서 호스트 자체 장치는 신뢰한다는 가정.

## 7. IOMMU 그룹 — VM Passthrough 의 단위

같은 IOMMU group 안 장치들은 **함께만 passthrough 가능**.

- 보드의 PCIe 토폴로지 + ACS (Access Control Services) capability 가 group 경계 결정.
- 일부 보드는 한 group 에 GPU + 사운드 + USB 컨트롤러 모두 묶임 — 한 VM 에 다 같이 넘겨야 함.

### `pci=ACS_override` (홈랩 트릭)
- 강제로 group 을 더 잘게 쪼개는 커널 패치.
- 메인보드 ACS support 안 해도 GPU 만 따로 passthrough 가능.
- **보안 trade-off**: DMA 격리가 실제로 보장 안 됨.
- 운영 환경에서는 사용 금지.

```bash
GRUB_CMDLINE_LINUX="... pci=ACS_override=downstream,multifunction"
```

## 8. SR-IOV (Single Root I/O Virtualization)

PCIe 장치 1 개를 여러 가상 PCIe 장치로 분할.

```
   Physical NIC (PF)
        │
        ├── VF 1 ──── IOMMU domain A ──── VM 1
        ├── VF 2 ──── IOMMU domain B ──── VM 2
        ├── VF 3 ──── IOMMU domain C ──── VM 3
        └── VF n ──── IOMMU domain D ──── VM n
```

| Function | 의미 |
| --- | --- |
| **PF** (Physical Function) | NIC / 가속기 본체. 드라이버가 관리. |
| **VF** (Virtual Function) | 가상. 각 VM 에 IOMMU 로 격리 후 노출. |

- NIC 안 embedded switch 가 VF 사이 트래픽 라우팅 + L2 forwarding.
- 서버 NIC (ConnectX, Intel E810), 일부 GPU (NVIDIA vGPU), NVMe SSD 가 지원.

### 활용
- 클라우드: EC2 의 SR-IOV (ENA) 가 가상 NIC 의 line rate.
- 데이터센터: 각 VM 이 독립 NIC 큐 + IRQ.
- AI 학습: GPU 의 vGPU 분할.

```bash
echo 4 > /sys/class/net/eth0/device/sriov_numvfs    # VF 4 개 생성
ip link show eth0
```

## 9. 인터럽트 진화 — Pin → MSI → MSI-X

### Pin IRQ (PIC / APIC)
- 별도 IRQ 핀 / line. 4-16 개 한계.
- 라우팅 + share 가 복잡.
- 옛 ISA / PCI legacy.

### MSI (Message Signaled Interrupt)
- PCIe TLP 의 Memory Write 로 인터럽트 전달.
- 특별 주소 (예: 0xFEE00000 on x86) 에 write → APIC 가 vector 로 변환.
- 핀 없이 vector 수십 개.
- 한 device 당 최대 32 vector.

### MSI-X
- 한 device 당 **최대 2048 vector**.
- 큐별 / CPU 코어별 인터럽트 분산 가능.
- **현대 멀티 큐 NIC / NVMe SSD 의 필수 조건**.

```bash
# 장치별 MSI/MSI-X 사용 확인
sudo lspci -vvv -s 01:00.0 | grep -E 'MSI|MSI-X'

# 시스템 전체 인터럽트 카운터
cat /proc/interrupts | head -20
cat /proc/interrupts | grep -E 'mlx|nvme|eth'
```

### Interrupt Remapping (IOMMU)
- IOMMU 가 인터럽트도 가상 → 실 vector 로 매핑.
- 장치가 임의 vector 로 인터럽트 발행하는 공격 방어.
- VM 의 vector 와 호스트 vector 분리.

## 10. NAPI — Linux 네트워크의 핵심

### 문제
- 10 GbE = 14.88 Mpps (64-byte 패킷).
- 1 패킷 = 1 인터럽트 → 14.88 M IRQ/s → CPU 폭주 (interrupt storm).

### NAPI 의 해법
- 첫 인터럽트로 깨고 → polling 으로 일정 budget 까지 (보통 64 패킷) 묶어 처리 → 다시 인터럽트 활성화.
- 패킷 폭주 시 자연스럽게 polling 모드, 한가하면 interrupt 모드.

### 코드 흐름
```c
napi_schedule(&napi);        // 첫 IRQ 핸들러가 호출
int poll(napi, budget) {
    int processed = 0;
    while (packet_available && processed < budget) {
        process_packet();
        processed++;
    }
    if (processed < budget) {
        napi_complete_done(napi, processed);    // IRQ 활성
    }
    return processed;
}
```

## 11. Interrupt Coalescing

- 일정 시간 / 일정 패킷 수 모아서 한 인터럽트로.
- ethtool 의 `rx-usecs` / `rx-frames`.

```bash
sudo ethtool -c eth0
# Default coalescing parameters:
# Adaptive RX: on  TX: on
# rx-usecs: 8
# rx-frames: 32
# tx-usecs: 8
# tx-frames: 32

sudo ethtool -C eth0 rx-usecs 50 rx-frames 32     # 더 모아서
sudo ethtool -C eth0 adaptive-rx off              # 고정
```

### Adaptive coalescing
- NIC 가 트래픽 양에 따라 자동 조절.
- 낮은 트래픽 = 작은 coalescing (latency 우선).
- 높은 트래픽 = 큰 coalescing (throughput 우선).

## 12. RSS / RPS / RFS / aRFS — 큐 분산

### RSS (Receive Side Scaling)
- 들어오는 패킷의 5-tuple (src IP / dst IP / src port / dst port / proto) 해시.
- 해시 → 큐 인덱스 → CPU 코어.
- 같은 flow 는 같은 큐 → cache 친화.
- NIC HW 에서 처리.

### RPS (Receive Packet Steering)
- RSS 의 소프트웨어 버전. NIC 가 RSS 미지원 시.
- 첫 IRQ 후 OS 가 hash 로 다른 코어에 패킷 redirect.

### RFS (Receive Flow Steering)
- flow 가 어느 CPU 에서 처리되는지 추적해서 다음 패킷을 그 코어로.
- cache locality 추가 확보.

### aRFS (Accelerated RFS)
- aRFS = HW (NIC) 가 ntuple filter 로 flow → 큐 매핑.
- 가장 효율.

```bash
sudo ethtool -l eth0                    # 최대 큐 수, 현재 큐 수
sudo ethtool -L eth0 combined 16         # 큐 수 변경
sudo ethtool -x eth0                    # RSS indirection table

# RFS 활성
echo 32768 > /proc/sys/net/core/rps_sock_flow_entries
echo 4096  > /sys/class/net/eth0/queues/rx-0/rps_flow_cnt
```

## 13. IRQ Affinity

여러 CPU 가 모든 인터럽트를 다 받으면 cache contention. 인터럽트를 특정 CPU 로 고정.

### 자동 — irqbalance
- daemon 이 부하 균등화.
- 일반적 환경에 적합. 일부 high-perf 워크로드는 수동 fix 가 더 좋음.

### 수동
```bash
# CPU 마스크 (16 진수) — 큐 0 을 CPU 1 에 고정
echo 2 | sudo tee /proc/irq/123/smp_affinity

# 또는 list
echo 0-3 | sudo tee /proc/irq/123/smp_affinity_list
```

### NIC 의 자동 affinity 스크립트
- 보통 NIC 드라이버가 `set_irq_affinity.sh` 또는 systemd unit 제공.
- 16-큐 NIC + 32 코어 → 큐 i 를 CPU i+0 에 고정.

## 14. ATS / PRI / PASID — Shared Virtual Memory (SVM)

기존 IOMMU 의 한계: 장치가 메모리 접근 전 IOVA 매핑을 미리 등록 + pin 해야 함.

### ATS (Address Translation Services)
- 장치가 IOMMU 의 IOTLB 를 캐시.
- 같은 page 반복 접근 시 IOMMU walk 안 함.

### PRI (Page Request Interface)
- 장치가 PA 모르는 IOVA 접근 → 페이지 fault 발생 → OS 가 페이지 가져오기 → 장치에 응답.
- swap-out 된 페이지 / unmapped 페이지 처리.

### PASID (Process Address Space ID)
- 같은 장치를 여러 프로세스가 격리해서 공유.
- 각 process 의 가상 메모리를 그대로 GPU/가속기에 노출.
- **SVM (Shared Virtual Memory)** 의 기반 — CUDA Unified Memory, ROCm HMM.

```
   Process A 의 page table ←──┐
   Process B 의 page table ←──┤
   Process C 의 page table ←──┤
                              │
                          IOMMU + PASID
                              │
                          GPU / 가속기 (PASID 별 격리)
```

## 15. Linux IOMMU API / VFIO

### VFIO (Virtual Function I/O)
- 사용자 공간에서 IOMMU + PCIe 장치 직접 접근.
- QEMU/KVM 의 device passthrough 기반.
- DPDK 의 NIC 직접 접근에도 사용.

```bash
# VFIO 활성
modprobe vfio-pci
echo "10de 1b80" > /sys/bus/pci/drivers/vfio-pci/new_id

# 장치 unbind from kernel driver, bind to vfio-pci
echo "0000:01:00.0" > /sys/bus/pci/devices/0000:01:00.0/driver/unbind
echo "0000:01:00.0" > /sys/bus/pci/drivers/vfio-pci/bind

# 그 다음 QEMU 의 -device vfio-pci,host=01:00.0
```

## 16. 진단 / 디버깅

```bash
# IOMMU 상태
sudo dmesg | grep -i iommu
ls /sys/kernel/iommu_groups/

# 각 group 의 장치
for g in /sys/kernel/iommu_groups/*; do
  echo "Group ${g##*/}:"
  ls $g/devices/
done

# 장치 별 IOMMU 사용 확인
sudo dmesg | grep "0000:01:00.0"

# DMA 통계 (일부 드라이버)
sudo cat /proc/iomem | head
cat /proc/dma          # legacy ISA DMA

# 인터럽트
cat /proc/interrupts | column -t | head -30
cat /proc/softirqs

# Network queue / RSS
sudo ethtool -l eth0
sudo ethtool -x eth0
sudo ethtool -S eth0 | grep -i rx_drop
```

## 17. 함정

1. **IOMMU 비활성 채 VM passthrough** — DMA attack 가능. 부팅 옵션 + BIOS 양쪽 확인.
2. **MSI-X vector 부족** — NIC 큐 수 vs vector 수 부족 → 큐 일부가 vector 공유 → interrupt storming.
3. **인터럽트 affinity 미설정** — 모든 인터럽트가 CPU0 → core 1 개 100%. `irqbalance` 또는 수동.
4. **coalescing 너무 공격적** — latency 민감 응용 (HFT, 게임) 에서 100 μs 추가 지연.
5. **iommu_group 같은 장치 한 묶음** — VM 에 GPU passthrough 시 그 group 안 USB / 사운드 도 같이 빼야 함.
6. **`pci=ACS_override` 운영 환경 사용** — DMA 격리가 사실상 깨짐. 보안 위험.
7. **RSS 미지원 NIC + 100 GbE** — 단일 코어 100%, 전체 NIC drop.
8. **NUMA + IOMMU group** — passthrough 장치가 cross-NUMA 메모리 사용 시 throughput 폭락.
9. **AF_XDP / DPDK 의 NIC 점유** — host 의 ping 도 못 나감. 운영 / 디버깅 시 회수 절차.
10. **VFIO 이후 host 가 장치 못 봄** — passthrough VM 종료 후 rebind 필요.

## 18. 관련

- [[bus-io]]
- [[pcie]] — MSI/MSI-X / capability / config space
- [[../network-hardware/nic-offload]] — RSS / NAPI / RFS
- [[../soc/datacenter-accelerators]] — GPU passthrough, SR-IOV vGPU
- [[../../operating-system/operating-system]] — IRQ handler, top-half / bottom-half, softirq
- [[../../security-theory/security-theory]] — DMA attack, Thunderspy, IOMMU bypass
