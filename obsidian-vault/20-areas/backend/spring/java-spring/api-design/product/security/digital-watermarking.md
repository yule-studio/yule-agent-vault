---
title: "디지털 워터마크 — PDF 마스킹 + 추적 + 토큰"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T19:04:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - security
  - watermark
  - digital
---

# 디지털 워터마크 — PDF 마스킹 + 추적 + 토큰 ★

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[security|↑ hub]]**

---

## 1. 보안 레이어

```
[원본 PDF]                            ← admin only, GDrive private
   ↓
[Visible watermark]                   ← 페이지 footer (옅게)
   ↓
[Invisible metadata]                  ← PDF /Custom dict
   ↓
[Hash chain]                          ← userId + orderId + timestamp 의 sha256
   ↓
[Download token + 한도]              ← 7d 만료 + 5회 한도
   ↓
[GDrive presigned (1h TTL)]
```

---

## 2. Visible watermark

```
페이지 footer (작게, 회색 RGB 200/200/200, font 8pt):
"Purchased by [user@xxxxx.com] on 2026-05-14 | Order: ORD-2026-A1B2C3 | UID: 01HX..."
```

### 2.1 왜 4가지 정보

- email partial → 누가 (개인 식별 X but 추적 가능).
- order_id → 언제 (시점).
- userId (ULID) → 누가 (DB 조회).
- date → 언제.

### 2.2 왜 옅은 회색

- 가독성 ↓ (사용자 짜증 ↓).
- 검출 / 추적 OK (sharp 한 이미지에서는 보임).

---

## 3. Invisible metadata

```java
var meta = doc.getDocumentInformation();
meta.setCustomMetadataValue("watermark_hash", watermarkInfo.hash());
meta.setCustomMetadataValue("user_id", userId);
meta.setCustomMetadataValue("order_id", orderId);
meta.setCustomMetadataValue("purchased_at", purchasedAt.toString());
```

### 3.1 왜 metadata

- 사용자가 visible footer crop / 가린 후 유출해도 metadata 가 남음.
- `pdftk dump_data` 또는 `pypdf` 로 추출 가능.

### 3.2 한계

- attacker 가 PDF 재생성 (스캔 → OCR) 시 metadata 사라짐.
- → visible watermark 가 1차, metadata 는 2차.

---

## 4. Hash chain

```java
public record WatermarkInfo(UserId userId, OrderId orderId, String emailMasked,
                            Instant purchasedAt, String hash) {
    public static WatermarkInfo of(UserId u, OrderId o, String email, Instant t) {
        var input = u.value() + "|" + o.value() + "|" + email + "|" + t.toString();
        var hash = sha256(input + serverPepper);   // server-side secret
        return new WatermarkInfo(u, o, maskEmail(email), t, hash);
    }
}
```

→ `digital_deliveries.watermark_hash` 컬럼에 저장.

### 4.1 사용

```
유출된 PDF 발견:
1. PDF metadata 추출 (watermark_hash)
2. DB 의 digital_deliveries WHERE watermark_hash = ?
3. user_id / order_id 추적 → 법적 조치
```

---

## 5. Download token

```java
public record DownloadToken(String raw, String hash, Instant expiresAt) {
    public static DownloadToken generate(DeliveryId did, UserId uid, Instant now) {
        var raw = base64(hmac(did.value() + uid.value() + now.toEpochMilli(), serverSecret));
        var hash = sha256(raw);
        return new DownloadToken(raw, hash, now.plus(Duration.ofDays(7)));
    }
}
```

→ DB 에 hash 만 저장 (raw 는 사용자에게만). Download 시 hash 로 검증.

### 5.1 다운로드 한도

```
download_max = 5
download_count++ at each access
초과 시 401
```

→ 5회 = 가족 / 다른 기기 정도 — 무한 공유 방지.

---

## 6. 함정

### 함정 1 — visible 만 (metadata X)
visible crop 후 유출 → 추적 X.
→ metadata + hash 다중.

### 함정 2 — metadata 만 (visible X)
attacker 가 PDF metadata 삭제 → 추적 X.
→ visible + metadata.

### 함정 3 — hash 에 server secret 없음
attacker 가 hash 위조.
→ HMAC + pepper.

### 함정 4 — token raw 저장
DB 유출 시 도용.
→ hash 만.

### 함정 5 — 한도 / 만료 없음
무한 공유.
→ 5회 + 7일.

### 함정 6 — 사용자 IP / UA log 없음
도용 시 어디서 접근했는지 X.
→ download audit log.

---

## 7. 다른 컨텍스트

### 7.1 영상 (Netflix-style)
- per-session subtitle overlay (재생 시 ID 표시).
- DRM (Widevine / FairPlay).
- streaming → 다운로드 X.

### 7.2 음원
- audio steganography (미세 noise pattern).
- player 강제.

### 7.3 SW / 라이센스
- 시리얼 키 (활성화 서버 의존).

---

## 8. 관련

- [[security|↑ hub]]
- [[../design-decisions/digital-delivery-policy]]
- [[../database/digital-deliveries-table]]
- [[../implementation/digital-delivery-impl]]
- [[../domain-model/value-objects#watermarkinfo]]
