---
title: "chat prerequisites — 시작 전 준비"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T23:45:00+09:00
tags: [backend, java-spring, api-design, chat, prerequisites]
---

# chat prerequisites — 시작 전 준비

**[[chat|↑ hub]]**

---

## 1. 기술

| 항목 | 버전 / 도구 | 비고 |
| --- | --- | --- |
| Java | 21 LTS | virtual thread (WebSocket I/O) |
| Spring Boot | 3.3.x | |
| Spring WebSocket | 6.x | spring-boot-starter-websocket |
| STOMP broker | 내장 SimpleBroker (F0~F4) → **RabbitMQ relay (F5+)** | scale-out |
| PostgreSQL | 16 | messages partitioned by month |
| Redis | 7 | **Pub/Sub** (cross-node) + presence + rate limit |
| Auth | signup recipe + **Google OAuth2** | [[../signup/security/security]] · [[implementation/google-login-impl]] |
| 첨부 | S3 + CloudFront | [[../file-upload-s3]] |
| Push | notification 모듈 ([[../notification/notification]]) | offline fallback |

### 1.1 왜 STOMP (raw WebSocket 아님)

- Pub/Sub 의미적 destination (/topic/room/X) — 그룹 / broadcast 자연스러움.
- Spring 통합 + 표준 (CONNECT / SUBSCRIBE / SEND / DISCONNECT).
- 자세히: [[design-decisions/stomp-vs-raw]].

### 1.2 왜 Redis Pub/Sub

- 다중 서버 노드 환경에서 WebSocket session 분산.
- 노드 A 의 session 이 보낸 메시지 → 노드 B 의 session 에게 전달.
- 자세히: [[design-decisions/scale-strategy]].

### 1.3 왜 RabbitMQ relay (F5+)

- SimpleBroker = in-memory, 단일 노드.
- RabbitMQ STOMP relay = 분산 + 영속 큐 + clustering.
- 자세히: [[implementation/websocket-stomp-config]].

---

## 2. 외부 서비스 발급

| 서비스 | 용도 |
| --- | --- |
| **Google Cloud Console** | OAuth2 client (FE 로그인) — [[implementation/google-login-impl]] |
| **FCM / APNs** | offline push (notification 모듈 의존) |
| **AWS S3 + CloudFront** | 첨부 (사진/음성/동영상/파일) |
| **AWS RDS + ElastiCache** | DB + Redis |
| **RabbitMQ** (옵션) | STOMP relay (F5+) |

---

## 3. 도메인 결정

| 결정 | 본 vault |
| --- | --- |
| 1:1 / 그룹 / 오픈채팅 | 모두 (F1 / F3 / F8) |
| 1:1 vs 그룹 model | 동일 schema (room_type 컬럼) |
| 그룹 max member | 500 명 (카톡 단톡방 기본) |
| 오픈채팅 max | 1500 명 (카톡 오픈채팅 기본) |
| 멀티 디바이스 | F4 부터 |
| 읽음 표시 | per-user per-message (F2 부터) |
| 메시지 retention | 영구 (사용자 삭제 가능) |
| 첨부 retention | 30일 (cold storage 후) |
| E2EE | 본 vault X (option) |
| 차단 (block) | 양방향 + 메시지 send 차단 |

---

## 4. 보안

| 항목 | 적용 |
| --- | --- |
| WebSocket CONNECT auth | JWT in `nativeHeaders["Authorization"]` |
| CSRF | Origin / SameSite cookie 정책 |
| XSS | message content sanitization |
| Rate limit | 1 user 의 메시지 10 / s |
| Attachment scan | ClamAV (옵션) |
| 차단 user filter | 모든 message list / receive |

---

## 5. 인프라

| 항목 | 본 vault |
| --- | --- |
| Sticky session | nginx ip_hash 또는 AWS ALB sticky cookie |
| Cross-node | Redis Pub/Sub |
| Presence | Redis (DB X — fast) |
| 메시지 partition | PostgreSQL partition by created_at month |

---

## 6. 시작 전 체크리스트

- [ ] signup F0~F8 운영 중 (JWT 인증)
- [ ] notification 레시피 F0~F7 운영 중 (push fallback)
- [ ] Redis 7 (Pub/Sub 지원)
- [ ] Google OAuth2 client (Console)
- [ ] S3 + CloudFront (첨부)
- [ ] Sticky session 인프라 (nginx / ALB)
- [ ] WebSocket monitoring (Prometheus)
- [ ] 욕설 / 모더 정책 (운영팀)
- [ ] 신고 / block 정책 (CS)

---

## 7. 관련

- [[chat|↑ hub]]
- [[requirements]]
- [[architecture]]
- [[design-decisions/stomp-vs-raw]]
- [[design-decisions/scale-strategy]]
- [[implementation/websocket-stomp-config]]
