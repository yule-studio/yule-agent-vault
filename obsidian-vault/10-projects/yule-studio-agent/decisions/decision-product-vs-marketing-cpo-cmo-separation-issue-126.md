---
title: "product-agent vs marketing-agent — CPO/CMO 부서 분리 결정"
kind: decision
session_id: corporate-structure-f15
project: yule-studio-agent
created_at: 2026-05-11T00:00:00+09:00
agent: engineering-agent/tech-lead
status: decided
related:
  - ../../../../policies/runtime/agents/corporate-org-chart.md
  - ../../../../policies/runtime/agents/department-boundary-policy.md
tags:
  - decision
  - corporate-structure
  - department-boundary
  - cpo
  - cmo
---

# 목표

F15 #126 회사 구조 작업에서 product 와 marketing 을 하나의 "growth-ish 에이전트"
로 합쳐도 되는지 / 분리를 유지해야 하는지를 한 번에 정리한다. 후일 "어차피 둘
다 funnel/실험 봄" 이라는 압력이 들어왔을 때 다시 펼쳐 볼 노트.

# 결정

| ID | 항목 | 결정 |
| --- | --- | --- |
| D-DEPT-1 | product-agent / marketing-agent 분리 유지 | 의도적 분리 |
| D-DEPT-2 | 두 부서의 C-level 매핑 | product = CPO, marketing = CMO |
| D-DEPT-3 | A/B test 책임 주체 | product/growth-analyst = 제품 funnel 진단, marketing/growth-marketer = 캠페인 attribution. 활성화는 cross-review |
| D-DEPT-4 | 리서치 책임 주체 | product/user-researcher = 제품 사용자, marketing/brand-manager = 시장/브랜드 인식 |
| D-DEPT-5 | 콘텐츠 책임 주체 | marketing/content-strategist 단일 (PRD 카피와 외부 게시 카피 혼동 금지) |
| D-DEPT-6 | Discord 채널 라우팅 | `#product-*` vs `#marketing-*` 분리 |
| D-DEPT-7 | 분리/통합 변경 게이트 | `department-boundary-policy.md` 1.3 멈춤 신호 확인 + 운영자 명시 승인 |

# 각 결정의 의미

## D-DEPT-1 / D-DEPT-2 의도적 분리

product 와 marketing 은 표면적으로 funnel / 실험 / 리서치를 공유하는 듯
보이지만 1차 책임이 다르다:

- product = 제품 발견 → 어떤 기능을 / 누구를 위해 / 왜 만드는가
- marketing = 그로스 → 만든 것을 / 누구에게 / 어떻게 알리고 살리는가

이 두 책임을 한 부서에 묶으면 "PRD 카피 ↔ 외부 카피" 경계가 흐려지고
브랜드 가드가 작동하지 않는다. C-level 매핑을 분리해 두면 자연스럽게
승인 경로 / 리스크 분류 / 채널 라우팅이 분리된다.

## D-DEPT-3 A/B test 는 cross-review

- product 쪽 실험: 신규 기능 활성 여부, 기능 노출 비율
- marketing 쪽 실험: 채널/메시지/CTA/UTM 변형

활성화 권한이 같지 않다. product/growth-analyst 가 funnel 진단 결과를
넘기면 marketing/growth-marketer 가 캠페인 가설로 변환한다. 둘 다
운영자 명시 승인 후에만 production traffic 영향.

## D-DEPT-4 / D-DEPT-5 리서치 / 콘텐츠 분리

- user-researcher 의 1차 소스 = 인터뷰 / usability test / journey map
- brand-manager 의 1차 소스 = 시장 인식 / PR 응대 / 파트너십 브랜드 리뷰

비슷한 "리서치" 라벨이지만 산출물의 다음 행선지가 다르다. 인터뷰 raw
는 PRD 근거로 가고, 시장 인식은 캠페인 톤/메시지 정합성 가드로 간다.

콘텐츠 작성 역할도 marketing/content-strategist 단일로 둔다. PRD/스펙
문서 작성은 product-manager 의 책임이지 콘텐츠 작성과는 다르다.

## D-DEPT-6 Discord 채널 라우팅

`#product-*` 와 `#marketing-*` 채널 prefix 를 분리. 운영자가 "어느
부서 일인지" 한 눈에 보이게. 부서간 협업 메시지는 양쪽 게이트웨이
모두 거친다 (직접 채널 침범 금지).

## D-DEPT-7 분리/통합 변경 게이트

본 결정을 뒤집고 싶을 때는 `department-boundary-policy.md` 1.3 의
멈춤 신호 3 가지를 다시 확인. 그래도 통합이 옳다고 판단되면 운영자
명시 승인 + 본 노트의 status 를 `superseded` 로 갱신한 새 decision
노트 작성.

# 비결정 (의도적)

- CRO (sales-cs-agent) 와 marketing-agent 의 경계: marketing 이
  리드 생성, CRO 가 리드 전환/유지. 본 노트의 직접 범위가 아니므로
  CRO 에이전트 도입 시 별도 decision 노트로 정리.
- engineering-agent/product-designer 와 product-agent/user-researcher
  의 경계: 디자이너는 화면/플로우 결정, 리서처는 사용자 동기 발굴.
  현재 engineering-agent 의 product-designer 는 잔존하되, 제품 발견
  단계는 product-agent/user-researcher 가 1차 책임. 후일 product-agent
  안으로 디자이너를 옮길지는 별도 결정.

# 변경 영향

- `agents/product-agent/{product-manager,user-researcher,growth-analyst}/`
  3 역할 정착 (F15 commit 2)
- `agents/marketing-agent/{growth-marketer,content-strategist,seo-specialist,brand-manager}/`
  4 역할 정착 (F15 commit 3)
- `policies/runtime/agents/department-boundary-policy.md` 본 결정의
  운영 정책 표면 (F15 commit 3.5)
- 후속 governance 테스트 (F15 commit 7) 에서 본 노트 + 정책 + manifest
  3 자가 일관되는지 검증
