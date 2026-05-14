---
title: "Webhook 서명 검증 — HMAC-SHA-256"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T18:59:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - security
  - webhook
---

# Webhook 서명 검증 — HMAC-SHA-256 ★

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[security|↑ hub]]**

---

## 1. 핵심

```
PG 와 서버가 공유하는 secret 으로 HMAC 서명
→ webhook URL 공개해도 위조 불가
```

---

## 2. PG 별 서명 헤더

| PG | 헤더 | algorithm |
| --- | --- | --- |
| Toss | `tosspayments-signature` | HMAC-SHA-256 |
| KCP | body 안 (`siteCd + ...`) | HMAC-SHA-256 |
| Stripe | `Stripe-Signature` (timestamp + signature) | HMAC-SHA-256 |
| 이니시스 | IP whitelist + 단순 서명 | (구식) |

---

## 3. 검증 코드

```java
@Component
@RequiredArgsConstructor
public class TossSignatureVerifier {

    @Value("${pg.toss.webhook-secret}")
    private String secret;

    public boolean verify(String rawBody, String signature) {
        try {
            var mac = Mac.getInstance("HmacSHA256");
            mac.init(new SecretKeySpec(secret.getBytes(UTF_8), "HmacSHA256"));
            var computed = Base64.getEncoder().encodeToString(
                mac.doFinal(rawBody.getBytes(UTF_8)));
            return MessageDigest.isEqual(
                signature.getBytes(UTF_8),
                computed.getBytes(UTF_8));    // timing-safe 비교
        } catch (Exception e) {
            return false;
        }
    }
}
```

### 3.1 왜 MessageDigest.isEqual

- `String.equals()` = early-return → timing attack 가능.
- `MessageDigest.isEqual` = constant-time 비교.

---

## 4. Controller 통합

```java
@PostMapping("/webhooks/toss")
public ResponseEntity<Void> tossWebhook(
        @RequestBody String rawBody,
        @RequestHeader("tosspayments-signature") String signature) {

    if (!verifier.verify(rawBody, signature))
        return ResponseEntity.status(401).build();

    // 처리 ...
    return ResponseEntity.ok().build();
}
```

→ `@RequestBody String` (DTO 아님) — raw body 가 서명 계산 input.

---

## 5. 함정

### 함정 1 — DTO 로 받음 (deserialize)
서명 계산 input 이 변경 (JSON 직렬화 순서 / whitespace).
→ raw String 으로 받음.

### 함정 2 — String.equals 비교
timing attack.
→ MessageDigest.isEqual.

### 함정 3 — secret 평문 config
git 노출.
→ vault / AWS Secrets Manager.

### 함정 4 — IP 화이트리스트 만
PG IP 변경 시 webhook 못 받음.
→ HMAC 우선 + IP 보조.

### 함정 5 — 서명 검증 실패 시 200 응답
attacker 의 fake event 가 처리됨.
→ 401 + audit log.

---

## 6. 관련

- [[security|↑ hub]]
- [[../design-decisions/webhook-strategy]]
- [[../database/webhook-events-table]]
- [[../implementation/payment-webhook-impl]]
