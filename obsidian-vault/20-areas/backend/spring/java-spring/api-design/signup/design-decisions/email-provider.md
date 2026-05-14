---
title: "이메일 발송 도구 — AWS SES 우선"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T04:25:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - design-decisions
  - email
  - ses
---

# 이메일 발송 도구 — AWS SES 우선

**[[design-decisions|↑ design-decisions hub]]**

> "이메일 어떻게 보내나" — 잘못 선택하면 **spam 폴더 직행 + 발송 비용 폭증**.

---

## 1. 본 vault 결정

- **AWS SES (서울 리전)** 우선.
- **Postmark** — 트랜잭션 메일 critical 케이스 (가입 / 결제 / 인증).
- **SendGrid** — 마케팅 / 대량 발송 시 (배달율 critical).

---

## 2. 옵션 비교 — 4구조

### 2.1 AWS SES (본 vault default)

**왜 이게 1순위**
- 가장 저렴 — $0.10 / 1,000 메일 (Postmark / SendGrid 의 1/100~1/200).
- AWS 인프라 통합 (서울 리전) — IAM / VPC / CloudTrail.
- bounce / complaint SNS / SQS 통합.
- 한국 SaaS 다수 채택.

**왜 안 되는 케이스**
- 신규 계정은 sandbox (sandbox 해제 1주 소요).
- 발송 평판 (reputation) 직접 관리 — bounce / complaint 처리 안 하면 IP blacklist.
- 마케팅 메일 / 대량 발송 시 평판 관리 부담 ↑.

**안 하면 무슨 문제**
- 평판 관리 안 하면 → AWS 가 발송 중단 (suspension) → 운영 사고.
- DKIM / SPF 설정 안 하면 → Gmail / Naver / Daum spam 폴더 직행.

**대안과 왜 안 됨**
- SendGrid — 비싸지만 평판 관리 자동 (warming, IP rotation).
- 자체 SMTP — IP 평판 0 에서 시작 → 도달율 망함.

**트레이드오프**
- 비용 ↓↓ vs 평판 관리 부담.
- 운영 / DevOps 가 모니터링 (bounce rate, complaint rate) 필수.

---

### 2.2 Postmark

**왜 적합한 케이스**
- 트랜잭션 메일 전문 (가입 / 결제 / 인증).
- 배달율 매우 좋음 — Postmark IP 평판 우수.
- 빠름 (수 초 내 발송) — 시간 critical 한 메일.

**왜 안 되는 케이스 (마케팅)**
- 마케팅 메일 거부 (Postmark 의 정책).
- 단일 IP / 도메인 평판 보호.

**대안과 왜 안 됨**
- SendGrid — 마케팅 + 트랜잭션 둘 다. Postmark 만큼 트랜잭션 specialized 아님.

**트레이드오프**
- 비용 ↑ ($15 / 10k vs SES 의 $1 / 10k).
- 트랜잭션 critical 한 경우만 가치 있음.

---

### 2.3 SendGrid

**왜 적합한 케이스**
- 마케팅 + 트랜잭션 둘 다.
- 40k / 월 무료 — 시작 단계.
- 다중 IP / warming 자동.

**왜 안 되는 케이스 (비용 critical)**
- 100만 메일 = $20+ (SES 의 1/100).
- 평판 관리는 자동이지만 비용 부담.

**트레이드오프**
- 평판 자동 vs 비용.
- 시작은 SendGrid 무료, 성장 후 SES 마이그레이션 흔한 패턴.

---

### 2.4 자체 SMTP (postfix / exim)

**왜 사용**
- 비용 0 (서버 비용만).
- 완전 제어.

**왜 안 됨 (한국 SaaS)**
- IP 평판 0 시작 → Gmail / Naver / Daum 이 spam 처리.
- Warming 수개월 + 모니터링 부담.
- bounce / complaint 처리 직접 구현.

**언제 적합**
- 사내 / B2B 한정 (특정 도메인만 발송).
- 평판 critical 하지 않음.

**트레이드오프**
- 비용 0 vs 운영 폭증.
- 거의 모든 SaaS 가 외부 서비스 사용.

---

### 2.5 NHN Cloud / 네이버 Cloud 이메일

**왜 적합한 케이스**
- 한국 사용자 우선 — 국내 도메인 도달율 ↑.
- 한국어 CS / 결제.

**왜 안 되는 케이스**
- 글로벌 도달율 (Gmail / Outlook) 은 AWS SES 가 ↑.
- 인지도 / 통합 부족.

**트레이드오프**
- 한국 도달율 ↑ vs 글로벌 도달율 ↓.

---

## 3. 비용 비교

| 도구 | 1k 메일 | 10k 메일 | 100k 메일 | 1M 메일 |
| --- | --- | --- | --- | --- |
| AWS SES | $0.10 | $1 | $10 | $100 |
| Postmark | ~$1.50 | $15 | $150 (옵션) | (협상) |
| SendGrid | $2 | $20 | $90 (Pro) | $900 |
| Mailgun | $3.50 | $35 | $90 (Foundation) | $900 |
| 자체 SMTP | $0 (서버) | $0 | $0 | $0 |
| NHN Cloud | ~$2 | $20 | $200 | (협상) |

**왜 SES 가 압도적**
- 100만 메일 = SES $100 vs SendGrid $900 — 9배 차이.
- 1000만 메일 = $1000 vs $9000.

---

## 4. 발송 시 필수 설정

### 4.1 DNS — SPF / DKIM / DMARC

**SPF (Sender Policy Framework)**
```
TXT @ "v=spf1 include:amazonses.com -all"
```
→ "이 도메인은 amazonses 만 발송 허용".

**DKIM (DomainKeys Identified Mail)**
```
TXT abc123._domainkey.example.com "v=DKIM1; k=rsa; p=<public-key>"
```
→ 발송 시 private key 로 서명, 수신측이 DNS 의 public key 로 검증.

**DMARC (Domain-based Message Authentication)**
```
TXT _dmarc.example.com "v=DMARC1; p=reject; rua=mailto:dmarc@example.com"
```
→ SPF / DKIM 실패 시 정책 (reject / quarantine / none).

### 4.2 왜 3개 모두 필요한가

- SPF 만 — 발송 IP 검증만. 헤더 spoofing 가능.
- DKIM 만 — 내용 변조 감지만. 발송 IP 무관.
- DMARC — SPF + DKIM 의 정책 강제.

**안 하면 무슨 문제**
- Gmail / Naver / Daum 이 spam 폴더 직행.
- 사용자가 인증 메일 못 받음 → 가입 실패 → CS 폭주.

### 4.3 IP Warming

```
Day 1: 50 emails
Day 2: 100
Day 3: 500
Day 4: 1,000
Day 5: 5,000
...
```

**왜 필요**
- 새 IP / 도메인 평판 0 시작.
- 갑작스러운 대량 발송 = spam 의심.
- 점진적 증량으로 평판 구축.

**언제 SES 가 자동 처리**
- Production 모드 (sandbox 해제) 후 SES 가 24h sending limit 설정.
- 사용량 ↑ 면 자동 limit 확대.

### 4.4 Bounce / Complaint Webhook

```
SES → SNS topic → SQS → Application
  → DB update (user.email_verified_at = null)
  → 해당 user 에게 발송 stop
```

**왜 필수**
- bounce rate > 5% → SES suspension.
- complaint rate > 0.1% → 평판 / suspension.
- 무효 email 에 계속 발송 = 평판 손상.

**안 하면 무슨 문제**
- 무효 email (잘못 입력 / 폐쇄 메일) 에 무한 발송 → 평판 망함 → 모든 사용자에게 spam 폴더.

---

## 5. application 구현

```java
@Component
public class SesEmailClient implements EmailClient {
    private final SesV2Client ses;

    @Override
    public SendResult send(String toEmail, String template, Map<String, Object> payload) {
        var request = SendEmailRequest.builder()
            .fromEmailAddress("noreply@example.com")
            .destination(d -> d.toAddresses(toEmail))
            .content(c -> c.simple(b -> b
                .subject(t -> t.data(subject(template)))
                .body(body -> body
                    .html(t -> t.data(renderHtml(template, payload)))
                    .text(t -> t.data(renderText(template, payload))))))
            .build();
        var response = ses.sendEmail(request);
        return SendResult.ok(response.messageId());
    }
}
```

**왜 HTML + Text 둘 다**
- 일부 클라이언트 (text-only) 또는 spam filter 가 text 만 분석.
- Text fallback 없으면 spam score ↑.

**왜 messageId 저장**
- bounce / complaint webhook 매핑.

자세히: [[../database/email-outbox-table]] · [[../email-verification-impl]].

---

## 6. Spring Boot 통합

```kotlin
// build.gradle.kts
implementation("org.springframework.boot:spring-boot-starter-mail")
implementation("software.amazon.awssdk:sesv2:2.25.50")
```

**왜 SES SDK 직접 (spring-mail 만 X)**
- spring-mail = JavaMailSender (SMTP).
- SES SDK = HTTPS API (서명 / 인증 자동, throughput ↑).

**언제 spring-mail 충분**
- 자체 SMTP / Gmail relay 같은 SMTP 환경.
- SES 의 SMTP endpoint 도 사용 가능 (단 SDK 가 더 효율).

---

## 7. 함정 모음

### 함정 1 — SPF / DKIM / DMARC 안 설정
spam 폴더 직행. 사용자가 인증 메일 못 받음.
→ 3개 모두 + DMARC `p=quarantine` 시작.

### 함정 2 — Bounce / complaint 처리 안 함
무효 email 무한 발송 → 평판 망함 → SES suspension.
→ SNS webhook 처리 필수.

### 함정 3 — HTML 만 발송
text 클라 / spam filter 가 spam 처리.
→ HTML + Text 둘 다.

### 함정 4 — 마케팅 + 트랜잭션 같은 도메인
마케팅 complaint 가 트랜잭션 메일 평판도 망침.
→ 서브도메인 분리 (`mail.example.com` vs `auth.example.com`).

### 함정 5 — From 주소가 `noreply@`
사용자가 답변 시도 → bounce → 평판 ↓.
→ `noreply@` + `Reply-To: support@`.

### 함정 6 — IP Warming 없이 대량 발송
새 IP 가 갑자기 100만 메일 = spam 의심.
→ Day 1 ~ Day 7 점진 증량.

### 함정 7 — 단일 IP / 도메인
한 메일이 평판 망치면 모두 영향.
→ 다중 IP / 서브도메인 분리.

### 함정 8 — 응답 시간 critical (가입 메일)
1분 후 도착 = 사용자 이탈.
→ Postmark (수 초) 또는 SES + outbox + worker 빠른 polling.

### 함정 9 — 글로벌 user 발송 (한국 서버)
latency ↑ + 일부 region 도달 실패.
→ SES 다중 region 또는 SendGrid.

### 함정 10 — sandbox 해제 안 하고 prod 배포
SES sandbox = 검증된 주소만 발송 → prod 에서 모든 사용자에게 안 감.
→ sandbox 해제 1주 전 신청.

### 함정 11 — Rate limit 없음 (재발송 abuse)
악의적 user 가 무한 재발송 → SES 비용 / 평판 손상.
→ Bucket4j + outbox 의 rate limit.

### 함정 12 — `[Yule]` 같은 brackets 없는 제목
일부 spam filter 가 brackets / 강조 텍스트 의심.
→ 단순 제목 + 본문 의 brackets 사용.

---

## 8. 다른 컨텍스트

### 8.1 글로벌

```yaml
email-provider:
  primary: aws-ses-multi-region
  fallback: sendgrid
  reason: region 별 latency / 도달율
```

### 8.2 트랜잭션 critical (금융)

```yaml
email-provider:
  primary: postmark
  fallback: aws-ses
  reason: 가입 / 결제 메일 도달율 critical
```

### 8.3 마케팅 우선

```yaml
email-provider:
  primary: sendgrid (marketing)
  transactional: aws-ses
  reason: 별도 서브도메인 + 평판 분리
```

---

## 9. 관련

- [[design-decisions|↑ hub]]
- [[../database/email-outbox-table]] — outbox 패턴 + worker
- [[../email-verification-impl]]
- [[../password-reset-impl]]
- 외부 — AWS SES Developer Guide, OWASP Email Authentication
