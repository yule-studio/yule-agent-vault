---
title: "DigitalDelivery Aggregate"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T18:20:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - domain-model
  - aggregate
  - digital
---

# DigitalDelivery Aggregate ★

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[domain-model|↑ hub]]**

> 사용자별 마스킹 사본 — 결제 후 발급 → 다운로드 → 환불 시 revoke 의 lifecycle.

---

## 1. 구조

```
DigitalDelivery (root)
  - orderId / orderItemId / userId / assetId
  - status
  - storageFileId
  - watermarkHash
  - downloadToken (hashed)
  - downloadCount / max
  - revoke 정보
```

---

## 2. Java

```java
public final class DigitalDelivery {
    private final DeliveryId id;
    private final OrderId orderId;
    private final OrderItemId orderItemId;
    private final UserId userId;
    private final AssetId assetId;
    private DeliveryStatus status;            // PENDING/PROCESSING/READY/REVOKED/FAILED
    private String storageProvider;
    private String storageFileId;
    private String watermarkHash;
    private String downloadTokenHash;
    private int downloadCount;
    private final int downloadMax;
    private Instant tokenExpiresAt;
    private Instant deliveredAt;
    private Instant firstDownloadedAt;
    private Instant lastDownloadedAt;
    private Instant revokedAt;
    private String revokeReason;
    private int attempts;
    private String lastError;
    private Instant nextAttemptAt;
    private final List<DomainEvent> events = new ArrayList<>();

    public static DigitalDelivery initiate(DeliveryId id, OrderId orderId,
                                           OrderItemId itemId, UserId userId,
                                           AssetId assetId, int downloadMax,
                                           Instant now) {
        return new DigitalDelivery(/* ... PENDING */);
    }

    public void startProcessing(Instant now) {
        require(status == DeliveryStatus.PENDING || status == DeliveryStatus.FAILED);
        this.status = DeliveryStatus.PROCESSING;
        this.attempts++;
    }

    public void markReady(String provider, String fileId, String watermarkHash,
                          String tokenHash, Instant tokenExpires, Instant now) {
        require(status == DeliveryStatus.PROCESSING);
        this.storageProvider = provider;
        this.storageFileId = fileId;
        this.watermarkHash = watermarkHash;
        this.downloadTokenHash = tokenHash;
        this.tokenExpiresAt = tokenExpires;
        this.deliveredAt = now;
        this.status = DeliveryStatus.READY;
        events.add(new DigitalDeliveryReady(id, userId, fileId, now));
    }

    public void recordFailure(String error, Instant nextAttempt) {
        if (attempts >= 5) {
            this.status = DeliveryStatus.FAILED;
            events.add(new DigitalDeliveryFailed(id, error, attempts));
        } else {
            this.lastError = error;
            this.nextAttemptAt = nextAttempt;
            this.status = DeliveryStatus.PENDING;
        }
    }

    public void download(Instant now) {
        require(status == DeliveryStatus.READY);
        require(revokedAt == null, "revoked");
        require(now.isBefore(tokenExpiresAt), "token expired");
        require(downloadCount < downloadMax, "download limit reached");
        downloadCount++;
        if (firstDownloadedAt == null) firstDownloadedAt = now;
        lastDownloadedAt = now;
    }

    public void revoke(String reason, Instant now) {
        if (status == DeliveryStatus.REVOKED) return;
        this.status = DeliveryStatus.REVOKED;
        this.revokedAt = now;
        this.revokeReason = reason;
        this.downloadTokenHash = null;
        events.add(new DigitalDeliveryRevoked(id, reason, now));
    }
}
```

---

## 3. 불변식

- READY 진입 = storage_file_id / watermark_hash / token 모두 set.
- attempts 5회 → FAILED.
- downloaded 후 firstDownloadedAt fixed.
- revoke 후 token = null.

---

## 4. 이벤트

- DigitalDeliveryReady
- DigitalDeliveryFailed
- DigitalDeliveryRevoked

---

## 5. 관련

- [[domain-model|↑ hub]]
- [[../database/digital-deliveries-table]]
- [[../design-decisions/digital-delivery-policy]]
- [[../security/digital-watermarking]]
- [[../implementation/digital-delivery-impl]]
