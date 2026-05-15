---
title: "실습 02 — Spring Boot Dockerfile"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T03:32:00+09:00
tags: [devops, docker, practice, spring-boot]
---

# 실습 02 — Spring Boot Dockerfile

**[[practice|↑ practice]]**

> 30분 — Spring Boot 앱을 multi-stage image 로 build.

---

## 1. 샘플 app

```bash
# https://start.spring.io 에서 dependencies: Web, Actuator
unzip demo.zip && cd demo
```

또는 기존 Spring Boot 프로젝트.

---

## 2. Dockerfile (multi-stage)

```dockerfile
# syntax=docker/dockerfile:1.7

# ===== Build stage =====
FROM eclipse-temurin:21-jdk-jammy AS build
WORKDIR /workspace

# 의존성 layer
COPY gradle gradle
COPY gradlew settings.gradle build.gradle ./
RUN chmod +x gradlew && ./gradlew dependencies --no-daemon || true

# 코드 + build
COPY src src
RUN ./gradlew bootJar --no-daemon

# ===== Runtime stage =====
FROM eclipse-temurin:21-jre-jammy
RUN groupadd -r app && useradd -r -g app app
WORKDIR /app
COPY --from=build /workspace/build/libs/*.jar app.jar
USER app
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s --start-period=30s \
    CMD wget -qO- http://localhost:8080/actuator/health || exit 1
ENTRYPOINT ["java", "-XX:+UseG1GC", "-XX:MaxRAMPercentage=75", "-jar", "app.jar"]
```

---

## 3. .dockerignore

```
.git
.gradle
build
target
*.iml
.idea
node_modules
```

---

## 4. Build

```bash
DOCKER_BUILDKIT=1 docker build -t demo:1.0 .
docker images demo
# REPOSITORY  TAG    SIZE
# demo        1.0    280MB  (multi-stage)
# without multi-stage: 1.5GB+
```

---

## 5. Run

```bash
docker run -d -p 8080:8080 --name demo demo:1.0
curl http://localhost:8080/actuator/health
# {"status":"UP"}
```

---

## 6. 로그 / 디버그

```bash
docker logs -f demo
docker exec -it demo bash       # USER app 이므로 root X
docker top demo                  # process
docker stats demo                # CPU / mem
```

---

## 7. Cache 확인

```bash
# 코드 변경 후 rebuild
touch src/main/java/.../Application.java
DOCKER_BUILDKIT=1 docker build -t demo:1.1 .
# 의존성 layer cached → 10초 안 끝남
```

---

## 8. Scan

```bash
docker scout cves demo:1.0     # Docker Scout
# 또는
trivy image demo:1.0
```

---

## 9. 다음

[[03-compose-full-stack]] — DB + Redis + nginx 통합.
