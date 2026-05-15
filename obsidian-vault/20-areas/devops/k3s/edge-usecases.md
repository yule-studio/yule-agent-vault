---
title: "k3s edge use cases — RPi / K3OS / 매장 / vehicle"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T09:19:00+09:00
tags: [devops, k3s, edge, iot]
---

# k3s edge use cases — RPi / K3OS / 매장 / vehicle

**[[k3s|↑ k3s]]**

---

## 1. edge 특징

```
- network 불안정 (오프라인 일시)
- resource 제약 (CPU/RAM/disk)
- 수십~수만 사이트
- 물리 접근 어려움 (자동 복구 필요)
- 보안 (도난 / 침해)
- 다양한 hardware (RPi / Intel NUC / 산업 PC)
- ARM / x86 mix
```

→ 일반 k8s = 부적합. k3s + GitOps 가 답.

---

## 2. 흔한 사용 사례

| | 무엇 |
| --- | --- |
| **homelab** | RPi 4-5 클러스터, 개인 service |
| **편의점 / 매장** | POS / kiosk / 디지털 사이니지 |
| **공장 / 제조** | OT (Operational Technology) data 수집 + analytics |
| **자동차** | 차량 내 ECU + cloud 통신 |
| **선박 / 항공** | 오프라인 service + 정기 sync |
| **agriculture** | sensor / drone aggregation |
| **healthcare** | ICU / 영상 처리 |
| **retail** | inventory / 결제 / 디지털 광고 |

---

## 3. RPi cluster (homelab) 패턴

```
RPi 4 (8GB) × 3 + PoE switch + NAS:
  RPi 1 — k3s server (control plane)
  RPi 2,3 — k3s agent (workload)
  NAS — shared storage (NFS / iSCSI)

OS: Raspberry Pi OS / Ubuntu 22.04 LTS for ARM
swap off, cgroup 활성
```

```bash
# cgroup (RPi 의 boot/firmware/cmdline.txt 끝에 추가)
cgroup_memory=1 cgroup_enable=memory

# install
curl -sfL https://get.k3s.io | sh -
```

→ 메모리 1.5GB / cluster 운영 가능.

---

## 4. K3OS — k8s 전용 immutable OS

```
- Linux distro (Alpine 기반).
- k3s 가 first-class.
- 설정이 yaml (cloud-init 스타일).
- A/B 파티션 (atomic upgrade + rollback).
- 작음 (256MB).

✅ edge / IoT 최적.
⚠ 2022 부터 archive 상태 — 대안 Talos / Flatcar.
```

→ 새 프로젝트는 **Talos Linux** 권장.

---

## 5. Talos Linux (★ 권장)

```
- API-driven (SSH 없음).
- immutable + minimal.
- single-binary k8s.
- 보안 강 (read-only root).

설치:
  PXE / USB → talosctl config
  talosctl gen config my-cluster https://10.0.0.1:6443
  talosctl apply-config --insecure --nodes 10.0.0.1 --file controlplane.yaml
  talosctl bootstrap --nodes 10.0.0.1

→ k3s 의 production edge 대안.
```

---

## 6. 매장 (single-node) 패턴

```
각 매장:
  - 1 미니 PC (Intel NUC / mini-server)
  - k3s single-node
  - GitOps (Fleet) — 자동 update
  - local-path storage
  - VPN / mTLS 으로 본사 연결
  - WAN 끊겨도 local service 동작
  - 정기 metric / log → 본사

본사:
  - Rancher / Fleet 관리
  - 한 click 으로 1000 매장 update
  - per-cluster label / target
```

---

## 7. WAN 끊김 대응

```
1. local DNS cache (NodeLocalDNS)
2. private registry mirror (정기 sync)
3. queue / outbox (Kafka / NATS) — 본사 끊겨도 local 큐
4. local DB (Postgres replica + local writes)
5. event-driven sync (eventual consistency)
6. local secret 캐시 (Vault agent / file)
```

---

## 8. 자동 복구 (★ unattended)

```
1. systemd-resolved + chrony 자동
2. k3s auto-restart (systemd Restart=always)
3. monitoring agent → local action (k8s liveness)
4. cluster auto-upgrade (system-upgrade-controller)
5. fallback boot (Talos A/B / read-only)
6. 정기 health check → 본사 보고
7. heartbeat lost → 자동 ticket 생성
```

→ "사람이 매장에 갈 일이 거의 없어야".

---

## 9. 보안 (★ edge 특수)

```
물리 도난:
  ☐ disk encryption (LUKS)
  ☐ TPM / secure boot
  ☐ remote wipe 기능

침해:
  ☐ network 격리 (VPN only)
  ☐ admission control (signed image 만)
  ☐ runtime (Falco) 침해 탐지
  ☐ 정기 patch (system-upgrade-controller)

credential:
  ☐ short-lived token (Vault dynamic)
  ☐ secrets 메모리만 (file X)
  ☐ mTLS for everything
```

---

## 10. monitoring (★ 작은 footprint)

```
edge 의 monitoring:
  - 본사로 모두 stream = bandwidth 비쌈
  - local 에서 aggregate / sample → 본사로 요약

도구:
  - Prometheus agent (push)
  - VictoriaMetrics agent
  - Telegraf
  - OpenTelemetry Collector (sampling / batch)

본사:
  - Mimir / Thanos / Datadog Cloud
```

---

## 11. multi-arch image (★)

```
RPi (ARM64) + Intel NUC (AMD64) 같은 cluster:

# build (docker buildx)
docker buildx create --use --name multi
docker buildx build \
    --platform linux/amd64,linux/arm64 \
    --push \
    -t registry/my-app:1.0 .

# k8s 의 Pod 가 자동 architecture 별 선택
```

---

## 12. resource 예 (예산)

| Workload | RPi 4 4GB | RPi 4 8GB | Intel NUC i5 32GB |
| --- | --- | --- | --- |
| k3s server alone | 500MB | 700MB | 1GB |
| + 10 pod (light) | 1.5GB | 2GB | 4GB |
| + DB + cache | 2.5GB | 4GB | 8GB |
| + ML inference | 부족 | 6GB | 12GB |

→ 4GB RPi = light workload only.

---

## 13. fleet management 예 (Rancher / Fleet)

```
1. Rancher 1대 (cloud / on-prem)
2. 각 매장 → Rancher agent install
3. cluster 자동 등록
4. label (env=prod, region=east, store=001)
5. Fleet GitRepo + clusterSelector → 자동 deploy
6. cluster 별 dashboard
7. failed cluster 자동 ticket
```

---

## 14. 함정

1. **RPi SD card** — 수명 짧음. SSD 권장.
2. **swap on** — k8s 가 거부. swap off.
3. **time drift** — chrony 필수.
4. **WAN 끊김 시 cluster 안 동작** — local-first 디자인.
5. **multi-arch image 안 만듦** — RPi 가 amd64 image 못 실행.
6. **cluster certificate 만료** — 1년. 자동 갱신 확인.
7. **monitoring 전부 stream** — 본사 비용 폭주.

---

## 15. 관련

- [[k3s|↑ k3s]]
- [[ha-mode]]
- [[gitops]]
- [[../sre/disaster-recovery|↗ DR]]
