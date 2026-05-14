---
title: "chat requirements — 60 AC"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T23:48:00+09:00
tags: [backend, java-spring, api-design, chat, requirements]
---

# chat requirements — 60 AC

**[[chat|↑ hub]]**

---

## 1. Room (방)

| # | AC | 우선 |
| --- | --- | --- |
| R-1 | 1:1 (DIRECT) 방 생성 — 두 user 의 unique pair | P0 |
| R-2 | 그룹 (GROUP) 방 생성 — N 명 + 방장 | P0 |
| R-3 | 오픈채팅 (OPEN) 방 — 비밀번호 / 초대링크 | P1 |
| R-4 | 방장 / admin / member 역할 (그룹) | P0 |
| R-5 | 멤버 초대 / 퇴장 / 강퇴 | P0 |
| R-6 | 방 이름 / 프로필 (그룹) | P0 |
| R-7 | 방 검색 (오픈채팅) | P1 |
| R-8 | 방 max member (그룹 500 / 오픈 1500) | P0 |

## 2. Message (메시지)

| # | AC | 우선 |
| --- | --- | --- |
| M-1 | TEXT 메시지 send | P0 |
| M-2 | IMAGE / VIDEO / AUDIO / FILE 메시지 send (S3 presigned) | P0 |
| M-3 | EMOJI / 이모티콘 메시지 | P1 |
| M-4 | 답글 (reply-to) — 메시지 인용 | P1 |
| M-5 | 메시지 5분 안 본인 삭제 가능 | P0 |
| M-6 | 메시지 영구 보관 (사용자 삭제만) | P0 |
| M-7 | XSS sanitization (TEXT) | P0 |
| M-8 | 욕설 필터 (옵션) | P2 |
| M-9 | 사용자 차단 시 메시지 send X | P0 |
| M-10 | 메시지 1 user 의 spam (10/s) → throttle | P0 |
| M-11 | REST 페이징 (cursor-based) | P0 |
| M-12 | 검색 (텍스트, 본 room 안) | P1 |

## 3. 읽음 표시 ★

| # | AC | 우선 |
| --- | --- | --- |
| RD-1 | 1:1 — 카톡 "1" → "0" (상대가 봤음) | P0 |
| RD-2 | 그룹 — "1" → "0" 까지 점진 (안 본 사람 수) | P0 |
| RD-3 | 읽음 표시는 사용자별 per-message READ row | P0 |
| RD-4 | 본 사람 list 보기 (그룹) | P1 |
| RD-5 | 읽음 표시 OFF 옵션 (사용자 preference) | P2 |

## 4. 멀티 디바이스 ★

| # | AC | 우선 |
| --- | --- | --- |
| MD-1 | 1 user 가 동시 N WebSocket session (폰 / 태블릿 / 웹) | P0 |
| MD-2 | A 가 폰에서 보낸 메시지 → 태블릿에서도 즉시 보임 | P0 |
| MD-3 | A 가 폰에서 읽음 → 태블릿에서도 읽음 반영 | P0 |
| MD-4 | Logout 시 해당 device 의 WebSocket session 종료 | P0 |
| MD-5 | Cross-node (서버 A 의 session → 서버 B 의 session) | P0 |

## 5. Presence

| # | AC | 우선 |
| --- | --- | --- |
| P-1 | ONLINE / AWAY / OFFLINE 상태 | P0 |
| P-2 | "마지막 접속 시각" 표시 (사용자 preference) | P1 |
| P-3 | typing indicator ("입력 중...") | P1 |
| P-4 | Presence Redis TTL 1분 (heartbeat) | P0 |

## 6. Push (offline)

| # | AC | 우선 |
| --- | --- | --- |
| PUSH-1 | offline user 의 메시지 → notification 모듈 호출 | P0 |
| PUSH-2 | 사용자 preference 의 chat 알림 OFF 시 push X | P0 |
| PUSH-3 | quiet hours 시 push 보류 (notification 모듈) | P1 |
| PUSH-4 | 같은 room 의 5초 burst → debounce ("3 새 메시지") | P1 |

## 7. 첨부

| # | AC | 우선 |
| --- | --- | --- |
| A-1 | 사진 (max 10MB) + 자동 thumbnail | P0 |
| A-2 | 동영상 (max 200MB) | P1 |
| A-3 | 음성 (max 30s) | P1 |
| A-4 | 파일 (max 100MB) | P1 |
| A-5 | S3 presigned upload | P0 |
| A-6 | virus scan (ClamAV — 옵션) | P2 |
| A-7 | 30일 후 cold storage 이동 | P2 |

## 8. 보안 / 운영

| # | AC | 우선 |
| --- | --- | --- |
| S-1 | WebSocket CONNECT 시 JWT 검증 | P0 |
| S-2 | Origin 검증 (CSRF) | P0 |
| S-3 | Rate limit 1 user 메시지 10/s | P0 |
| S-4 | 차단 user 의 모든 메시지 receive 필터 | P0 |
| S-5 | 신고 / 자동 모더 (그룹) | P1 |
| S-6 | Audit (admin action 5년) | P0 |
| S-7 | active WebSocket connection 모니터링 | P0 |
| S-8 | message throughput / latency p95 | P0 |

## 9. 비기능

| 항목 | 목표 |
| --- | --- |
| WebSocket CONNECT p95 | < 500ms |
| 메시지 send → 다른 user receive | < 1s (online) |
| 동시 connection | 10,000 / 노드 |
| 메시지 send TPS | 500 / 노드 |
| 가용성 | 99.9% |

---

## 10. Phase 매핑

| Phase | AC |
| --- | --- |
| F1 | R-1, M-1, S-1, S-2 |
| F2 | M-5, M-6, M-7, M-11, RD-1 |
| F3 | R-2, R-4~R-6, RD-2 |
| F4 | MD-1~MD-5 |
| F5 | P-1~P-4 |
| F6 | A-1~A-5 |
| F7 | PUSH-1~PUSH-4 |
| F8 | R-3, R-7, M-12, M-9, S-4, S-5 |
| F9 | S-6~S-8 |
| F10 | Kafka |

---

## 11. 관련

- [[chat|↑ hub]]
- [[implementation-order]]
- [[testing/test-scenarios]]
