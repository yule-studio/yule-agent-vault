---
title: "스토리지 인터페이스 (SATA / SAS / NVMe / EDSFF)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, storage, sata, sas, nvme, m.2, u.2, edsff, nvme-of, scsi, ahci, scm]
---

# 스토리지 인터페이스

**[[storage|↑ 저장 장치]]**

## 1. 프로토콜 비교

| 프로토콜 | 위에 | 큐 (max) | 속도 | 듀얼 포트 | 명령셋 |
| --- | --- | --- | --- | --- | --- |
| **SATA** | AHCI | 1 × 32 | 6 Gbps | ✗ | ATA |
| **SAS** | SCSI | 1 × 256 (이론 65K) | 12 / 24 Gbps | ✓ | SCSI |
| **NVMe** | PCIe x1/x2/x4/x8 | 64K × 64K | PCIe gen × lane | ✓ (U.2/U.3) | NVMe |
| **NVMe-oF** | TCP / RDMA / FC | NVMe over fabric | 100 GbE+ | ✓ | NVMe |
| **eSATA** | SATA 외부 | 1 × 32 | 6 Gbps | ✗ | ATA |
| **Thunderbolt enclosure** | NVMe over PCIe tunnel | NVMe 동일 | 40-80 Gbps | ✗ | NVMe |

## 2. SATA / AHCI — 데스크탑 표준

### SATA 진화

| 세대 | 출시 | 속도 |
| --- | --- | --- |
| SATA 1.0 | 2003 | 1.5 Gbps |
| SATA 2.0 | 2004 | 3 Gbps |
| **SATA 3.0** | 2009 | **6 Gbps** — 현 표준 |
| SATA 3.1 | 2011 | 6 Gbps + improvements |
| SATA Express | 2013 | 16 Gbps — 실패, NVMe 가 대체 |

6 Gbps SATA = 실 최대 ~550 MB/s (8b/10b 인코딩 overhead 후).

### AHCI (Advanced Host Controller Interface)
- 2004 Intel.
- legacy IDE 의 후속.
- SATA 컨트롤러의 표준 인터페이스.
- **하나의 명령 큐, 깊이 32** — NCQ 로 명령 재정렬.

### NCQ (Native Command Queuing)
- 호스트가 디스크에 32 개 명령 한 번에 전달.
- 디스크 / FTL 이 자체 최적화 (HDD 는 seek 거리 최소화, SSD 는 채널 분산).

### SATA 의 한계
- AHCI 가 단일 큐 기반 → 멀티 코어 IO 에 락 경합.
- 6 Gbps 가 SSD 의 NAND BW 의 1/10 도 안 됨.
- → NVMe 로 산업 전환.

## 3. SAS / SCSI — 서버 표준

### SAS 진화

| 세대 | 출시 | 속도 |
| --- | --- | --- |
| SAS-1 | 2004 | 3 Gbps |
| SAS-2 | 2009 | 6 Gbps |
| SAS-3 | 2013 | 12 Gbps |
| **SAS-4** | 2017 | **24 Gbps** |
| SAS-5 | (개발 중단 — NVMe 대체) | — |

### SCSI 명령셋
- 매우 풍부한 기능: reservation, persistent reservation, T10 PI (Protection Information), copy offload.
- 엔터프라이즈 storage array 의 표준.

### SAS 의 강점

1. **듀얼 포트**: 한 디스크에 2 개 컨트롤러 동시 연결. 한 컨트롤러 fail 시 자동 페일오버.
2. **deeper queue**: 명령 큐 깊이 256+ (이론 65K).
3. **expander chain**: 1 HBA 가 SAS expander 통해 수백 디스크 연결.
4. **T10 DIF/PI**: 데이터 무결성 metadata (8 byte) 가 디스크 → host 까지 동행.
5. **클러스터 공유**: SCSI Reservation 으로 공유 disk (옛 SAN).

### SAS HBA
- LSI / Broadcom MegaRAID (HW RAID) vs HBA (IT mode).
- IT mode = 디스크를 그냥 OS 에 노출. ZFS / Ceph 와 짝.
- IR mode = HW RAID. 컨트롤러가 RAID 직접 처리.

## 4. NVMe — 현 데이터센터 표준

### 역사
- 2011 NVMe 1.0.
- PCIe 위 비휘발성 메모리 전용 프로토콜.
- AHCI 의 한계 (1 큐, 32 깊이) 정면 돌파.

### 핵심 혁신
- **64K queues × 64K depth** (이론).
- 실 NVMe SSD = 8~128 큐.
- 멀티 코어 IO 가 락 없이 큐 분산.
- 결과: 100 만+ IOPS 가능.

### NVMe 명령 종류

| 종류 | 명령 예 |
| --- | --- |
| Admin queue | Identify, Create/Delete I/O Queue, Get Log, Format, Firmware Download |
| I/O queue | Read, Write, Flush, Compare, Write Zeroes, Dataset Management (TRIM), Copy, Reservation |

### NVMe 의 구조

```
       Host
       │
   ┌───┴────────────────────────────────┐
   │ Admin Queue                        │
   ├────────────────────────────────────┤
   │ I/O Queue 0  ↔ CPU 0               │
   ├────────────────────────────────────┤
   │ I/O Queue 1  ↔ CPU 1               │
   ├────────────────────────────────────┤
   │ ...                                │
   ├────────────────────────────────────┤
   │ I/O Queue N  ↔ CPU N               │
   └────────────────────────────────────┘
              │
              ▼
         PCIe x4 link
              │
              ▼
        NVMe SSD Controller
```

### NVMe 1.x → 2.x

| 버전 | 출시 | 주요 |
| --- | --- | --- |
| 1.0 | 2011 | 기본 |
| 1.1 | 2012 | Active power |
| 1.2 | 2014 | namespace |
| 1.3 | 2017 | Sanitize, telemetry |
| 1.4 | 2019 | I/O determinism |
| 2.0 | 2021 | 분리된 사양 (Base, Command Set, Transport) |
| 2.1 | 2024 | Live migration, computational storage |

### Namespace
- 한 NVMe SSD 안 여러 namespace = 가상 분리.
- 다른 namespace 끼리 OP / 보안 / 액세스 정책 다르게.
- 1 SSD 를 여러 응용에 SLA 별로 분할.

### NVM Sets / Endurance Group
- 여러 namespace 가 같은 NAND 자원 공유 vs 분리.
- 워크로드 격리.

## 5. NVMe 폼팩터 — 자세히

### 2.5" SATA
- legacy. 노트북 / 데스크탑.
- 7 mm / 9.5 mm 두께.

### M.2

| Module | 크기 (mm) | 용도 |
| --- | --- | --- |
| 2230 | 22 × 30 | 작은 노트북 (Surface, Steam Deck) |
| 2242 | 22 × 42 | 노트북 |
| 2260 | 22 × 60 | 일반 노트북 |
| **2280** | 22 × 80 | 데스크탑 / 노트북 표준 |
| 22110 | 22 × 110 | enterprise / 워크스테이션 |

### M.2 키 (핀 노치 위치)

| 키 | 핀 | 용도 |
| --- | --- | --- |
| **A** | 8 핀 | Wi-Fi/BT |
| **B** | 12 핀 | SATA, PCIe x2 |
| **E** | 8 핀 | Wi-Fi / Bluetooth |
| **M** | 12 핀 | NVMe (PCIe x4) — 가장 흔함 |
| **B+M** | dual | SATA 또는 PCIe x2 |

→ **M-key 슬롯이 PCIe x4 NVMe**. B-only 슬롯은 SATA 또는 PCIe x2 만.

### U.2 / U.3 / SFF-8639

- 2.5" 폼팩터 + 직접 PCIe x4.
- SFF-8639 커넥터 — SATA / SAS / NVMe 동시 지원.
- **U.2** — NVMe 또는 SAS.
- **U.3** — tri-mode (NVMe / SAS / SATA 같은 슬롯).
- hot-swap. enterprise.

### AIC (Add-in Card)

- PCIe x4 / x8 카드 폼팩터.
- M.2 폼팩터로 너무 작아 발열·전력 한계인 경우.
- 워크스테이션 / 일부 서버.

### EDSFF (Enterprise & Data center SSD Form Factor) — 새 표준

EDSFF 컨소시엄 (SNIA) 이 2021+ 데이터센터 표준화.

| 폼팩터 | 크기 (mm) | 두께 | 전력 | 용도 |
| --- | --- | --- | --- | --- |
| **E1.S** | 31.5 × 111.5 | 5.9 | 12 W | thin ruler. dense chassis (1U 에 ~32 drive) |
| E1.S | 31.5 × 111.5 | 9.5 | 16 W | |
| E1.S | 31.5 × 111.5 | 15 | 20 W | |
| E1.S | 31.5 × 111.5 | 25 | 25 W | |
| **E1.L** | 38.4 × 318.75 | 9.5 | 25 W | 긴 ruler. 대용량. |
| E1.L | 38.4 × 318.75 | 18 | 40 W | 큰 enterprise |
| **E3.S** | 76 × 112.75 | 7.5 | 25 W | 2U server hot-swap |
| E3.S 2T | 76 × 112.75 | 16.8 | 40 W | 2-times thick |
| E3.L | 76 × 142.2 | 7.5 / 16.8 | 40 W | 더 큰 용량 |

### EDSFF 의 장점
1. **더 많은 drive / U** (E1.S 는 1U 에 32 drive).
2. **더 나은 발열 처리** (수평 정렬, 풍압 효율 ↑).
3. **CXL 호환** (E3 는 CXL.mem 모듈 지원).
4. **표준 hot-swap connector** (CDFP).

## 6. 인터페이스 별 속도 vs SSD 실 BW

| 인터페이스 | 이론 BW | 실 SSD seq read | 실 random 4K |
| --- | --- | --- | --- |
| SATA 6 Gbps | 600 MB/s | 550 MB/s | 80-100K IOPS |
| SAS 12 Gbps | 1.2 GB/s | 1.1 GB/s | 200K IOPS |
| SAS 24 Gbps | 2.4 GB/s | 2.0 GB/s | 1M IOPS |
| PCIe Gen3 x4 | 4 GB/s | 3.5 GB/s | 700K IOPS |
| PCIe Gen4 x4 | 8 GB/s | 7 GB/s | 1M IOPS |
| PCIe Gen5 x4 | 16 GB/s | 14 GB/s | 2M+ IOPS |
| PCIe Gen5 x8 | 32 GB/s | 28 GB/s | 4M+ IOPS |

→ PCIe Gen 한 단 차이 = 대략 SSD 속도 2 배 차이.

## 7. NVMe-oF (Over Fabrics)

NVMe 명령을 네트워크 위로.

### 전송 종류

| 전송 | 비고 |
| --- | --- |
| **NVMe/TCP** | 가장 보편. 100 GbE+. 설치 단순. |
| **NVMe/RDMA (RoCE v2)** | 낮은 latency. RDMA NIC 필요. RoCE PFC/ECN 튜닝 필요. |
| **NVMe/RDMA (iWARP)** | TCP 기반 RDMA. PFC 안 필요. |
| **NVMe/FC** | Fibre Channel SAN. enterprise legacy. |

### Latency 비교 (1-side 4K read)

| 매체 | latency |
| --- | --- |
| local NVMe | ~80 μs |
| NVMe/TCP over 100 GbE | 100-150 μs |
| NVMe/RDMA over 100 GbE | 90-100 μs |
| iSCSI over 10 GbE | 300-500 μs |

→ NVMe-oF (특히 RDMA) 가 local NVMe 의 ~1.2× latency 로 원격 SSD 사용 가능.

### 활용 사례
1. **분리형 스토리지 (composable infrastructure)** — storage chassis 에 NVMe, compute 노드는 NIC 만.
2. **데이터센터 SSD pool**.
3. **AI 학습 dataset 공유 read**.
4. **컨테이너 / VM volume**.

### Linux nvme-cli

```bash
# Discovery
sudo nvme discover -t rdma -a 10.0.0.1 -s 4420
sudo nvme discover -t tcp -a 10.0.0.1 -s 4420

# Connect
sudo nvme connect -t rdma -n nqn.2024-01.com.example:disk1 -a 10.0.0.1 -s 4420
sudo nvme list

# 결과: /dev/nvme0n1 처럼 보임
# 원격 SSD 가 local NVMe device 로 보임 → 모든 NVMe tool 사용 가능

# Disconnect
sudo nvme disconnect -n nqn.2024-01.com.example:disk1
```

## 8. SCM (Storage Class Memory) — DRAM 과 SSD 사이

### Intel Optane DC PMM (단종)
- 3D XPoint 기술. PCM (Phase Change Memory).
- DRAM 보다 100x 비쌌지만 100x 큰 용량.
- 메모리 슬롯에 byte-addressable persistent memory.
- 2022 단종.

### Samsung Z-NAND / Toshiba XL-FLASH
- SLC 같은 fast NAND.
- DRAM 보다 10-100x 느리지만 NAND 보다 빠름.
- NVMe SSD 형태로 출시.

### CXL.mem 기반 미래
- DRAM 또는 PMM 을 CXL 위.
- 자세히: [[ddr-evolution]] §7.

## 9. 디스크 인증 / 무결성 — T10 DIF/PI

T10 Protection Information — SCSI 디스크의 데이터 무결성 메타데이터.

- 8 byte / 512 byte logical block 또는 4096 byte block.
- guard (CRC) + tag + reference tag.
- host 가 데이터 보낼 때 PI 도 같이 보내 → 디스크 controller 가 검증.
- 디스크 read 시 PI 동행 → host 검증.

→ 전체 IO path (host → HBA → switch → disk → NAND) 의 silent corruption 검출.

엔터프라이즈 storage / NVMe enterprise 에서 사용.

## 10. HBA (Host Bus Adapter)

### IT Mode (Initiator-Target)
- 컨트롤러가 디스크 raw 노출.
- ZFS / Ceph / 소프트웨어 RAID / Linux MD 와 짝.
- 권장: 데이터센터 / NAS / 자가운영.

### IR Mode (Internal RAID)
- 컨트롤러가 RAID 처리.
- HW RAID 5/6/10 자체 제공.
- BBU (Battery Backup Unit) 또는 FBWC (Flash-Backed Write Cache) 로 write cache 보호.

### Tri-mode HBA (Broadcom 9500 series)
- SAS / SATA / NVMe 동시 한 컨트롤러.
- 데이터센터의 mixed pool 에 유용.

## 11. 진단 / 측정 — 종합

```bash
# 디스크 목록
lsblk
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT,MODEL
sudo nvme list
sudo nvme list-subsys                    # NVMe-oF 포함

# NVMe 자세히
sudo nvme id-ctrl /dev/nvme0n1 | head -40
sudo nvme smart-log /dev/nvme0n1
sudo nvme list-ns /dev/nvme0
sudo nvme show-regs /dev/nvme0          # controller register

# SATA / SAS SMART
sudo smartctl -a /dev/sda

# 인터페이스 / PCIe link
sudo lspci -vvv -s $(lspci | grep -i nvme | head -1 | awk '{print $1}') | grep -E 'LnkCap|LnkSta'

# Topology
lsblk -t
sudo dmidecode -t 3                     # chassis

# Performance 벤치
sudo apt install fio
fio --rw=randread --bs=4k --iodepth=32 --numjobs=4 --runtime=30 \
    --filename=/dev/nvme0n1 --direct=1 --time_based --group_reporting

# Sequential
fio --rw=read --bs=1M --iodepth=8 --numjobs=1 --runtime=30 \
    --filename=/dev/nvme0n1 --direct=1 --time_based

# Mixed 70/30 random
fio --rw=randrw --rwmixread=70 --bs=4k --iodepth=32 --numjobs=4 \
    --runtime=30 --filename=/dev/nvme0n1 --direct=1 --time_based

# 큐 / 인터럽트 통계
cat /proc/interrupts | grep nvme
```

## 12. 함정

1. **M.2 키 미스매치** — B-only 슬롯 (SATA) 에 M-key (NVMe) 디스크 안 들어감. 보드 매뉴얼 필수.
2. **PCIe lane 충돌** — M.2 NVMe 가 chipset 의 SATA / 다른 PCIe 슬롯 lane 공유 → 그 SATA 포트 비활성.
3. **U.2 와 SATA 같은 SFF-8639 커넥터** — 케이블 / backplane 이 NVMe 지원 명시 확인.
4. **NVMe heatsink 없음** — Gen5 SSD 가 80 °C 넘으면 throttling. 발열 처리 필수.
5. **NVMe-oF over standard TCP** — RDMA 없이 latency 향상 한계. AI 학습은 RDMA 권장.
6. **HBA 와 RAID 카드 혼동** — IT mode 펌웨어 flash 안 하면 디스크가 ZFS 에 raw 로 안 보임.
7. **EDSFF 호환성** — E1.S 와 E3.S 는 connector 다름. chassis 매핑 확인.
8. **AHCI 모드 BIOS** — 옛 보드 BIOS 가 디폴트 IDE 모드 → SSD 성능 1/3.
9. **SATA SSD 의 random IOPS 한계** — 80K IOPS 가 천장. DB / VM 호스트는 NVMe.
10. **NVMe 큐 affinity 미설정** — irqbalance 가 균등하게 분산. NUMA pinning 으로 latency tail 줄임.
11. **SAS expander chain 폭주** — expander 1 개 뒤 SAS 디스크 24+ 면 BW 공유. enterprise 는 직결 권장.
12. **NVMe firmware bug** — 컨슈머 SSD (Samsung 990 Pro 등) 도 펌웨어 bug 가 흔함. 정기 업데이트.

## 13. 관련

- [[storage]]
- [[hdd]]
- [[ssd-and-ftl]]
- [[raid-levels]]
- [[../bus-io/pcie]] — NVMe 가 올라타는 layer
- [[../network-hardware/rdma-infiniband]] — NVMe-oF over RDMA
