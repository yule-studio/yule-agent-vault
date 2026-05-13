---
title: "BGP 사고 사례 — 인터넷의 취약점"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T10:20:00+09:00
tags:
  - network
  - bgp
  - incidents
  - routing
---

# BGP 사고 사례 — 인터넷의 취약점

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | AS7007 / FB 2021 / Hijacks |

**[[topics|↑ 토픽 hub]]** · **[[../network|↑↑ network hub]]**

---

## 1. 한 줄 정의

BGP — 인터넷의 라우팅 신뢰 기반. **잘못된 announcement 한 번에 인터넷 분할 / 트래픽 hijack**.

---

## 2. BGP 의 신뢰 문제

### 디자인
- 1989 — 신뢰 기반 (당시 인터넷 작음)
- 어느 AS 든 — IP prefix 발표 가능
- 검증 메커니즘 약함

### 약점
- IP prefix hijacking
- Route leak (실수)
- Withdrawal flood (DoS)

자세히 → [[../routing/bgp]]

---

## 3. 사례 1 — AS7007 (1997)

### 경위
- 1997 4월 25일
- Florida 의 작은 ISP MAI Network (AS7007)
- Router 설정 오류 — 인터넷의 거의 모든 prefix 를 자기로 발표
- 큰 사업자 — 정책 검증 없음
- 결과 — 인터넷의 큰 부분이 AS7007 로 라우팅 → 폭주 → 다운

### 영향
- 1-2 시간 — 일부 지역 인터넷 멈춤
- 후속 — Route filtering 시작

---

## 4. 사례 2 — Pakistan Telecom vs YouTube (2008)

### 경위
- 2008 2월 24일
- Pakistan 정부 — YouTube 차단 명령
- Pakistan Telecom (AS17557) — `208.65.153.0/24` 자기 라우팅
- 본래는 PT 내부만 — 잘못 외부 announce
- PCCW (PT의 transit) — 검증 없이 수용
- 전 세계 — YouTube 트래픽이 Pakistan 으로

### 영향
- YouTube — 약 2 시간 다운 (글로벌)
- 후속 — RPKI 논의 가속

---

## 5. 사례 3 — China Telecom (2010)

### 경위
- 2010 4월 8일 — 18 분
- China Telecom (AS23724) — 50,000 prefix announce
- Dell / Apple / Yahoo / Microsoft 일부 — 중국 경유
- 18 분 후 정상화

### 의혹
- 우연 / 실험 / 의도? — 명확치 않음
- 정부 / 사회 영향 가능성

---

## 6. 사례 4 — Amazon Route53 hijack (2018)

### 경위
- 2018 4월 24일
- 한 ISP (AS10297) — Amazon Route 53 prefix announce
- 사용자가 MyEtherWallet.com → 가짜 사이트 redirect
- HTTPS — self-signed cert (사용자 경고)
- 일부 사용자 — 클릭 → ETH 도난 ($150K+)

### 영향
- DNS hijack → 금융 손실
- TLS 검증의 중요성

---

## 7. 사례 5 — Facebook outage (2021)

### 경위
- 2021 10월 4일 (6 시간 17 분)
- 페이스북 — BGP withdraw → 자기 DNS 서버도 unreachable
- 직원 — VPN / 내부 도구 X (DNS 의존)
- 사옥 출입 cards 도 영향 (네트워크 의존)

### 흐름
```
1. Backbone 라우터 명령 — 의도치 않게 모든 BGP 광고 withdrawal
2. Facebook 의 DNS 서버 (자체 운영) 도 분리됨
3. 외부 DNS resolver — Facebook DNS 응답 X → 사용자 facebook.com 못 찾음
4. 직원 — 내부 VPN 도 DNS 의존 → 작업 불가
5. 사옥 카드 시스템 — 네트워크 의존 → 출입 어려움
6. 한 엔지니어가 데이터센터 가서 수동 복구
```

### 교훈
- "Single point of failure" — DNS / VPN / 보안
- 자체 운영의 위험성
- 비상 통신 채널

---

## 8. 사례 6 — Cloudflare 1.1.1.0/24 hijack (다수)

### 경위
- 1.1.1.0/24 (Cloudflare DNS) — 다수 ISP 가 잘못 announce
- 일부 사용자 — Cloudflare 못 가고 다른 곳 redirect

### 빈도
- 자주 발생 (월 1+ 회)
- Cloudflare — 모니터링 / 빠른 대응

---

## 9. 사례 7 — KlaySwap (2022 한국)

### 경위
- 2022 2월
- 한국의 KlaySwap (DEX) — BGP hijack
- Kakao 의 IP 가 영향
- 사용자 → 악성 JS 다운로드 → 자금 손실 (~$2M)

---

## 10. 사례 8 — AWS US-EAST-1 (2017)

### 경위
- 2017 2월 28일
- S3 운영 자료 입력 실수 (typo) — 큰 S3 partition 정지
- 4 시간 일부 사용자 — S3 / 의존 사이트 다운

### 영향
- AWS console 자체 — S3 의존 → 못 보임 (irony)
- 메이저 사이트 (Quora, Slack) 영향

→ BGP 가 아니지만 — 인터넷 인프라 의존의 약점 사례.

---

## 11. RPKI — Resource Public Key Infrastructure

### 정의
- IP prefix 의 소유 — 암호화 서명
- ROA (Route Origin Authorization)

### 흐름
```
RIR (ARIN / RIPE) → 발급
AS owner → ROA: "이 prefix 는 우리 AS"
Router → ROA 검증 → invalid 무시
```

### 도입
- 2010s 시작
- 2020+ — 메이저 ISP 70%+ 채택
- Cloudflare / Amazon / Google — 강제

### 한계
- 모든 ISP 채택 필요
- Route leak (정책) — RPKI 만으로 부족

자세히 → [[../routing/bgp]]

---

## 12. BGPsec — 더 강한 (RFC 8205)

### 정의
- AS path 도 서명
- Hop-by-hop 검증

### 한계
- 매우 무거움 (CPU)
- 도입 매우 느림

---

## 13. 방어 / 모니터링

### Tools
- **BGPmon** (Cisco) — alert
- **Cloudflare Radar** — 인터넷 측정
- **RIPE NCC routing** beacons
- **bgp.he.net** — Hurricane Electric

### 운영
- ROA 발급 / 등록
- 정기 모니터링
- Route filters

---

## 14. 인터넷 인프라의 SPOF

### Cloudflare 사고 (다수)
- 2019 — 정규식 폭주 → 30 분 다운
- 2020 — DNS 코드 버그 → 일부 region

### AWS US-EAST-1
- 가장 큰 region — 자주 사고
- "어떤 회사 — us-east-1 + 한 region 만" 위험

### DigitalOcean / GitHub / Slack
- 각자 다양한 사례

### 교훈
- 다중 region / 다중 cloud
- Self-healing
- Chaos engineering

---

## 15. 함정

### 함정 1 — "인터넷은 항상 작동"
실제 — 자주 사고. 모니터링 필수.

### 함정 2 — DNS 의존
DNS 죽으면 — 모든 사이트 안 보임. 다중 NS / GTLD.

### 함정 3 — 한 cloud 의존
us-east-1 사고 — 큰 사이트 도미노. 다중 region.

### 함정 4 — BGP 의 빠른 대응
사고 시 — 분 단위 대응. 자동화 / runbook.

### 함정 5 — RPKI 의 모든 게 아님
Route leak (사고) — RPKI 만으론 못 잡음. AS path 검증도.

---

## 16. 학습 자료

- "Routing in the Internet" (Huitema)
- RIPE NCC RPKI docs
- Cloudflare 블로그 (사고 분석)
- BGPmon / BGP Stream
- Facebook 2021 outage post-mortem

---

## 17. 관련

- [[topics]] — Hub
- [[../routing/bgp]] — BGP 동작
- [[../dns/dns]] — Facebook DNS 사고
- [[bufferbloat]] (TBD)
