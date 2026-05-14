---
title: "notification prerequisites — 외부 서비스 발급"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T21:08:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - notification
  - prerequisites
---

# notification prerequisites — 외부 서비스 발급

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[notification|↑ hub]]**

---

## 1. 기술 전제

| 항목 | 버전 / 도구 | 비고 |
| --- | --- | --- |
| Java | 21 LTS | virtual thread (worker scale-out) |
| Spring Boot | 3.3.x | |
| PostgreSQL | 16 | outbox + dedup |
| Redis | 7 | rate limit + idempotency |
| Auth | signup recipe | [[../signup/security/security]] |

---

## 2. FCM (Firebase Cloud Messaging — Android + 일부 iOS)

### 2.1 발급 절차

```
1. console.firebase.google.com 프로젝트 생성
2. Project Settings → Service accounts → Generate new private key (JSON)
3. JSON 을 AWS Secrets Manager / Vault 에 저장
4. Android app 에 google-services.json 통합 (FE 담당)
5. iOS app 에 GoogleService-Info.plist 통합
```

### 2.2 라이브러리

```kotlin
dependencies {
    implementation("com.google.firebase:firebase-admin:9.3.0")
}
```

### 2.3 비용

- 무료 (현재 Firebase Cloud Messaging unlimited).
- → 채택 1순위.

### 2.4 한계

- topic 당 25개 subscriber, project 당 200만 token.
- API rate: ~500 req/s default (증액 가능).

자세히: [[implementation/fcm-channel-impl]].

---

## 3. APNs (Apple Push Notification service — iOS)

### 3.1 발급 절차

```
1. Apple Developer (developer.apple.com) 계정 ($99/년)
2. Certificates → APNs Auth Key (.p8) 생성
3. Key ID + Team ID + Bundle ID + .p8 파일 보관 (KMS / Vault)
4. iOS app 의 Bundle ID 와 일치 (FE 담당)
```

### 3.2 인증 방식

- **Token-based (JWT)** ★ — .p8 key, 1년 영구.
- Certificate-based (.p12) — legacy, 1년 만료.
- → 본 vault: token-based.

### 3.3 라이브러리

```kotlin
dependencies {
    implementation("com.eatthepath:pushy:0.15.4")
}
```

또는 직접 HTTP/2 호출 (Java HttpClient + JWT 서명).

### 3.4 비용

- 무료 (Apple Developer 계정 비용 만).

### 3.5 환경 분리

- sandbox: `api.sandbox.push.apple.com` (개발 / TestFlight)
- production: `api.push.apple.com`

자세히: [[implementation/apns-channel-impl]].

---

## 4. AWS SES (Simple Email Service)

### 4.1 발급 절차

```
1. AWS Console → SES 서비스
2. Verify Domain (DKIM / SPF / DMARC 설정 필수)
3. Verify From email (admin@example.com)
4. Sandbox → Production 신청 (Verify 후 사용량 제출)
5. IAM user / role 생성 (ses:SendEmail / ses:SendRawEmail)
```

### 4.2 라이브러리

```kotlin
dependencies {
    implementation("software.amazon.awssdk:ses:2.27.0")
}
```

### 4.3 비용

- 첫 62,000 / 월 무료 (EC2 발송 시).
- 그 후 $0.10 / 1000 emails.

### 4.4 sandbox 한계

- Production 승인 전 — verify 한 이메일에만 발송.

자세히: [[implementation/ses-email-channel-impl]].

---

## 5. WebPush (Web)

### 5.1 표준 (RFC 8030)

- 브라우저: Chrome / Firefox / Safari 16.4+ / Edge.
- VAPID 키 쌍 (서버 ↔ 브라우저 인증).

### 5.2 발급

```bash
# VAPID key pair 생성 (한 번)
npm install -g web-push
web-push generate-vapid-keys
```

### 5.3 라이브러리

```kotlin
dependencies {
    implementation("com.maximjas:webpush-java:1.1.0")
    // 또는: nl.martijndwars:web-push:5.1.1
}
```

### 5.4 비용

- 무료.

### 5.5 한계

- 브라우저 종속 (iOS Safari 16.4+ — 비교적 최근).

자세히: [[implementation/webpush-channel-impl]].

---

## 6. Slack (Admin)

### 6.1 발급

```
1. api.slack.com → Create App
2. Incoming Webhooks 활성화 → 채널별 webhook URL
3. URL 을 Vault 저장
```

### 6.2 라이브러리

- 본 vault: 단순 `RestTemplate` POST (별도 lib X).

### 6.3 비용

- 무료 (workspace plan 별 한도).

### 6.4 용도

- admin alert 전용 (사용자 알림 X).
- payment-alerts / fraud / admin-actions 채널.

자세히: [[implementation/slack-channel-impl]].

---

## 7. 채널 별 매트릭스

| 채널 | 사용자 알림 | admin 알림 | 비용 | 한계 |
| --- | --- | --- | --- | --- |
| **FCM** | Android (필수) + iOS (대안) | X | 무료 | rate limit |
| **APNs** | iOS (필수) | X | $99/년 (developer) | sandbox 분리 |
| **WebPush** | Web 사용자 | X | 무료 | 일부 브라우저 |
| **SES** | Email 중요 알림 (영수증 / 보안) | X | $0.10/1k | sandbox 승인 |
| **Slack** | X | admin 운영 | 무료 | webhook 한도 |
| **In-App** | 사용자 화면 목록 | X | DB row | 사용자 직접 열어야 봄 |

---

## 8. 시작 전 체크리스트

- [ ] FCM 프로젝트 + service account JSON (Vault 저장)
- [ ] APNs Auth Key (.p8) + Key ID + Team ID + Bundle ID
- [ ] AWS SES domain verify + DKIM/SPF/DMARC + production 승인
- [ ] WebPush VAPID key pair
- [ ] Slack webhook URL (per-channel)
- [ ] signup recipe F0~F8 (인증 통합)
- [ ] PostgreSQL 16 + Redis 7
- [ ] AWS Secrets Manager (모든 key 저장)
- [ ] notification 도메인 / 정책 문서 (법무 + UX 검토)

---

## 9. 관련

- [[notification|↑ hub]]
- [[requirements]]
- [[architecture]]
- [[design-decisions/channel-selection]]
- [[security/device-token-rotation]]
- [[../signup/prerequisites|↗ signup prerequisites]]
