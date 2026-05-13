---
title: "HDD (Hard Disk Drive)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:30:00+09:00
tags: [hardware, storage, hdd, platter, seek, rpm]
---

# HDD (Hard Disk Drive)

**[[storage|↑ 저장 장치]]**

## 1. 구조

```
  ┌──────────────────────────┐
  │   ┌──────────┐           │
  │   │ platter  │ ◄── 회전 (RPM)
  │   │ ───────  │           │
  │   │ ───────  │           │
  │   └──────────┘           │
  │       ▲                  │
  │       │ head             │
  │  ─────┴──────  ◄── 액추에이터 암 (seek)
  └──────────────────────────┘
```

- **Platter**: 자성 코팅된 알루미늄/유리 원판. 1~9 장.
- **Head**: 양 면에 1 개씩. 비행 높이 수 nm.
- **Actuator arm**: 헤드 위치 이동.
- **Spindle motor**: 5400 / 7200 / 10000 / 15000 RPM.

## 2. 핵심 지표

| 지표 | 의미 | 예 |
| --- | --- | --- |
| RPM | 회전 속도 | 5400 (저전력), 7200 (일반), 10K (엔터프라이즈), 15K (Tier 0) |
| Seek time | 헤드 이동 평균 | 5–15 ms |
| Rotational latency | 평균 (0.5 회전) | 7200 → 4.17 ms |
| Sustained throughput | 연속 read/write | 100–250 MB/s |
| Random IOPS | 4K random | 100–200 |
| MTBF | 평균 무고장 시간 | 1.0–2.5 M hours |

### 식

- `random access time ≈ seek + rotational latency`
- 7200 RPM, 9 ms seek → ~13 ms 평균.

## 3. 인터페이스

- **SATA 6 Gbps** — 데스크탑 / 가정용.
- **SAS 12/24 Gbps** — 서버. 듀얼 포트 (페일오버), 더 깊은 큐, full-duplex.

## 4. 폼팩터

| 폼팩터 | 크기 | 용량 |
| --- | --- | --- |
| 3.5" | 데스크탑 / 서버 | 1–24 TB (CMR), 28 TB (SMR/HAMR) |
| 2.5" | 노트북 / 서버 | 0.5–5 TB |

## 5. CMR / SMR / HAMR / MAMR

- **CMR (Conventional Magnetic Recording)** — 일반.
- **SMR (Shingled Magnetic Recording)** — 트랙을 기와처럼 겹쳐 밀도 ↑. 단점: random write 가 매우 느림 (re-write zone 필요). 가정용 NAS 에 일부 채택, 사용자가 모르고 사면 fail.
- **HAMR (Heat-Assisted Magnetic Recording)** — 작은 레이저로 매체 가열 후 기록. Seagate 28 TB+ (2024).
- **MAMR (Microwave-Assisted)** — Western Digital 의 대안.

## 6. 큐 깊이 / NCQ / TLER

- **NCQ (Native Command Queuing)** — SATA 가 디스크 안 명령 큐 (최대 32) 를 재정렬.
- **TCQ** — SAS 의 더 깊은 큐.
- **TLER (Time-Limited Error Recovery)** — Western Digital 의 RAID 친화 펌웨어. 일시적 read 지연이 RAID 컨트롤러에 fail 로 오해되지 않게 7 초 안에 포기.

## 7. 데이터센터 활용

- **Cold storage / 백업** — TB 당 가격이 SSD 의 1/5~1/10. Glacier, BackBlaze B2.
- **HDFS / Ceph** — replication 으로 가용성, HDD 가 backing store.
- **CCTV 저장** — 연속 write 최적.
- **24/7 가동 enterprise HDD** — Seagate Exos, WD Ultrastar.

## 8. 함정

1. **SMR 디스크를 RAID 에** — random write 폭주 시 동결.
2. **데스크탑 HDD 를 24/7 NAS 에** — 일반 디스크는 8/5 가동 가정 / vibration 약함. enterprise grade 사용.
3. **TLER 없는 디스크를 RAID 에** — recovery 길게 잡으면 RAID 가 fail 로 오해.
4. **물리 충격** — 가동 중 충격은 헤드 crash 위험. 노트북 / 외장 운반 주의.

## 9. 관련

- [[storage]]
- [[storage-interfaces]]
- [[raid-levels]]
- [[ssd-and-ftl]]
