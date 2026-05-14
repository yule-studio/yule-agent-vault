---
title: "Device 관리 — token 등록 / 만료 / 다중 디바이스"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T21:25:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - notification
  - design-decisions
  - device
---

# Device 관리 — token 등록 / 만료 / 다중 디바이스 ★

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[design-decisions|↑ hub]]**

> FCM token / APNs deviceToken / WebPush subscription = 유효 / 만료 lifecycle 관리.

---

## 1. 본 vault 정책

| 시점 | 동작 |
| --- | --- |
| App 첫 실행 | FE 가 FCM token 받아 `POST /me/devices` |
| App 재실행 | FE 가 token 비교 → 변경 시 PATCH |
| logout | DELETE 또는 `active=false` |
| 90일 미사용 | 자동 cleanup |
| FCM `UNREGISTERED` 응답 | 즉시 DELETE |
| APNs `BadDeviceToken` 응답 | 즉시 DELETE |
| 사용자 device 양도 (다른 user 가 같은 token) | UPSERT (이전 user 의 active=false) |

---

## 2. 왜 / 안 하면 / 대안 / 트레이드오프

### 2.1 왜 device 별 관리

**왜 필요**
- 사용자 1 명 → 폰 + 태블릿 + 웹 = 다중 device.
- 각 device 가 별도 token — 모두에게 발송.
- iOS / Android / Web 다른 채널.

**안 하면 어떤 문제**
- 사용자 1개 token 만 → 다른 device 의 알림 누락.
- 옛 token 보관 → invalid token 에 발송 → FCM 비용 / rate limit 소진.

### 2.2 왜 자동 cleanup (UNREGISTERED 응답 시)

**왜 필요**
- app 삭제 / OS 재설치 / 사용자 알림 권한 거부 → token 무효.
- 무효 token 에 계속 발송 → FCM 비용 + 응답 대기 시간 ↑.
- → 즉시 제거.

**안 하면 어떤 문제**
- token DB 무한 누적.
- 사용자 1 명 의 device 10+ (옛 device 모두).
- 발송 시 token check round-trip 비용.

**대안**
- 7일 retry 후 제거 — 너무 늦음.
- 즉시 제거 ★ — 표준.
- DEACTIVATED 상태로 (영구 보관) — 분석용.

### 2.3 왜 90일 미사용 cleanup

- 사용자가 app 삭제 후 다시 install — token 다름.
- 사용자가 OS 재부팅 — token 변경 가능.
- 90일 = balance (사용자 가끔 사용 user 보호 + 오래된 token 청소).

### 2.4 왜 UPSERT (token unique)

```sql
ON CONFLICT (token) DO UPDATE
SET user_id = EXCLUDED.user_id, ...
```

**왜 필요**
- 사용자 A 가 핸드폰 사용 후 친구 B 에게 양도 → B 가 같은 device 에서 login.
- 같은 token 이 user_id 변경 — UPSERT 가 자연스럽게 처리.

---

## 3. Schema

```sql
CREATE TABLE user_devices (
    id              CHAR(26) PRIMARY KEY,
    user_id         CHAR(26) NOT NULL,
    platform        VARCHAR(20) NOT NULL,        -- ANDROID / IOS / WEB / DESKTOP
    channel_type    VARCHAR(20) NOT NULL,        -- FCM / APNS / WEBPUSH
    token           TEXT NOT NULL,                -- FCM token / APNs deviceToken / WebPush endpoint
    token_encrypted BYTEA,                        -- 옵션 — KMS encryption
    device_id       VARCHAR(100),                  -- FE 가 부여 (UUID)
    app_version     VARCHAR(50),
    os_version      VARCHAR(50),
    locale          VARCHAR(10) NOT NULL DEFAULT 'ko-KR',
    timezone        VARCHAR(50) NOT NULL DEFAULT 'Asia/Seoul',
    active          BOOLEAN NOT NULL DEFAULT TRUE,
    last_used_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT chk_device_platform CHECK (platform IN ('ANDROID','IOS','WEB','DESKTOP')),
    CONSTRAINT chk_device_channel CHECK (channel_type IN ('FCM','APNS','WEBPUSH'))
);

CREATE UNIQUE INDEX ux_user_devices_token ON user_devices (token);
CREATE INDEX ix_user_devices_user ON user_devices (user_id, active);
CREATE INDEX ix_user_devices_cleanup ON user_devices (last_used_at)
    WHERE active = TRUE;
```

자세히: [[../database/user-devices-table]].

---

## 4. 등록 API

```http
POST /api/v1/me/devices
{
  "platform": "ANDROID",
  "channelType": "FCM",
  "token": "ekqJ:APA91...",
  "deviceId": "uuid-from-fe",
  "appVersion": "1.2.3",
  "osVersion": "Android 13",
  "locale": "ko-KR",
  "timezone": "Asia/Seoul"
}

200 OK
```

```http
DELETE /api/v1/me/devices/{deviceId}     # logout
PATCH  /api/v1/me/devices/{deviceId}     # locale / timezone 변경
GET    /api/v1/me/devices                 # 사용자 본인 device 목록
```

---

## 5. FCM/APNs 응답 처리 (자동 청소)

```java
@Component
public class FcmChannel implements NotificationChannel {

    public DeliveryResult send(NotificationPayload payload, UserDevice device) {
        try {
            var response = firebaseMessaging.send(buildMessage(payload, device.token()));
            return DeliveryResult.success(response);

        } catch (FirebaseMessagingException e) {
            if (e.getMessagingErrorCode() == MessagingErrorCode.UNREGISTERED) {
                // 즉시 DELETE
                deviceService.markInvalid(device.id(), "UNREGISTERED");
                return DeliveryResult.permanent("token-unregistered");
            }
            if (e.getMessagingErrorCode() == MessagingErrorCode.INVALID_ARGUMENT) {
                deviceService.markInvalid(device.id(), "INVALID_ARGUMENT");
                return DeliveryResult.permanent("invalid-token");
            }
            if (e.getMessagingErrorCode() == MessagingErrorCode.QUOTA_EXCEEDED) {
                return DeliveryResult.transient("quota-exceeded");
            }
            return DeliveryResult.transient(e.getMessage());
        }
    }
}
```

자세히: [[../implementation/fcm-channel-impl]] · [[../implementation/apns-channel-impl]].

---

## 6. 미사용 cleanup cron

```java
@Scheduled(cron = "0 0 4 * * *")    // 매일 04:00
public void cleanupInactiveDevices() {
    var threshold = clock.now().minus(Duration.ofDays(90));
    int deactivated = repo.deactivateInactiveBefore(threshold);
    log.info("deactivated {} inactive devices", deactivated);
}
```

→ deactivate 만 (DELETE X) — 분석 / audit 보존.

---

## 7. 함정

### 함정 1 — token 평문 저장
DB dump 유출 시 attacker 가 사용자 push spam.
**Why**: token 은 push 발송 의 비밀.
**How to apply**: KMS encryption or BYTEA + pgcrypto.

### 함정 2 — token UNIQUE 없음
같은 device 가 multi user (양도 / 도용) 시 multi row.
**Why**: token 은 device 의 고유 식별자.
**How to apply**: UNIQUE + UPSERT.

### 함정 3 — UNREGISTERED 무시
무효 token 에 영구 발송.
**Why**: FCM cost + 시간.
**How to apply**: 즉시 DELETE / deactivate.

### 함정 4 — 90일 cleanup 없음
DB 무한 누적.
**Why**: 옛 device row 가 query 느리게.
**How to apply**: cron.

### 함정 5 — device_id (FE 부여) 없음
같은 token 갱신인지 새 device 인지 X.
**Why**: token 은 갱신 가능 (FCM rotate) — device 식별은 별도.
**How to apply**: FE 가 UUID 발급 + persist.

### 함정 6 — locale / timezone 저장 X
i18n template / quiet hours 적용 X.
**Why**: 사용자별 알림 시간 / 언어.
**How to apply**: device row 에 저장.

### 함정 7 — logout 시 cleanup 안 함
사용자 logout 후 옛 token 으로 발송 (다른 user 가 device 사용 시 보안 누설).
**Why**: token 은 device 종속 — logout 시 user 종속 끊김.
**How to apply**: DELETE / active=false.

---

## 8. 다른 컨텍스트

### 8.1 카카오톡 / 라인
- multi-device sync (PC + 폰) — token 외 자체 device session 관리.

### 8.2 enterprise (Slack desktop)
- desktop client 의 별도 token.

### 8.3 게임
- guest account → 1 device only.

---

## 9. 관련

- [[design-decisions|↑ hub]]
- [[../database/user-devices-table]]
- [[../implementation/device-management-impl]]
- [[../security/device-token-rotation]]
- [[../enums/device-platform]]
- [[../enums/channel-type]]
