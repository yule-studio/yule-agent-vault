---
title: "실습 03 — Compose full stack (Spring + PG + Redis + nginx)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T03:34:00+09:00
tags: [devops, docker, practice, compose]
---

# 실습 03 — Compose full stack (Spring + PG + Redis + nginx)

**[[practice|↑ practice]]**

> 1시간 — 실 production 비슷한 stack.

---

## 1. 디렉토리 구조

```
my-stack/
├── docker-compose.yml
├── Dockerfile
├── .dockerignore
├── .env
├── nginx/
│   └── default.conf
└── src/...                  # Spring Boot
```

---

## 2. docker-compose.yml

```yaml
services:
  app:
    build: .
    environment:
      SPRING_PROFILES_ACTIVE: prod
      DB_URL: jdbc:postgresql://postgres:5432/demo
      DB_USER: ${DB_USER}
      DB_PASSWORD: ${DB_PASSWORD}
      REDIS_HOST: redis
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 3s
      retries: 5
      start_period: 60s
    networks: [app-net]

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: demo
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - pg-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $$POSTGRES_USER"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks: [app-net]

  redis:
    image: redis:7-alpine
    command: ["redis-server", "--appendonly", "yes"]
    volumes:
      - redis-data:/data
    networks: [app-net]

  nginx:
    image: nginx:1.27-alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - app
    networks: [app-net]

volumes:
  pg-data:
  redis-data:

networks:
  app-net:
```

---

## 3. .env

```
DB_USER=demo
DB_PASSWORD=demopw
```

→ git ignore (절대 commit X).

---

## 4. nginx/default.conf

```nginx
upstream app {
    server app:8080;
}

server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://app;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /actuator/ {
        return 403;  # health endpoint 외부 노출 X
    }
}
```

---

## 5. 실행

```bash
docker compose up -d
docker compose ps
# NAME            STATUS
# my-stack-app    Up (healthy)
# my-stack-pg     Up (healthy)
# my-stack-redis  Up
# my-stack-nginx  Up

curl http://localhost/
```

---

## 6. 디버그

```bash
docker compose logs -f app
docker compose exec app bash
docker compose exec postgres psql -U demo -d demo
docker compose exec redis redis-cli
```

---

## 7. 정리

```bash
docker compose down          # stop + container 제거
docker compose down -v       # + volume (DB 데이터 삭제)
```

---

## 8. 다음 단계

- Kubernetes 로 이전 → [[../../kubernetes/practice/practice|↗ k8s 실습]]
- CI/CD 자동화 → [[../../cicd/cicd|↗ cicd]]
