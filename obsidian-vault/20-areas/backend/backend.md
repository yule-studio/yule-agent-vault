---
title: "backend — 백엔드 영역 Hub"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-19T00:30:00+09:00
tags: [area, backend, hub]
home_hub: areas
related:
  - "[[../areas]]"
  - "[[architecture/architecture]]"
  - "[[spring/spring]]"
  - "[[../computer-science/object-oriented-programming/object-oriented-programming]]"
  - "[[../computer-science/design-patterns/design-patterns]]"
  - "[[../devops/devops]]"
  - "[[../database/database]]"
---

# backend — 백엔드 영역 Hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-19 | engineering-agent/tech-lead | 최초 — 13 frameworks + architecture / _common 통합 hub |

**[[../areas|↑ 20-areas]]**

---

## 1. 영역 정의

본 영역은 서버 측 응용을 만드는 데 사용되는 **언어 / 프레임워크 / 아키텍처 / 공통 패턴** 의 인덱스다.

본 영역이 정의하는 것:
- 13 프레임워크 hub 의 진입점
- application 내부 아키텍처 (Layered / Hexagonal / Clean) 와 시스템 수준 아키텍처 (Monolith / MSA / Monorepo) 의 위치
- 공통 패턴 (`_common`) 의 위치
- 인접 영역 (database / devops / OOP / design-patterns) 과의 cross-link

본 영역이 정의하지 않는 것:
- 특정 프레임워크의 syntax / API — 각 프레임워크 hub
- DB 모델링 / 인덱스 / 쿼리 — [[../database/database]]
- 컨테이너 / 배포 / 운영 — [[../devops/devops]]
- OOP 원칙 / SOLID — [[../computer-science/object-oriented-programming/object-oriented-programming]]

---

## 2. 영역 분류

### 2.1 아키텍처 / 공통

| 디렉토리 | 진입점 | 다루는 주제 |
| --- | --- | --- |
| [[_common/|_common]] | (디렉토리 직접 참조) | 모든 프레임워크에 공통되는 패턴 / 유틸 |
| [[architecture/architecture|architecture]] | architecture.md | 시스템 수준 분리 (Monolith / MSA / Monorepo / 통신 / 데이터 일관성) |

### 2.2 JVM

| 프레임워크 | 진입점 | 비고 |
| --- | --- | --- |
| [[spring/spring|spring]] | spring.md | Java / Kotlin Spring — 가장 비대한 ecosystem |

### 2.3 Python

| 프레임워크 | 진입점 |
| --- | --- |
| [[python-django/|python-django]] | (django hub) |
| [[python-fastapi/|python-fastapi]] | (fastapi hub) |
| [[python-flask/|python-flask]] | (flask hub) |

### 2.4 Node.js

| 프레임워크 | 진입점 |
| --- | --- |
| [[node-express/|node-express]] | (express hub) |
| [[node-nestjs/|node-nestjs]] | (nestjs hub) |

### 2.5 Go / Rust / Ruby / PHP / .NET

| 프레임워크 | 진입점 |
| --- | --- |
| [[go-gin/|go-gin]] | (gin hub) |
| [[rust-axum/|rust-axum]] | (axum hub) |
| [[ruby-rails/|ruby-rails]] | (rails hub) |
| [[php-laravel/|php-laravel]] | (laravel hub) |
| [[dotnet-aspnet/|dotnet-aspnet]] | (asp.net hub) |

---

## 3. 학습 진입점 권장

| 목적 | 진입점 |
| --- | --- |
| 백엔드 입문 | [[../computer-science/object-oriented-programming/object-oriented-programming]] → [[architecture/architecture]] → 1 프레임워크 선택 |
| 시스템 설계 | [[architecture/architecture]] → [[../../40-patterns/architecture-patterns/architecture-patterns]] |
| 도메인 모델링 | [[../computer-science/object-oriented-programming/ddd-tactical-patterns]] → [[../computer-science/object-oriented-programming/value-objects]] |
| 프레임워크 적용 | [[../computer-science/object-oriented-programming/oop-for-backend]] → 해당 프레임워크 hub |
| API 설계 | [[../../40-patterns/api-patterns/]] |
| 인프라 / 배포 | [[../devops/devops]] |

---

## 4. 인접 영역과의 책임 분리

| 본 영역 | 인접 영역 | 분리 기준 |
| --- | --- | --- |
| 시스템 수준 분리 (Monolith / MSA / Monorepo) | application 내부 구조 (Layered / Hexagonal / Clean) → [[../../40-patterns/architecture-patterns/architecture-patterns]] | 같은 service 내부 vs 여러 service 간 |
| 프레임워크 사용법 | OOP 원칙 / SOLID → [[../computer-science/object-oriented-programming/object-oriented-programming]] | 도구 vs 원리 |
| 프레임워크 사용법 | 디자인 패턴 → [[../computer-science/design-patterns/design-patterns]] | 도구 vs 패턴 |
| 프레임워크 사용법 | DB 쿼리 / 인덱스 → [[../database/database]] | 응용 vs 저장 |
| 프레임워크 사용법 | 컨테이너 / 배포 → [[../devops/devops]] | 응용 vs 인프라 |
| 응용 계층 보안 | 암호 / 인증 이론 → [[../security/security]] | 적용 vs 원리 |
| HTTP API | TCP / HTTP / TLS → [[../computer-science/network/network]] | 적용 vs 프로토콜 |

---

## 5. 공통 결정 — 새 백엔드 프로젝트 시작

| 결정 | 옵션 |
| --- | --- |
| 언어 / 프레임워크 | 팀 skill / 생태계 / 성능 요구 / 채용 시장 |
| application 내부 구조 | [[../../40-patterns/architecture-patterns/architecture-patterns]] §3 매트릭스 |
| 배포 단위 | [[architecture/architecture]] §0 — Monolith → MSA |
| ORM 패턴 | [[../computer-science/object-oriented-programming/oop-for-backend]] §9 — Active Record vs Data Mapper |
| 도메인 모델링 | [[../computer-science/object-oriented-programming/ddd-tactical-patterns]] — Aggregate / VO / Repository |
| API 통신 | REST / GraphQL / gRPC — [[../../40-patterns/api-patterns/]] |
| DB | RDBMS / NoSQL / cache — [[../database/database]] |
| 인증 / 인가 | OAuth / JWT / mTLS — [[../security/security]] |

---

## 6. 본 영역의 유지보수

| 트리거 | 갱신 항목 |
| --- | --- |
| 새 프레임워크 디렉토리 추가 | §2 분류 + frontmatter related |
| 인접 영역과의 책임 변경 | §4 |
| 새 공통 결정 사항 | §5 |

---

## 7. 관련

- [[../areas|↑ 20-areas]]
- [[architecture/architecture]]
- [[spring/spring]]
- [[../computer-science/object-oriented-programming/object-oriented-programming]]
- [[../computer-science/design-patterns/design-patterns]]
- [[../computer-science/network/network]]
- [[../database/database]]
- [[../devops/devops]]
- [[../security/security]]
- [[../../40-patterns/architecture-patterns/architecture-patterns]]
- [[../../40-patterns/refactoring-patterns/refactoring-patterns]]
