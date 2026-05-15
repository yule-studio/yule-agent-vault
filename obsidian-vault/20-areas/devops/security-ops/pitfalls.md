---
title: "security-ops — 함정 모음"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T08:20:00+09:00
tags: [devops, security-ops, pitfalls]
---

# security-ops — 함정 모음

**[[security-ops|↑ security-ops]]**

---

## 1. secrets

1. **git 에 .env / secret** — history 영구. 즉시 rotate + filter-repo.
2. **CI log 에 echo** — masked 설정.
3. **environment variable** — ps / docker inspect 노출.
4. **rotation 어려운 코드** — credential 변경 감지 X.
5. **Vault HA 없음** — Vault down = service down.
6. **secret 권한 광범위** — least priv.
7. **audit log 없음** — 누가 봤는지.
8. **암호화 key 자체 도난** — KMS/HSM.

---

## 2. IAM

1. **`*` 모든 action/resource** — least priv 위반.
2. **access key hardcode** — Role 사용.
3. **root credential 일상 사용**.
4. **권한 늘림 → 줄이기 어려움**.
5. **퇴사 시 access key 제거 안 함**.
6. **CloudTrail 안 켬**.
7. **prod/dev 같은 account** — blast radius.
8. **shared admin account** — audit 불가.

---

## 3. zero-trust

1. **VPN + zero-trust 혼동**.
2. **인증 빈도 너무 높음** — 우회 시도.
3. **device trust 부재** — 도난/감염 device.
4. **3rd party 적용 어려움**.
5. **legacy app modern auth 미지원**.
6. **observability 부재**.

---

## 4. vulnerability

1. **scan only, fix 안 함**.
2. **noise 무시**.
3. **false positive 모두 ignore**.
4. **base image 안 갱신**.
5. **build time only, runtime 무관심**.
6. **secrets in repo**.
7. **patching 분기 1회**.

---

## 5. supply chain

1. **Docker Hub rate limit**.
2. **`:latest` tag**.
3. **lockfile 미커밋**.
4. **build 환경 access 광범위**.
5. **signing key 도난** — keyless 권장.
6. **admission controller 없음**.
7. **action `@v3` (tag)** — `@<sha>` 권장.

---

## 6. SIEM

1. **모든 log 다 — noise**.
2. **rule 너무 적음 / 많음** — alert fatigue.
3. **PII 평문 저장** — GDPR.
4. **retention 너무 짧음** — forensic 불가.
5. **SIEM 자체 노출**.
6. **SOC 없음**.

---

## 7. compliance

1. **인증 받고 운영 안 함** — Type II fail.
2. **evidence 수동 수집**.
3. **vendor 검토 안 함**.
4. **breach 72h 못 지킴**.
5. **GDPR 삭제 + backup 미반영**.
6. **encryption 만, key 평문**.
7. **PII 어디 있는지 모름** — data discovery.

---

## 8. threat modeling

1. **design 후 시작** — 비쌈.
2. **너무 자세 → 시간 폭주**.
3. **dev 참여 없이 security 만**.
4. **document 만, 적용 X**.
5. **위협 추적 안 함**.

---

## 9. security incident

1. **즉시 shutdown** — memory dump 손실.
2. **공격자 알게 함** — out-of-band channel.
3. **혼자 결정** — legal/leadership.
4. **breach 72h 어김**.
5. **postmortem 없이 종결**.
6. **cyber insurance 미가입**.

---

## 10. 일반

1. **"cloud = 자동 안전" 오해** — shared responsibility.
2. **encryption only at rest** — in transit 도.
3. **MFA root 만** — 모든 IAM user.
4. **2-person rule 없음** — production 변경.
5. **security 가 product 보다 후순위** — culture 문제.
6. **training 없음** — phishing 가장 흔한 vector.
7. **3rd party 검토 안 함** — supply chain.

---

## 11. 관련

- [[security-ops|↑ security-ops]]
