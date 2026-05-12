---
title: "links — 에이전트 학습 reference 카탈로그 (부서 hierarchy)"
kind: knowledge
project: agent-references
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-12T18:30:00+09:00
tags:
  - reference
  - links
  - agent-learning
  - catalog
---

# links — 에이전트 학습 Reference (부서 hierarchy)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.2.0.0 | 2026-05-12 | engineering-agent/tech-lead | 부서 hierarchy 도입 (engineering/product/marketing/general) |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 — flat 단일 폴더 |

> 각 에이전트 역할이 학습 / 자료 조사 / 패턴 참고할 때 펼쳐볼 외부
> reference. 부서 (CTO / CPO / CMO) 별로 묶고, 부서 안에서 역할별로
> 세분.

**[[../inbox|↑ 00-inbox/]]**

## 부서별 인덱스

| 부서 (C-level) | 진입점 | 산하 역할 |
| --- | --- | --- |
| Engineering (CTO) | [[engineering/engineering]] | tech-lead / backend / frontend / ai / devops / qa / product-designer |
| Product (CPO) | [[product/product]] | product-manager / user-researcher / growth-analyst |
| Marketing (CMO) | [[marketing/marketing]] | growth-marketer / content-strategist / seo-specialist / brand-manager |
| General (전부서) | [[general/general]] | 시스템 설계 / 책 / 커뮤니티 / 공통 도구 |

## 운영 규칙

1. **검증된 출처만** — 공식 docs > 회사 엔지니어링 블로그 > 개인 블로그.
   광고성 / 죽은 링크는 등록 금지.
2. **링크 1 줄 양식** — `[제목](url) — 한 줄 설명 (왜 좋은지 / 누가 참고)`
3. **부서 / 역할 매칭** — 한 링크가 두 역할에 걸치면 두 카탈로그 모두에
   기록 OR 더 자주 참고할 한 곳에 + cross-link.
4. **30-resources/ 와의 관계** — 본 카탈로그는 **링크 모음**. 책 1 권의
   깊은 요약은 `30-resources/books/` 로 별도 노트.
5. **카탈로그 갱신** = 본 노트 + 해당 부서 인덱스 + 해당 역할 파일 3
   곳의 "현재 링크" 수 갱신.

## 자동화 에이전트 진입

`live-research-provider` 플러그인의 host 화이트리스트 (`YULE_RESEARCH_LIVE_ALLOW_HOSTS`)
가 본 카탈로그와 동기화될 예정. 새 host 추가 시 본 카탈로그 + .env.local
양쪽 갱신.

## 관련

- [[../inbox|↑ 00-inbox 인덱스]]
- [[../../30-resources/resources|30-resources/ (깊은 요약)]]
