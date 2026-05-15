---
title: "Docker 실습 hub"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T03:28:00+09:00
tags: [devops, docker, practice]
---

# Docker 실습 hub

**[[../docker|↑ docker]]**

> 단계별 실습 — 본인 PC 에서 직접 실행.

---

| # | 실습 | 시간 |
| --- | --- | --- |
| 1 | [[01-hello-world]] — 첫 컨테이너 | 15분 |
| 2 | [[02-dockerfile-spring-boot]] — 본인 앱 image build | 30분 |
| 3 | [[03-compose-full-stack]] — Spring + PostgreSQL + Redis + nginx | 1시간 |

---

## 준비

```bash
docker --version           # 24+
docker compose version     # v2.20+
```

설치: [https://docs.docker.com/get-docker/](https://docs.docker.com/get-docker/)

---

## 관련

- [[../docker|↑ docker]]
- [[../concepts]]
- [[../dockerfile-best-practices]]
- [[../compose]]
