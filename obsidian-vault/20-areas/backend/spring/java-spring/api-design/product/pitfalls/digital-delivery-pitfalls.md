---
title: "Digital delivery 함정 — 워터마크 / 토큰 / 도용"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T20:18:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - pitfalls
  - digital
---

# Digital delivery 함정 — 워터마크 / 토큰 / 도용 ★

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[pitfalls|↑ hub]]**

---

## 함정 1 — 원본 PDF public URL
SNS 공유 → 매출 0.
→ private + 워터마크 PDF 만 사용자에게.

## 함정 2 — 워터마크 없음
유출 시 추적 X.
→ visible footer + metadata + hash.

## 함정 3 — 사용자별 사본 안 만듦 (원본 공유)
같은 PDF — 도용 추적 X.
→ per-user copy.

## 함정 4 — 워터마크 동기 (트랜잭션 안)
100MB PDF — DB 락.
→ AFTER_COMMIT + worker.

## 함정 5 — Worker 재시도 무한
GDrive 일시 장애 → 무한 fail.
→ max 5 + exp backoff + DEAD_LETTER.

## 함정 6 — Worker 동시 같은 row
2 worker pickup.
→ SKIP LOCKED + ShedLock.

## 함정 7 — token raw DB 저장
DB 유출 시 도용.
→ hash 만.

## 함정 8 — 다운로드 한도 / 만료 없음
무한 공유.
→ 5회 + 7일.

## 함정 9 — 환불 시 access 안 닫음
부당 이득.
→ AFTER_COMMIT permission revoke.

## 함정 10 — 환불 시 file 즉시 삭제
evidence X.
→ permission 만 / 30일 후 file.

## 함정 11 — visible 만 (metadata X)
crop 후 유출 → 추적 X.
→ multi-layer.

## 함정 12 — metadata 만 (visible X)
metadata 삭제 시 추적 X.
→ multi-layer.

## 함정 13 — server_secret pepper 없음
attacker hash 위조.
→ HMAC + pepper.

## 함정 14 — GDrive permission anyoneWithLink
검색엔진 indexing.
→ service account + 사용자 reader.

## 함정 15 — 사용자 폴더 분리 X
같은 폴더 — 사용자 A 가 B 의 사본 봄.
→ folder per user.

## 함정 16 — 다운로드 audit 없음
도용 시 어디서 접근?
→ IP / UA / time log.

## 함정 17 — 워터마크 PDF 매번 재생성
worker 비용 ↑.
→ 한 번 만 + driveFileId 재사용.

## 함정 18 — PDF 크기 폭증 (PDFBox 비효율)
100MB → 500MB.
→ qpdf 압축.

## 함정 19 — checksum 검증 X
업로드 변조 / 손상 감지 X.
→ sha256.

---

## 관련

- [[pitfalls|↑ hub]]
- [[../design-decisions/digital-delivery-policy]]
- [[../security/digital-watermarking]]
- [[../implementation/digital-delivery-impl]]
- [[../database/digital-deliveries-table]]
