---
title: "DMA 와 IOMMU"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, bus, dma, iommu, vt-d, msi, msi-x, napi, interrupt]
---

# DMA 와 IOMMU

**[[bus-io|↑ 버스·I/O]]**

## 1. DMA — CPU 거치지 않는 데이터 이동

### 왜 필요한가
- 디스크 / NIC / GPU 가 데이터를 메모리에 옮길 때 CPU 가 직접 byte 단위로 복사하면 비효율.
- **장치가 메모리에 직접 read/write 하고 완료만 CPU 에 인터럽트.**

### 동작
1. 드라이버가 장치에 "이 메모리 주소 / 길이로 데이터 옮겨" 라고 명령.
2. 장치의 DMA 엔진이 메모리 컨트롤러와 직접 통신.
3. 완료 시 장치가 인터럽트 발생.
4. 드라이버가 받아서 OS / 응용에 전달.

### scatter-gather

연속하지 않은 메모리 페이지들의 리스트로 한 번에 전송. 대용량 IO 의 표준.

## 2. IOMMU

장치가 메모리에 무한히 접근하면 보안·안정성 문제:

- 버그 / 악성 펌웨어가 임의 메모리 덮어쓰기.
- VM 에 PCIe 장치를 노출 (passthrough) 하면 게스트가 호스트 메모리 접근 위험.

→ **IOMMU (I/O Memory Management Unit)** 이 장치의 메모리 접근을 가상 → 물리 매핑 + 권한 검사로 격리.

| 이름 | 출처 |
| --- | --- |
| **VT-d** | Intel |
| **AMD-Vi** | AMD |
| **SMMU** | ARM |

### 효용
1. **DMA 격리** — 장치 A 가 장치 B 의 버퍼 접근 못 함.
2. **VM passthrough** — VM 에 PCIe 장치 직접 노출 (SR-IOV, GPU passthrough).
3. **DMA Remapping** — 32-bit 장치가 4 GiB 이상 메모리 사용 가능.
4. **Page fault on device** — Shared Virtual Memory (SVM, PCIe ATS+PRI).

### 활성화

```bash
# Linux 부팅 옵션
intel_iommu=on iommu=pt       # Intel
amd_iommu=on iommu=pt         # AMD

# 확인
dmesg | grep -i 'IOMMU\|DMAR'
ls /sys/kernel/iommu_groups/
```

### IOMMU 그룹

- 같은 그룹 안 장치들은 함께만 passthrough 가능.
- 보드의 PCIe 토폴로지 / ACS (Access Control Services) 가 그룹 경계를 결정.
- "ACS override" 패치로 강제 분리 가능 (홈랩에서 흔히 사용, 보안 trade-off).

## 3. 인터럽트 — 진화

### 옛 핀 IRQ (PIC / APIC)

- 별도 IRQ 핀 / line. 4-16 개 제한.
- 라우팅 + share 가 복잡.

### MSI (Message Signaled Interrupt)

- PCIe 패킷 자체로 인터럽트 전달 (특별 메모리 주소로 write).
- 칩셋이 라우팅. 핀 없이 vector 수십 개.

### MSI-X

- 장치당 최대 **2048 vectors**.
- 멀티 큐 NIC / NVMe 의 필수 조건 — 각 큐가 자기 CPU 코어로 인터럽트.

```bash
# 인터럽트 분포 확인
cat /proc/interrupts | head -20
cat /proc/interrupts | grep -E 'mlx|nvme|eth'
```

## 4. NAPI — Linux 네트워크 핵심

### 문제
- 10 GbE NIC = 14.88 Mpps (64-byte 패킷). 인터럽트 당 1 패킷 처리하면 14.88 M IRQ/s → CPU 폭주.

### 해법
- **NAPI** = 인터럽트 + polling 의 하이브리드.
- 첫 인터럽트로 깨고 → 일정 budget (보통 64 패킷) 까지 polling 으로 묶어 처리 → 다시 인터럽트 활성화.
- 패킷 폭주 시 자연스럽게 polling 모드로 전환.

코드: `napi_schedule()` / `napi_complete_done()`.

## 5. Interrupt Coalescing

- 인터럽트를 일정 시간 / 일정 패킷 수 모아서 한 번에.
- ethtool 의 `rx-usecs` / `rx-frames` 옵션.
- latency 와 CPU 부담 사이 trade-off.

```bash
sudo ethtool -c eth0
sudo ethtool -C eth0 rx-usecs 50 rx-frames 32
```

## 6. RSS (Receive Side Scaling)

- 들어오는 패킷의 5-tuple (src/dst IP, port, proto) 을 hash 해 여러 CPU 큐로 분산.
- 한 NIC 의 트래픽을 8/16/32 코어에 자동 분배.
- 같은 flow 는 같은 큐 → cache 친화.

```bash
sudo ethtool -l eth0   # 큐 수
sudo ethtool -x eth0   # RSS hash 매핑
```

## 7. SR-IOV (Single Root I/O Virtualization)

PCIe 장치 1 개를 여러 가상 PCIe 장치 (VF = Virtual Function) 로 분할.

- 1 개 PF (Physical Function) 가 NIC / 가속기 본체.
- N 개 VF 가 각 VM 에 직접 할당.
- IOMMU 가 VF 사이 격리.
- VF 트래픽은 NIC 의 embedded switch 가 라우팅.

서버 NIC (ConnectX, Intel E810), 일부 GPU (NVIDIA vGPU), NVMe SSD 가 지원.

## 8. 함정

1. **IOMMU 비활성 채 VM passthrough** — DMA attack 가능. BIOS 에서 VT-d / AMD-Vi 활성 필수.
2. **MSI-X vector 부족** — NIC 큐가 16 인데 보드가 MSI-X 만 8 개 지원 → 큐 일부가 vector 공유 → 인터럽트 storming.
3. **인터럽트 affinity 미설정** — 모든 인터럽트가 CPU0 으로 → core 1 개 100%. `irqbalance` 또는 수동 `/proc/irq/X/smp_affinity`.
4. **coalescing 너무 공격적** — latency 민감한 응용 (HFT / 게임) 에서 100 μs 지연 발생.
5. **iommu_group 같은 장치 한 묶음** — VM 에 GPU passthrough 하려는데 같은 그룹에 USB 컨트롤러 → USB 도 같이 빼야 함.

## 9. 관련

- [[bus-io]]
- [[pcie]]
- [[../network-hardware/nic-offload]] — RSS / NAPI 의 NIC 면
- [[../../operating-system/operating-system]] — 인터럽트 핸들링 / 드라이버 모델
