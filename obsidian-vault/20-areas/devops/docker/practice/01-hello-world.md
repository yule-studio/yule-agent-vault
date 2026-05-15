---
title: "실습 01 — Hello World"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T03:30:00+09:00
tags: [devops, docker, practice]
---

# 실습 01 — Hello World

**[[practice|↑ practice]]**

> 15분 — 첫 컨테이너 실행.

---

## 1. nginx 실행

```bash
docker run -d -p 8080:80 --name web nginx:1.27-alpine
```

브라우저 → `http://localhost:8080` → "Welcome to nginx!"

---

## 2. 안 들어가 보기

```bash
docker exec -it web sh
ls /usr/share/nginx/html
cat /etc/nginx/nginx.conf
exit
```

---

## 3. log

```bash
docker logs -f web
```

---

## 4. 정리

```bash
docker stop web
docker rm web
```

---

## 5. 직접 image build

```bash
mkdir hello-docker && cd hello-docker
cat > index.html <<EOF
<h1>Hello Docker!</h1>
EOF

cat > Dockerfile <<EOF
FROM nginx:1.27-alpine
COPY index.html /usr/share/nginx/html/
EOF

docker build -t hello:1.0 .
docker run -d -p 8080:80 hello:1.0
```

브라우저 → "Hello Docker!"

---

## 6. 확인

| 명령 | 결과 |
| --- | --- |
| `docker images` | hello:1.0 + nginx:1.27-alpine |
| `docker ps` | running 컨테이너 |
| `docker history hello:1.0` | layer 별 |

---

## 7. 다음

[[02-dockerfile-spring-boot]] — Spring Boot 앱 image build.
