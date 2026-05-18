---
title: "career — 커리어 관리 영역 Hub"
kind: knowledge
project: career
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-19T01:00:00+09:00
tags: [area, career, hub]
home_hub: areas
related:
  - "[[../areas]]"
  - "[[interview/]]"
  - "[[resume/]]"
  - "[[portfolio/]]"
  - "[[company-research/]]"
  - "[[certificates/]]"
---

# career — 커리어 관리 영역 Hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-19 | engineering-agent/tech-lead | 최초 — 5 영역 통합 hub |

**[[../areas|↑ 20-areas]]**

---

## 1. 영역 정의

본 영역은 소프트웨어 엔지니어 커리어의 **준비 / 탐색 / 검증** 산출물의 인덱스다.

본 영역이 정의하는 것:
- 5 영역의 진입점 (resume / portfolio / interview / company-research / certificates)
- 각 영역의 책임 분리
- 단계별 진입점 (탐색 / 지원 / 면접 / 협상)
- 인접 영역 (computer-science / backend / frontend / devops / ai) 과의 cross-link

본 영역이 정의하지 않는 것:
- 기술 학습 자체 — 각 기술 영역 (cs / backend / frontend / devops / ai)
- 회사 / 사내 인사 정책 — 외부
- 협상 / 연봉 액수 자체 — 외부 (개인적 결정)

---

## 2. 영역 분류

| 디렉토리 | 다루는 주제 |
| --- | --- |
| [[resume/|resume]] | 이력서 / CV — 한국 / 영어 / 포맷 / 항목 |
| [[portfolio/|portfolio]] | 포트폴리오 사이트 / GitHub README / 프로젝트 정리 |
| [[interview/|interview]] | 기술 면접 (coding / system design / behavior) / 단계별 준비 |
| [[company-research/|company-research]] | 회사 조사 (tech stack / 문화 / 보상 / 규모) |
| [[certificates/|certificates]] | 자격증 (CKA / AWS SAA / OSCP 등) — 가치 평가 + 학습 가이드 |

---

## 3. 단계별 진입점

| 단계 | 진입점 |
| --- | --- |
| 1. 자기 분석 | [[resume/|resume]] (현재 경험 정리부터) |
| 2. 시장 조사 | [[company-research/|company-research]] |
| 3. 산출물 | [[portfolio/|portfolio]] + [[../computer-science/computer-science|↗ CS 영역]] (학습 정리) |
| 4. 면접 준비 | [[interview/|interview]] + [[../computer-science/algorithm/algorithm]] + [[../computer-science/distributed-systems/distributed-systems]] |
| 5. 검증 | [[certificates/|certificates]] (선택) |
| 6. 지원 / 협상 | (외부) |

---

## 4. 기술 영역과의 연결

| 면접 영역 | 본 vault 의 학습 진입점 |
| --- | --- |
| 자료 구조 / 알고리즘 | [[../computer-science/data-structure/data-structure]] + [[../computer-science/algorithm/algorithm]] |
| 시스템 디자인 | [[../computer-science/distributed-systems/distributed-systems]] + [[../../40-patterns/architecture-patterns/architecture-patterns]] |
| 객체지향 / 설계 | [[../computer-science/object-oriented-programming/object-oriented-programming]] + [[../computer-science/design-patterns/design-patterns]] |
| 네트워크 | [[../computer-science/network/network]] |
| OS / 동시성 | [[../computer-science/operating-system/operating-system]] |
| DB / SQL | [[../computer-science/database-theory/database-theory]] + [[../database/database]] |
| 백엔드 / 프레임워크 | [[../backend/backend]] |
| 프론트엔드 | [[../frontend/frontend]] |
| 인프라 / DevOps | [[../devops/devops]] |
| AI / LLM | [[../ai/ai]] + [[../ai-agent/ai-agent]] |
| 보안 | [[../computer-science/security-theory/security-theory]] + [[../security/security]] |

---

## 5. 면접 종류 별 준비 항목

| 면접 종류 | 준비 항목 |
| --- | --- |
| **coding interview** | [[../computer-science/algorithm/algorithm]] (Leetcode / 이코테 / Cracking) + 5+ 언어 중 1 선택 |
| **system design** | [[../computer-science/distributed-systems/distributed-systems]] + scale-up 사례 (Twitter / Uber / Netflix) |
| **OOP design** | [[../computer-science/object-oriented-programming/object-oriented-programming]] (SOLID / DDD / 패턴) |
| **behavioral (STAR)** | 경험 5-7 개를 STAR 구조로 사전 정리 — leadership / failure / conflict / 협업 / 성장 |
| **domain-specific** | 본 vault 의 해당 영역 hub |
| **take-home** | scope / 시간 / 기술 명세 / 평가 기준 명확화 |

---

## 6. 자격증의 위치

자격증은 학습 가이드 + 외부 신호로 활용. 기술 영역의 학습 가이드라인이 우선.

| 자격증 | 본 vault 의 영역 매핑 |
| --- | --- |
| CKA (Kubernetes) | [[../devops/kubernetes/kubernetes]] foundational+operational |
| CKAD / CKS | [[../devops/kubernetes/kubernetes]] foundational / [[../devops/security-ops/]] |
| AWS SAA / SAP | [[../devops/cloud-aws/cloud-aws]] |
| GCP ACE / PCA | [[../devops/cloud-gcp/cloud-gcp]] |
| Azure AZ-104 / AZ-305 | [[../devops/cloud-azure/cloud-azure]] |
| OSCP | [[../computer-science/security-theory/security-theory]] |
| LFCS / RHCE | [[../devops/linux/linux]] |
| OCJP | [[../backend/spring/java-spring/]] |
| 정처기 (한국) | [[../computer-science/computer-science]] 광범위 |

상세: [[certificates/|certificates]].

---

## 7. 본 영역의 유지보수

| 트리거 | 갱신 항목 |
| --- | --- |
| 새 면접 종류 / 평가 도구 등장 | §5 |
| 새 자격증 / 표준 | §6 |
| 기술 영역 hub 갱신 | §4 cross-link |

---

## 8. 관련

- [[../areas|↑ 20-areas]]
- [[resume/|resume]]
- [[portfolio/|portfolio]]
- [[interview/|interview]]
- [[company-research/|company-research]]
- [[certificates/|certificates]]
- [[../computer-science/computer-science|↗ CS]]
- [[../backend/backend|↗ backend]]
- [[../frontend/frontend|↗ frontend]]
- [[../devops/devops|↗ devops]]
- [[../ai/ai|↗ ai]]
- [[../ai-agent/ai-agent|↗ ai-agent]]
