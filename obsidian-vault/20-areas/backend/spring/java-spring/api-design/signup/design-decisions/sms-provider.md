---
title: "SMS / 알림톡 / 본인인증 도구"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T04:30:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - design-decisions
  - sms
  - alimtalk
  - identity-verification
---

# SMS / 알림톡 / 본인인증 도구

**[[design-decisions|↑ design-decisions hub]]**

> "휴대폰으로 어떻게 메시지 보내나" + "본인 인증 어떻게 하나" — 한국 SaaS 의 핵심 결정.

---

## 1. 본 vault 결정

- **SMS**: NCP SENS (1순위) + AlimTalk (2순위 조합).
- **알림톡**: NCP SENS 의 AlimTalk 또는 카카오 비즈 직접.
- **본인인증** (금융 / 의료만): PASS / NICE.

---

## 2. SMS 도구 비교 — 4구조

### 2.1 NCP SENS (네이버 클라우드)

**왜 1순위**
- 한국 1위 점유율 — 운영 안정성 검증.
- API 단순 (REST + signature 인증).
- 가격 합리 — SMS 9-13원 / LMS 30원.
- AlimTalk / RCS 통합 (한 API 로 다 발송).

**한계**
- 외국 발송 X (글로벌 SaaS 부적합).

**왜 적합**
- 한국 B2C SaaS = NCP SENS 가 사실상 표준.

---

### 2.2 AlimTalk (카카오 비즈)

**왜 같이 사용**
- SMS 보다 저렴 (7원 vs 9원, LMS 대비 30원 vs 7원).
- 카카오 사용자 (한국 95%) 도달 - 친숙한 UI.
- 신뢰감 (카카오 브랜드).

**한계**
- 사전 등록된 템플릿만 — 검수 1주.
- 카카오톡 미사용자 (5%) 에겐 fallback SMS 필요.
- 마케팅 발송 제한 (카카오 정책).

**왜 SMS 와 같이**
- AlimTalk 시도 → 실패 (미사용자) → SMS fallback.
- 운영 비용 최적 + 도달율 ↑.

---

### 2.3 Twilio

**왜 적합한 케이스**
- 글로벌 — 200+ 국가.
- 풍부한 기능 (Voice, Video, WhatsApp).

**왜 안 됨 (한국 한정)**
- 비싸다 (한국 SMS ~50원, NCP 의 5배).
- 한국 통신사와 직접 계약 X → 도달율 / 속도 ↓.

**언제 적합**
- 글로벌 SaaS — 한국 user 외 다수.
- 음성 / WhatsApp 도 필요.

---

### 2.4 알리고 / Daou / NHN Cloud SMS

**왜 대안**
- NCP SENS 의 alternative — 가격 / 기능 비슷.
- 일부 기업 정책 (네이버 외 사용).

**왜 안 선택 (본 vault)**
- NCP SENS 가 인프라 통합 + 문서 / 사례 풍부.

---

## 3. SMS vs LMS vs MMS

| 종류 | 길이 | 비용 (NCP) | 사용 |
| --- | --- | --- | --- |
| SMS | 90 byte (한글 45자) | 9-13원 | 인증 코드 / 짧은 알림 |
| LMS | 2000 byte (한글 1000자) | 30원 | 긴 안내 / 약관 |
| MMS | 2000 byte + 이미지 | 100원+ | 광고 / 영수증 이미지 |

**본 vault 인증 코드 = SMS**
- 6-digit + 짧은 안내 = 90 byte 충분.
- 비용 최저.

**왜 LMS 가 비싼가**
- 통신사 인프라 비용 — 한 번에 4-5 SMS 분량 전송.

---

## 4. 본인인증 비교 (PASS / NICE / KCB / SCI)

### 4.1 PASS (통신 3사 통합)

**왜 사용**
- 통신 3사 (SKT / KT / LGU+) 통합 API.
- 휴대폰 본인인증 표준 — 가장 친숙한 UX.
- 비용 합리 (~150원).

**언제 필수**
- 금융 — 계좌 / 송금 / 결제.
- 의료 — 처방 / 진료.
- 청소년 보호 — 게임 / 성인 사이트.

**한계**
- 외국인 X (한국 통신사 가입자만).
- 가족 명의 휴대폰 사용자 (자녀 / 부모) — 명의 불일치.

---

### 4.2 NICE 평가정보

**왜 사용**
- 가장 범용 — 휴대폰 / 신용카드 / 공동인증서 다 지원.
- 한국 본인인증 시장 1위.

**언제 적합**
- 금융 / 보험 — NICE 가 표준.
- 다양한 인증 수단 제공 필요.

**한계**
- 비싸다 (~200원).

---

### 4.3 KCB / SCI

**왜 대안**
- NICE 와 비슷. 가격 경쟁.
- SCI 는 전자서명 (계약 / 동의서) 가능.

---

## 5. 흐름 — SMS 인증

```
[클라]
  POST /auth/verify/phone/request   { phone: "010-0000-0000" }
   ↓
[서버]
  - 정규화 (010-0000-0000 → 01000000000)
  - 6자리 코드 생성 (random 100000-999999)
  - Redis SET (key: phone:verify:01000000000, value: { codeHash, attempts: 0 }, TTL: 3m)
  - NCP SENS API 호출 (SMS 발송)
   ↓
[사용자 휴대폰]
  "[Yule] 인증번호 123456 (3분 유효)"
   ↓
[클라]
  POST /auth/verify/phone/confirm   { phone: "010-0000-0000", code: "123456" }
   ↓
[서버]
  - Redis GET → attempts++ → match
  - 일치 시: phoneAuthToken 발급 (TTL 10m, signup 진행 동안)
  - 불일치: attempts 5 → lock 30분
   ↓
[가입 시]
  POST /auth/signup   { email, password, phoneAuthToken, ... }
  - phoneAuthToken 검증 → users INSERT (phoneVerifiedAt=now)
```

자세히: [[../phone-verification-impl]].

---

## 6. 흐름 — 본인인증 (PASS)

```
[클라]
  사용자 클릭 → "본인 인증 (PASS)"
   ↓
[서버]
  POST /auth/identity-verification/init
  → PASS 의 popup URL + transaction_id 반환
   ↓
[클라]
  PASS popup 열림 (PASS 앱 / SMS 인증)
   ↓
[PASS → 서버 callback]
  POST /webhook/pass/result
  → transaction_id + 실명 / 휴대폰 / 생년월일 / 성별 + 서명
   ↓
[서버]
  - 서명 검증
  - DB 의 transaction_id status = VERIFIED
  - 본인 정보 임시 저장
   ↓
[클라]
  GET /auth/identity-verification/result?txId=...
  → 가입 진행 (본인 정보 prefill)
```

자세히 - 본인인증 레시피는 별도 (본 vault 외).

---

## 7. NCP SENS API 호출

```java
@Component
public class NcpSensSmsClient {
    private final WebClient webClient = WebClient.create("https://sens.apigw.ntruss.com");

    public void sendSms(String to, String message) {
        String timestamp = String.valueOf(Instant.now().toEpochMilli());
        String signature = makeSignature(timestamp);    // HMAC-SHA256

        webClient.post()
            .uri("/sms/v2/services/{serviceId}/messages", serviceId)
            .header("x-ncp-apigw-timestamp", timestamp)
            .header("x-ncp-iam-access-key", accessKey)
            .header("x-ncp-apigw-signature-v2", signature)
            .contentType(MediaType.APPLICATION_JSON)
            .bodyValue(Map.of(
                "type", "SMS",
                "from", "01012345678",
                "content", message,
                "messages", List.of(Map.of("to", to))
            ))
            .retrieve()
            .toBodilessEntity()
            .block();
    }
}
```

**왜 WebClient (RestTemplate 아님)**
- Spring 6 권장 (RestTemplate 은 maintenance only).
- async / reactive 친화 — 대량 발송 시 throughput ↑.

자세히: [[../phone-verification-impl#3 NCP SENS]].

---

## 8. 비용 비교

| 도구 | 1k 인증 | 10k 인증 | 100k 인증 |
| --- | --- | --- | --- |
| NCP SENS (SMS) | ~13,000원 | ~130k | ~1.3M |
| AlimTalk | ~10,000원 | ~100k | ~1M |
| PASS (본인인증) | ~150k | ~1.5M | ~15M |
| NICE (본인인증) | ~200k | ~2M | ~20M |
| Twilio (글로벌 SMS) | ~50k | ~500k | ~5M |

→ SMS 와 본인인증의 비용 차이 ≈ 10배.

---

## 9. 함정 모음

### 함정 1 — phone 정규화 안 함
`010-1234-5678` / `01012345678` / `+821012345678` 모두 다른 hash → 중복 가입 가능.
→ 모두 `01012345678` 로 정규화 후 처리.

### 함정 2 — 인증 코드 너무 짧음 (4자리)
1만 경우 → brute force 가능.
→ 6자리 + 5회 lock + 3분 TTL.

### 함정 3 — 인증 코드 만료 너무 김 (10분+)
도난 시 사용 가능 시간 ↑.
→ 3분.

### 함정 4 — Rate limit 없음
악의적 발송 폭주 → SMS 비용 폭증.
→ 1시간 5회 + 60초 cooldown.

### 함정 5 — phoneAuthToken 재사용 가능
같은 토큰으로 여러 가입.
→ 사용 시 status=USED 전이.

### 함정 6 — AlimTalk 만 사용 (SMS fallback 없음)
카카오 미사용자 (5%) 가입 못함.
→ AlimTalk → SMS fallback.

### 함정 7 — 한국 외 발송 (Twilio 없음)
글로벌 사용자 인증 불가.
→ Twilio fallback 또는 글로벌 X 정책.

### 함정 8 — 일반 SaaS 에 PASS 강제
비용 ↑ + 사용자 friction.
→ SMS 만. PASS 는 금융 / 의료만.

### 함정 9 — 본인인증 정보 평문 저장
주민번호 / 실명 평문 = PIPC 위반.
→ 본인인증은 통과 결과 (boolean, verified_at) 만 저장. 실명 / 생년월일은 필요 최소만.

### 함정 10 — SMS 발송 트랜잭션 안에서
SMS API 5초 → DB 락 5초 → 부담.
→ outbox 패턴 ([[../database/email-outbox-table]] 와 같은 패턴, sms_outbox).

### 함정 11 — 발송 평판 (반송) 관리 안 함
번호 폐쇄 / 잘못 입력에 무한 발송 → 비용 + 평판 손상.
→ webhook 처리.

### 함정 12 — 한국 휴대폰 패턴 검증 안 함
`02-1234-5678` (일반 전화) → SMS 발송 실패.
→ 휴대폰 패턴 정규식 (010|011|016|017|018|019).

---

## 10. 다른 컨텍스트

### 10.1 글로벌 SaaS

```yaml
sms-provider:
  primary: twilio
  fallback: regional (예: NCP for 한국 user)
  reason: 200+ 국가
```

### 10.2 금융 / 의료

```yaml
sms-provider:
  primary: ncp-sens
identity-verification:
  primary: pass + nice
  reason: 법적 요구
```

### 10.3 청소년 보호 (게임)

```yaml
sms-provider:
  primary: ncp-sens
identity-verification:
  primary: pass
  reason: 게임산업법 (실명 인증 의무)
```

---

## 11. 관련

- [[design-decisions|↑ hub]]
- [[auth-channels]] — 이메일 / 휴대폰 선택
- [[../phone-verification-impl]] — 구현
- [[../database/verification-tokens-table#4 phone_verifications]]
- 외부 — NCP SENS Developer Guide, Kakao AlimTalk Docs
