---
title: "networking-ops — 함정 모음"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T07:59:00+09:00
tags: [devops, networking-ops, pitfalls]
---

# networking-ops — 함정 모음

**[[networking-ops|↑ networking-ops]]**

---

## 1. LB

1. NLB target IP — 같은 AZ health check 안 됨.
2. ALB health check path 가 auth 필요 → 항상 401.
3. cross-zone disable + AZ unbalanced.
4. WAF 안 함 → bot / SQLi 직접.
5. dereg delay 너무 짧음 → in-flight 끊김.
6. NLB client IP preservation 안 함 → 모든 traffic 이 LB IP 로 보임.

---

## 2. API Gateway

1. SPOF — HA / 무중단 deploy 필수.
2. fat gateway — 너무 많은 logic.
3. rate-limit per IP only — 같은 NAT 다른 사용자 영향.
4. cache 잘못 — 사용자별 다른 응답 같은 cache.
5. auth 만 gateway, backend 무방비.
6. version 정책 없음 — /v1 /v2 무한 누적.

---

## 3. Service Mesh

1. mesh 부재로 충분한데 도입 — over-engineering.
2. sidecar overhead — 메모리 50-100MB × pod.
3. mTLS PERMISSIVE → STRICT 마이그 신중.
4. istioctl ↔ data plane version mismatch.
5. VirtualService 우선순위 (첫 매칭).
6. 외부 → mesh = IngressGateway 별 flow.

---

## 4. CDN

1. dynamic API cache → 사용자 데이터 누출 (★).
2. cache key 에 user 누락.
3. TTL 길게 → 변경 안 반영, invalidation 비쌈.
4. vary header 무시.
5. gzip / brotli 미지원 origin.
6. HTTPS origin 아님 → CDN ↔ origin 평문.

---

## 5. DNS

1. TTL 길게 → 변경 후 며칠 wait.
2. 변경 전 TTL 줄이기 안 함.
3. apex CNAME 오용.
4. propagation 24-48h 까지 걸림.
5. CAA 없음 → phishing cert.
6. glue record 없음 (NS 자기 참조).

---

## 6. WAF

1. false positive 차단 — 정상 traffic 영향.
2. WAF 만 의존, app validation 부재.
3. rule update 안 함.
4. paranoia level 4 처음부터.
5. body size limit false positive.
6. log 검토 안 함 → 공격 시도 무시.

---

## 7. NetworkPolicy

1. default allow — deny-all 없음.
2. CNI 미지원 (flannel only) — policy 무시.
3. DNS egress 누락 → service DNS resolve 실패.
4. kube-system 차단 → kubelet 영향.
5. L4 만 — Cilium L7 활용 X.
6. observability 없음 → 어떤 traffic deny 됐는지 모름.

---

## 8. TLS

1. 자동 갱신 hook 누락 — nginx reload 안 함.
2. wildcard 안 쓰고 sub-domain 폭증.
3. HTTP-01 challenge path block.
4. rate limit (Let's Encrypt).
5. intermediate 빠짐.
6. cert ↔ key mismatch.
7. 만료 모니터링 없음.
8. HSTS 잘못 등록 → 1년 잠금.

---

## 9. 일반

1. **multi-layer 디버깅 안 됨** — DNS / CDN / LB / Ingress / Mesh / Pod 어디서 fail?  
   → 각 layer 의 access log + trace_id 전파.
2. **MTU 호환성** — VPN / tunnel 환경에서 1500 → 1400-1450.
3. **TCP TIME_WAIT 누적** — short-lived connection 많을 때.
4. **conntrack overflow** — `nf_conntrack_max`.
5. **GeoIP 부정확** — VPN / proxy / mobile.

---

## 10. 관련

- [[networking-ops|↑ networking-ops]]
