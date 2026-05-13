---
title: "스토리지 인터페이스 (SATA / SAS / NVMe / EDSFF)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, storage, sata, sas, nvme, m.2, u.2, edsff]
---

# 스토리지 인터페이스

**[[storage|↑ 저장 장치]]**

## 1. 프로토콜 비교

| 프로토콜 | 위에 | 큐 | 속도 | 큐 깊이 | 듀얼 포트 |
| --- | --- | --- | --- | --- | --- |
| **SATA** | AHCI | 1 | 6 Gbps | 32 (NCQ) | × |
| **SAS** | SCSI | 1 | 12/24 Gbps | 256+ | ✓ |
| **NVMe** | PCIe x1/x2/x4/x8 | 64K × 64K | up to PCIe gen × lane | 64K | ✓ (U.2/U.3) |
| **NVMe-oF** | TCP / RDMA / FC | NVMe over fabric | 100 GbE+ | NVMe 동일 | ✓ |

## 2. SATA / AHCI

- 2003 도입. PATA (IDE) 의 직렬 후속.
- 한 디스크에 하나의 채널, NCQ 로 32 명령 큐.
- 데스크탑 / 가정용 SSD·HDD 의 표준.
- 6 Gbps 가 마지막 — 그 이상은 NVMe.

## 3. SAS / SCSI

- SCSI 명령셋 + Serial Attached 물리 계층.
- 12 Gbps (SAS-3) / 24 Gbps (SAS-4).
- **듀얼 포트** — 한 디스크에 2 개 컨트롤러 동시 연결 (페일오버).
- 256+ 깊은 큐. SCSI Reservation 으로 클러스터 공유 디스크.
- 엔터프라이즈 HDD / 일부 SSD.

## 4. NVMe

- 2011 사양 1.0. PCIe 위 비휘발성 메모리 전용 프로토콜.
- **AHCI 의 한계 (1 큐 / 32 깊이) 를 정면 돌파**: 64K 큐 × 64K 깊이.
- 결과: 멀티 코어 IO 가 락 없이 큐 분산. 100 만+ IOPS 가능.

### NVMe 명령 종류
- **Admin queue** — Identify, Create/Delete Queue, Get Log, Format.
- **I/O queue** — Read, Write, Flush, Compare, Write Zeroes, Dataset Management (TRIM 포함).

### NVMe 폼팩터 / 인터페이스

| 폼팩터 | 인터페이스 | 용도 |
| --- | --- | --- |
| M.2 | NVMe (PCIe x2/x4) 또는 SATA | 노트북 / 데스크탑. 작은 사이즈. 핀 키 (M/B/B+M) 로 PCIe / SATA 구분. |
| U.2 | NVMe PCIe x4 | 서버 2.5" hot-swap (SFF-8639). |
| U.3 | NVMe / SAS / SATA tri-mode | U.2 후속. 같은 슬롯이 3 프로토콜 모두. |
| AIC | NVMe PCIe x4/x8 | PCIe 카드. 워크스테이션 / 일부 서버. |
| **E1.S** | NVMe PCIe x4 | 데이터센터 thin ruler (5.9 mm 두께). 100+ drives/rack. |
| **E1.L** | NVMe PCIe x4 | 긴 ruler. 대용량. |
| **E3.S / E3.L** | NVMe / CXL | 새 EDSFF 표준. CXL 지원. |

### 속도

| PCIe gen | per-lane | x4 NVMe |
| --- | --- | --- |
| 3.0 | ~1 GB/s | 3.5 GB/s |
| 4.0 | ~2 GB/s | 7 GB/s |
| 5.0 | ~4 GB/s | 14 GB/s |
| 6.0 (예정) | ~8 GB/s | 28 GB/s |

## 5. NVMe-oF (Over Fabrics)

NVMe 명령을 네트워크 위로 전달.

| 전송 | 비고 |
| --- | --- |
| **NVMe/TCP** | 가장 보편. 100 GbE+ |
| **NVMe/RDMA (RoCE v2 / iWARP)** | 낮은 latency. RDMA NIC 필요. |
| **NVMe/FC** | Fibre Channel SAN. enterprise |

쓰임:
- 분리형 스토리지 (composable infrastructure).
- 데이터센터 SSD pool.
- AI 학습 datasets 의 공유 read.

## 6. 폼팩터별 PCIe lane / 전력

| 폼팩터 | lane | 일반 전력 |
| --- | --- | --- |
| M.2 2280 | x4 | 8.25 W max |
| U.2 | x4 | 25 W |
| U.3 | x4 | 25 W |
| E1.S 5.9 mm | x4 | 12 W |
| E1.S 9.5/15/25 mm | x4 | 16/25 W |
| E1.L | x4 | 25 W |
| E3.S | x4/x8 | 25/35 W |
| E3.L | x4/x8 | 40 W |

→ 더 빠른 PCIe Gen5 SSD 는 발열 ↑. 데스크탑 M.2 는 heatsink 필수 (없으면 thermal throttling).

## 7. HBA (Host Bus Adapter) / RAID 카드

- **HBA (IT mode)** — 디스크를 그냥 OS 에 노출. JBOD. ZFS / Ceph / 소프트웨어 RAID 와 짝.
- **HW RAID (IR mode)** — 컨트롤러가 RAID 처리. 별도 BBU / FBWC.
- 현대 데이터센터는 HBA + 소프트웨어 RAID 트렌드.

## 8. 함정

1. **M.2 키 미스매치** — B key (SATA only) 슬롯에 M key (NVMe) 디스크는 안 들어감. 보드 매뉴얼 확인.
2. **PCIe lane 충돌** — M.2 NVMe 가 chipset 의 SATA / 다른 PCIe 슬롯 lane 을 공유.
3. **U.2 와 SATA 가 같은 SFF-8639 커넥터** — 케이블 / backplane 이 NVMe 지원 명시 확인.
4. **NVMe heatsink 없음** — Gen5 SSD 가 80 °C 넘으면 throttling.
5. **NVMe-oF over standard TCP** — RDMA 없이 latency 향상 한계. ML 학습은 RDMA 권장.
6. **HBA 와 RAID 카드를 혼동** — IT mode 펌웨어 flash 하기 전엔 디스크가 ZFS 에 raw 로 안 보임.

## 9. 관련

- [[storage]]
- [[hdd]]
- [[ssd-and-ftl]]
- [[../bus-io/pcie]] — NVMe 가 올라타는 layer
