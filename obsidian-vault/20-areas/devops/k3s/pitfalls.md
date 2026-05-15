---
title: "k3s — 함정 모음"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T09:25:00+09:00
tags: [devops, k3s, pitfalls]
---

# k3s — 함정 모음

**[[k3s|↑ k3s]]**

---

## 1. installation

1. **`latest` channel** — 의도치 않은 minor jump.
2. **token leak** — `/var/lib/rancher/k3s/server/token` 권한 600.
3. **NTP 안 함** — TLS / etcd 영향.
4. **kubeconfig 권한 644 일반 환경** — 일반 user 가 cluster 접근.
5. **install script 의 `curl|sh`** — script 검증 후 사용.
6. **firewall port 누락** — 6443/10250/8472 안 열림.
7. **swap on** — k8s 거부.

---

## 2. HA

1. **2 server (짝수)** — quorum 위협.
2. **NoExecute taint 없음** — server 에 일반 pod → control plane 영향.
3. **LB 없이 첫 server IP** — 그 server down 시 join 실패.
4. **cluster-init 두 번** — 두 다른 cluster.
5. **token 다름** — join 실패.
6. **external DB single instance** — DB down = cluster down.
7. **etcd disk = HDD** — fsync slow → cluster 느림.

---

## 3. networking

1. **firewall 6443/8472 막힘** — join / pod 통신 X.
2. **flannel + Calico 동시** — 충돌.
3. **Traefik + nginx-ingress** — port 충돌.
4. **klipper-lb hostPort 충돌** — 한 node 의 :80.
5. **cluster-cidr 회사 LAN 중복** — 10.42.x.x 충돌.
6. **CoreDNS upstream 느림** — resolv.conf.
7. **dual-stack 미설정** — IPv6 안 됨.

---

## 4. storage

1. **local-path + 여러 replica** — pod 다른 node 가면 데이터 X.
2. **Longhorn 1 node** — replica 1 == HA 없음.
3. **open-iscsi 없음** — Longhorn 동작 안 함.
4. **NFS SPOF** — 전체 service 영향.
5. **backup 검증 안 함** — 복구 깨짐.
6. **PVC 삭제 = data 삭제** — Retain 고려.
7. **storage class default 변경** — 기존 manifest break.

---

## 5. upgrade

1. **`stable` 자동 upgrade** — 의도치 않은 minor jump.
2. **agent 가 먼저** — control plane 보다 신 version → error.
3. **etcd snapshot 안 함** — rollback 불가.
4. **minor 2개 jump** — k8s 정책 위반.
5. **CRD migration 안 함** — apply 실패.
6. **off-peak 안 함** — 사용자 영향.
7. **controller plan 두 번** — 동시 drain → down.

---

## 6. backup / DR

1. **snapshot 만, S3 미upload** — disk 손실 시 잃음.
2. **manifest backup 안 함** — 새 cluster 띄워도 service X.
3. **secrets 분리 X** — backup 노출.
4. **retention 없음** — disk full.
5. **restore drill 안 함** — disaster 시 fail.
6. **PV backup 빠짐** — etcd 만.
7. **encryption 없음** — backup 도난.

---

## 7. GitOps

1. **manifests/ + GitOps 동시** — drift / 충돌.
2. **prune: true 가 git delete = cluster 즉시 삭제** — 위험.
3. **Application name 충돌** — namespace 분리.
4. **webhook 없음** — 3분 polling.
5. **secret git 평문** — SealedSecret / SOPS.
6. **rollback git revert** — image tag 옛 것 필요.
7. **ApplicationSet generator 잘못** — 모든 cluster 에 잘못 manifest.

---

## 8. edge

1. **RPi SD card 수명** — SSD 권장.
2. **swap on** — k8s 거부.
3. **time drift** — chrony 필수.
4. **WAN 끊김 시 cluster X** — local-first 디자인.
5. **single-arch image** — RPi 가 amd64 못 실행.
6. **cert 만료** — 1년. 자동 갱신.
7. **monitoring 전부 stream** — 본사 비용 폭주.

---

## 9. k3d

1. **Docker memory limit** — Mac Docker Desktop 부족.
2. **registry-create 후 별도 사용** — `registry-use` 필요.
3. **WSL2 DNS 충돌**.
4. **너무 많은 cluster** — laptop 죽음.
5. **LB port 충돌** — cluster 마다 다른 port.
6. **kubeconfig context 혼동** — kubectx 활용.

---

## 10. migration

1. **default storageClass 다름** — manifest 검토.
2. **klipper-lb vs ALB** — Service type 다름.
3. **Traefik vs nginx annotation** — 호환 X.
4. **secret 평문 git** — 마이그 중 누출.
5. **DR drill 안 함**.
6. **DNS TTL 길게** — phase 전환 늦음.
7. **양쪽 cluster DB write 경합**.

---

## 11. production 일반

1. **single server production** — SPOF.
2. **resource limit 없음** — 한 pod 가 node 전부.
3. **readiness / liveness probe 없음** — traffic 너무 일찍.
4. **PDB 없음** — drain 끊김.
5. **Pod Security Admission 없음** — privileged pod 가능.
6. **NetworkPolicy 없음** — flat network.
7. **k3s version `latest`** — breaking.
8. **monitoring 없음** — incident 인지 X.

---

## 12. 관련

- [[k3s|↑ k3s]]
