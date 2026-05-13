---
title: "저장 장치 (Storage) — Hub"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, storage, hdd, ssd, nand, nvme, raid]
---

# 저장 장치 (Storage) — Hub

**[[../hardware|↑ hardware]]**

> 비휘발성 매체. 휘발성 메모리는 [[../memory/memory]]. 파일 시스템 / 볼륨 매니저는 OS 노트 ([[../../operating-system/operating-system]]) 에서.

## 0. 한 줄

**전원 끊겨도 데이터를 유지하는 매체 — HDD (자기) / SSD (NAND) / Tape / Optane / NVDIMM 등.**

## 1. HDD (Hard Disk Drive)
→ [[hdd|HDD]]

회전 디스크 + 헤드. 5400~15000 RPM. seek + rotational latency 합산 ms 단위. 큰 용량 / 싼 단가의 cold storage.

## 2. SSD + FTL
→ [[ssd-and-ftl|SSD 와 FTL]]

NAND Flash 기반. block 단위 erase 한계 때문에 FTL 이 logical→physical 매핑 + wear leveling + GC + TRIM. SATA 500 MB/s, NVMe Gen5 14 GB/s.

## 3. NAND 타입

| 종류 | bit/cell | P/E 수명 | 가격 |
| --- | --- | --- | --- |
| SLC | 1 | 100,000 | 비쌈 |
| MLC | 2 | 10,000 | 중 |
| TLC | 3 | 3,000 | 싸다 |
| QLC | 4 | 1,000 | 가장 싸 |
| PLC | 5 | ~150 | 시제품 |

3D NAND: 셀을 수직 적층 (232 layer+, 2025). 같은 면적에 용량 ↑.

## 4. RAID
→ [[raid-levels|RAID 레벨]]

RAID 0/1/5/6/10 의 trade-off. 큰 HDD 에서는 RAID 5 비추 (rebuild URE 위험), RAID 6 / 10 / Z2 권장.

## 5. 인터페이스
→ [[storage-interfaces|스토리지 인터페이스]]

- **SATA** 6 Gbps — 데스크탑.
- **SAS** 12/24 Gbps — 서버, 듀얼 포트.
- **NVMe** PCIe x4 — Gen3 3.5 GB/s, Gen4 7 GB/s, Gen5 14 GB/s.
- **M.2 / U.2 / U.3 / E1.S / E1.L / E3.S / EDSFF** — 폼팩터.

## 6. Optane / 3D XPoint / Persistent Memory

- Intel + Micron 3D XPoint, 2015 발표. DRAM-SSD 사이.
- 2022 Optane 단종.
- 대체는 CXL.mem (DRAM 또는 비휘발 매체).

## 7. 함정

1. **무한 쓰기 가정** — TLC 3000 / QLC 1000 P/E. TBW 라벨 확인.
2. **PCIe lane 충돌** — M.2 NVMe 가 SATA 포트 비활성시키는 보드.
3. **RAID 5 + 큰 HDD** — rebuild URE 위험.
4. **TRIM 미사용** — SSD 가 deleted LBA 를 모르면 GC 가 효율 ↓.
5. **fsync 누락** — power loss 시 데이터 손실. enterprise SSD 는 PLP (Power Loss Protection) 캐패시터 보유.

## 8. 관련

- [[../memory/memory]]
- [[../bus-io/pcie]] — NVMe 가 올라타는 layer
- [[../../operating-system/operating-system]] — 파일 시스템, IO 스케줄러
