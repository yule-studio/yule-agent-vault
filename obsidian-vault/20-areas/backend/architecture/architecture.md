---
title: "Backend Architecture — Hub"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T11:00:00+09:00
tags:
  - backend
  - architecture
  - hub
---

# Backend Architecture — Hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | hub — 배포 단위 / 코드 단위 / 데이터 패턴 / repo strategy |
| v.1.1.0 | 2026-05-19 | engineering-agent/tech-lead | [[../../../40-patterns/architecture-patterns/architecture-patterns|↗ 40-patterns/architecture-patterns]] (Layered/Hexagonal/Clean) link |

**[[../backend|↑ backend]]**

> 이 폴더는 **framework / 언어 무관 아키텍처 결정 문서**.
> "Spring 어떻게 쓰지" 는 [[../spring/spring|↗ spring hub]], "어떤 DB 쓰지" 는 [[../../database/database|↗ database hub]].
> 여기는 **시스템 전체를 어떻게 자르나** — 모놀리식 / MSA / 모노레포 / 통신 / 데이터 일관성.

> 한 application 내부 구조 (Layered / Hexagonal / Clean) 는 [[../../../40-patterns/architecture-patterns/architecture-patterns|↗ 40-patterns/architecture-patterns]] 영역. 본 hub 는 시스템 수준 분리만 다룬다.

---

## 0. 4 개의 독립적 결정 축

아키텍처 논의에서 자주 섞이는 4 개를 분리해 생각:

| 축 | 질문 | 선택지 |
| --- | --- | --- |
| **배포 단위** | 무엇을 하나의 프로세스 / 서비스로 배포하나 | Monolith / Modular Monolith / **MSA** / Serverless |
| **코드 저장소** | 코드를 어떻게 나눠 저장하나 | **Monorepo** / Multi-repo / Hybrid |
| **데이터 저장소** | 데이터를 어떻게 나누나 | 단일 DB / DB-per-service / shared schema / event-sourced |
| **통신** | 서비스 간 어떻게 대화하나 | REST / gRPC / Event (Kafka) / GraphQL Federation |

> **함정**: "MSA 가자" = 4 축 모두 바꾸는 것 아님. **배포 단위만 쪼개도 MSA**. repo / DB 는 한참 뒤에 바꿔도 됨.

---

## 1. 배포 단위 — Monolith / Modular Monolith / MSA / Serverless

### 1.1 비교 표

| | Monolith | Modular Monolith | MSA | Serverless (FaaS) |
| --- | --- | --- | --- | --- |
| 배포 단위 | 1 | 1 | N | 함수 단위 |
| 코드 응집도 | 한 덩어리 | 모듈 분리 | 서비스 분리 | 함수 분리 |
| 운영 복잡도 | 낮음 | 낮음 | **높음** | 중간 |
| 팀 독립성 | 낮음 | 중간 | 높음 | 높음 |
| 트랜잭션 | DB 트랜잭션 1개 | DB 트랜잭션 1개 | Saga / 2PC 필요 | 함수 간 어려움 |
| 디버깅 | 쉬움 | 쉬움 | 분산 트레이싱 필수 | 분산 트레이싱 필수 |
| 비용 효율 (초기) | ✅ 가장 좋음 | ✅ | ⚠️ 인프라 ↑ | ✅ 트래픽 가변 시 |
| 비용 효율 (성장 후) | ⚠️ 팀 충돌 | ✅ | ✅ | ⚠️ 콜드 스타트 |
| **권장 시점** | 0~Series A | A~B (10~30명) | B+ (30+명 또는 분리 강제) | 이벤트 처리 / 주변부 |

### 1.2 가는 길

```
Monolith ──→ Modular Monolith ──→ MSA (점진적 추출)
                              ╲
                               ╲──→ Modular Monolith 영구 (대부분 정답)
```

> "MSA 가 정답" 이 아님. **Modular Monolith 가 충분한 회사가 80%**. MSA 는 팀이 30+ 이거나, 서비스마다 다른 SLA / 보안 등급 / 스케일링 패턴이 강제될 때.

### 1.3 깊이 노트

- [[monorepo-vs-multirepo|↗ monorepo vs multirepo]] — repo 결정
- [[msa-overview|↗ MSA 개요와 분리 방법]] — 언제 / 어떻게
- [[modular-monolith|↗ Modular Monolith]] — MSA 의 90% 효과를 10% 비용으로
- saga-pattern (다음) — 분산 트랜잭션
- event-driven-architecture (다음)
- api-gateway (다음)
- service-mesh (다음)

---

## 2. 코드 저장소 — Monorepo / Multi-repo

### 2.1 한 줄

| 전략 | 한 줄 |
| --- | --- |
| **Monorepo** | 모든 서비스/패키지를 하나의 저장소. atomic commit, 공유 코드, 일관 도구. |
| **Multi-repo** | 서비스마다 별도 저장소. 독립 배포, 권한 분리, 빌드 격리. |
| **Hybrid** | core 는 monorepo, 일부 외부 노출 SDK 는 별 repo. |

상세: [[monorepo-vs-multirepo]]

### 2.2 배포 단위와 무관함

| 배포 \ Repo | Monorepo | Multi-repo |
| --- | --- | --- |
| Monolith | 자연스러움 | 가능 (드물게) |
| Modular Monolith | **흔함** | 가능 |
| MSA | **Google / Meta 식** | **Netflix / Amazon 식** |

→ MSA 가도 monorepo 유지 가능 (Google 이 그 증거). 결정은 **분리**.

---

## 3. 데이터 저장소 — DB 분리 전략

| 패턴 | 설명 | 적합 |
| --- | --- | --- |
| **Shared DB** | 모든 서비스가 같은 DB | 모놀리식 / 초기 MSA |
| **DB-per-service** | 서비스마다 자체 DB. cross-service JOIN 금지 | 성숙 MSA |
| **Read-replica per service** | 쓰기 master 공유, 읽기 replica 분리 | 단계적 분리 |
| **Event Sourcing + CQRS** | 쓰기는 event stream, 읽기는 projection | 도메인이 시간순/감사 중요 |

> **MSA 의 큰 함정**: "DB-per-service 가 진짜 MSA" 라며 처음부터 분리 → cross-service JOIN 불가 + 데이터 일관성 직접 구현. **초기엔 shared DB 도 OK**, 도메인 경계 명확해진 뒤 분리.

상세 (작성 예정):
- saga-pattern — 분산 트랜잭션
- transactional-outbox — 데이터 + 이벤트 원자성
- cqrs-event-sourcing

---

## 4. 통신 — 동기 vs 비동기

| 방식 | 사용처 | 장 | 단 |
| --- | --- | --- | --- |
| **REST / HTTP+JSON** | 외부 API / 단순 내부 | 디버깅 쉬움 / 도구 많음 | 스키마 강제 약함 |
| **gRPC** | 내부 서비스 간 | 빠름 / 강타입 (proto) | 브라우저 직접 X (grpc-web 필요) |
| **GraphQL (Federation)** | client-aggregation | 클라이언트 효율 | 캐싱·복잡도 ↑ |
| **Event (Kafka / SQS)** | 비동기 / pub-sub | 결합도 ↓ / replay / scale | 일관성 추론 어려움 |
| **WebSocket / SSE** | 실시간 push | 양방향 | 상태 stateful |

기본 권장:
- **사용자 요청 흐름** = REST 또는 gRPC (동기)
- **부수효과 / 통보** = Event (비동기)
- **streaming / 실시간** = WebSocket / SSE

---

## 5. 어떤 시점에 무엇을 결정하나

```
시점          배포 단위         Repo            DB                 통신
────         ─────────         ────            ──                 ────
0~5명         Monolith          Monorepo        Shared             REST 내부 모듈
5~15명        Modular Mono      Monorepo        Shared             모듈 함수 호출
15~30명       Modular Mono      Mono / Hybrid   Shared + replica   REST + Event 시작
30~50명       MSA 일부 추출     Hybrid          분리 시작           REST + Event
50명+         MSA               Hybrid / Multi  DB-per-service     REST/gRPC + Event
```

**가장 흔한 사고**: 5명 팀이 MSA 부터 시작 → 인프라 7개 + Kafka + 분산 트레이싱 운영 + 사람 부족 → 6개월 잃음.

---

## 6. 이 폴더의 노트 구조

```
architecture/
├── architecture.md                         ← 본 hub
├── monorepo-vs-multirepo.md                ← 결정 + 도구 + 마이그레이션
├── msa-overview.md                         ← MSA 언제·어떻게
├── modular-monolith.md                     ← MSA 가기 전 중간단계
├── saga-pattern.md                         (다음)
├── transactional-outbox.md                 (다음)
├── event-driven-architecture.md            (다음)
├── api-gateway.md                          (다음)
├── service-mesh.md                         (다음)
├── distributed-tracing.md                  (다음)
├── feature-flags.md                        (다음)
├── deployment-blue-green-canary.md         (다음)
└── multi-tenancy.md                        (다음)
```

진행 상태:

| 노트 | 상태 |
| --- | --- |
| [[monorepo-vs-multirepo]] | ✅ |
| [[msa-overview]] | 🟡 |
| [[modular-monolith]] | 🟡 |
| saga-pattern | 🟡 |
| transactional-outbox | 🟡 |
| event-driven-architecture | 🟡 |
| api-gateway | 🟡 |
| service-mesh | 🟡 |
| distributed-tracing | 🟡 |
| feature-flags | 🟡 |
| deployment-blue-green-canary | 🟡 |
| multi-tenancy | 🟡 |

---

## 7. 안티패턴 모음

### A1 — Distributed Monolith
서비스를 잘랐는데 **모두 동기 호출 체인** + **공유 DB**. 배포는 따로지만 한 서비스 죽으면 다 죽음. MSA 의 모든 비용 + 모놀리식의 모든 결합도.

### A2 — Premature MSA
초기 팀에서 "확장 대비" 라며 MSA. 도메인 경계도 모르는 상태에서 자른 경계 → 6개월 뒤 재분리.

### A3 — Shared Library Hell
공유 라이브러리에 모든 도메인 코드 → 라이브러리 버전 충돌 / 변경 시 모든 서비스 빌드. **`shared/common`** 은 진짜 framework-level 만.

### A4 — Database as Integration
서비스 A 가 서비스 B 의 DB 직접 SELECT. **DB 는 사적**. 통합은 API 또는 Event 로만.

### A5 — Synchronous Saga
주문 → 결제 → 재고 → 배송 모두 동기 호출. 1초 × 4 = 4초 + 한 단계라도 실패하면 sync rollback 지옥. **Event 기반 saga** 권장.

### A6 — Monorepo without Tooling
거대한 monorepo + 도구 없음 = 매 PR 마다 모든 테스트 30분. **Nx / Bazel / Turborepo** 등으로 affected projects 만 빌드.

### A7 — Multirepo without SemVer
서비스 100개 + 공유 라이브러리 v1.2.3 인지 v1.2.4 인지 아무도 모름. semver + 의존성 lockfile 필수.

---

## 8. 학습 자료

- **Building Microservices** (2nd) — Sam Newman (1순위)
- **Monolith to Microservices** — Sam Newman
- **Microservices Patterns** — Chris Richardson
- **Software Architecture: The Hard Parts** — Mark Richards & Neal Ford
- **Domain-Driven Design** — Eric Evans (DDD 원전)
- **Implementing DDD** — Vaughn Vernon (실전)
- **Accelerate** — Forsgren / Humble / Kim (조직 + 배포)
- **microservices.io** — Chris Richardson 의 카탈로그

---

## 9. 관련

- [[../spring/spring|↗ Spring]] — Spring Cloud 가 MSA 구현체
- [[../../database/database|↗ database hub]] — DB-per-service / replication
- [[../../devops/devops|↗ devops]] — 배포 / k8s / 관측성
- [[../backend|↑ backend]]
