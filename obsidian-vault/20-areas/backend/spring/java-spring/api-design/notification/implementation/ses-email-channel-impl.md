---
title: "SES Email Channel 구현 ★"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T22:52:00+09:00
tags: [backend, java-spring, api-design, notification, implementation, email, ses]
---

# SES Email Channel 구현 ★

**[[implementation|↑ hub]]**

---

## 1. 설정

```yaml
aws:
  ses:
    region: ap-northeast-2
    from-email: noreply@example.com
    from-name: YULE
    reply-to: support@example.com
    configuration-set: yule-prod    # bounce / complaint 추적
```

---

## 2. 코드 (AWS SDK v2)

```java
@Component
@RequiredArgsConstructor
public class SesEmailChannel implements NotificationChannel {

    private final SesV2Client ses;
    private final UserRepository users;
    private final SesProperties props;
    private final Clock clock;

    public ChannelType type() { return ChannelType.EMAIL; }

    public DevicePlatform[] supportedPlatforms() {
        return new DevicePlatform[] {};   // device 무관 (user email)
    }

    public DeliveryResult send(NotificationPayload payload, UserDevice device) {
        // device 는 null (이메일은 user.email 사용)
        // 호출자가 user.email 을 payload 에 담아 보내거나 router 가 lookup
        var toEmail = payload.data().get("toEmail");
        if (toEmail == null) {
            return DeliveryResult.permanent(ChannelType.EMAIL,
                "NO_EMAIL", "user has no email", clock.now());
        }

        var request = SendEmailRequest.builder()
            .fromEmailAddress("%s <%s>".formatted(props.fromName(), props.fromEmail()))
            .replyToAddresses(props.replyTo())
            .destination(Destination.builder()
                .toAddresses(toEmail)
                .build())
            .content(EmailContent.builder()
                .simple(Message.builder()
                    .subject(Content.builder().data(payload.title()).charset("UTF-8").build())
                    .body(Body.builder()
                        .html(Content.builder().data(payload.body()).charset("UTF-8").build())
                        .build())
                    .build())
                .build())
            .configurationSetName(props.configurationSet())
            .emailTags(MessageTag.builder().name("notif_type")
                .value(payload.data().get("notificationType")).build())
            .build();

        try {
            var response = ses.sendEmail(request);
            return DeliveryResult.success(ChannelType.EMAIL,
                response.messageId(), clock.now());

        } catch (AccountSuspendedException | MailFromDomainNotVerifiedException e) {
            slack.alert("SES account / domain issue", e.getMessage());
            return DeliveryResult.permanent(ChannelType.EMAIL,
                e.getClass().getSimpleName(), e.getMessage(), clock.now());

        } catch (SendingPausedException | TooManyRequestsException e) {
            return DeliveryResult.transient(ChannelType.EMAIL,
                e.getClass().getSimpleName(), e.getMessage(), clock.now());

        } catch (Exception e) {
            return DeliveryResult.transient(ChannelType.EMAIL,
                "EX", e.getMessage(), clock.now());
        }
    }
}
```

---

## 3. Bounce / Complaint webhook

SES → SNS → 서버:
- Bounce: 사용자 email 비활성화 (`user_email_blocked` 별도 테이블).
- Complaint: 사용자 marketing 알림 모두 OFF + admin alert.

자세히: [[../security/webhook-signature]].

---

## 4. 함정

- From email verify 안 함 → 발송 fail.
- SES sandbox → 미인증 email 발송 X (production 신청 필요).
- DKIM/SPF 없이 발송 → spam 분류.
- bounce 무시 → SES account suspend 위험.

---

## 5. 관련

- [[implementation|↑ hub]]
- [[../prerequisites#4]]
- [[template-impl]]
