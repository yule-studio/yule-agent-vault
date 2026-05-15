---
title: "platform-engineering — 함정 모음"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T08:36:00+09:00
tags: [devops, platform-engineering, pitfalls]
---

# platform-engineering — 함정 모음

**[[platform-engineering|↑ platform-engineering]]**

---

## 1. IDP 일반

1. **top-down 강제** — 매력 없으면 우회.
2. **platform 만 만들고 사용자 없음** — 광고 / training 필요.
3. **DevOps team 이름만 platform team** — mindset 다름.
4. **너무 많은 abstraction** — leaky abstraction.
5. **단일 거대 platform** — 모듈화 부재.
6. **product mindset 부재** — "사용자 알아서".

---

## 2. Backstage

1. **catalog 안 채움** — 빈 UI. 자동 discovery 필수.
2. **plugin 너무 많음** — 무거움 / 충돌.
3. **app-config.yaml 비대** — 환경별 분리.
4. **upgrade 어려움** — major breaking.
5. **자체 plugin 작성** — TypeScript + React 학습.
6. **권한 정책 없음** — 누구나 catalog 삭제.

---

## 3. Crossplane

1. **간단 case 도 Crossplane** — over-engineering.
2. **Composition 추상화 너무 일찍**.
3. **state migration (Terraform → Crossplane) 어려움**.
4. **delete propagation** — claim 삭제 시 진짜 cloud 도 삭제 (★).
5. **secret 평문** — ESO / SealedSecret 같이.
6. **provider version drift**.

---

## 4. scaffolding

1. **template 만 만들고 안 갱신** — stale.
2. **너무 많은 variant** — fragmentation.
3. **너무 적은 variant** — 다양 case 못 다룸.
4. **upgrade path 없음** — 옛 service stale.
5. **변경 후 강제** — 반발. opt-in.
6. **template 자체 test 없음** — render 깨짐.

---

## 5. self-service

1. **광범위 self-service** — 사고. policy 필요.
2. **guardrail 너무 빡빡** — 우회.
3. **UI / CLI / GitOps 정책 불일치**.
4. **failure mode 없음** — error 시 갈피 X.
5. **audit log 없음**.
6. **non-technical 무시**.

---

## 6. DX

1. **DORA 만 — overoptimize** — quality 무시.
2. **activity metric 만** — outcome 무시.
3. **개발자 surveillance** — 신뢰 무너짐.
4. **개선 시도 → metric 조작**.
5. **단발성 survey**.
6. **tool fragmentation** — 5개 dashboard.

---

## 7. 운영 일반

1. **platform team 이 너무 작음** — capacity 부족.
2. **operating manual 없음** — 문서 부재.
3. **roadmap 없음** — 사용자에게 무엇 기대 X.
4. **외부 contribute 못 함** — bottleneck.
5. **backward compatibility 안 함** — 사용자 break.

---

## 8. 관련

- [[platform-engineering|↑ platform-engineering]]
