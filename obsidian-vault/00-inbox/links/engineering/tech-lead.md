---
title: "tech-lead — 학습 reference 카탈로그"
kind: knowledge
project: agent-references
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-12T18:30:00+09:00
tags:
  - reference
  - links
  - tech-lead
  - system-design
  - leadership
---

# tech-lead — 학습 Reference

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 15 링크 |

> tech-lead 의 책임 (작업 분해 / 의존 순서 / 합의 조율 / 외부 회신 /
> 시스템 설계) 에 맞춘 외부 reference. 한 사이클 시작 전 한 번 펼쳐
> 보고 패턴 / 안티패턴을 점검.

**[[links|↑ links 카탈로그]]**

## 시스템 설계

- [System Design Primer (donnemartin)](https://github.com/donnemartin/system-design-primer) — 면접·실무 시스템 설계의 표준 reference (200k+ star).
- [Designing Data-Intensive Applications 노트 모음](https://github.com/ept/ddia-references) — 책의 각 장 reference 모음. 시스템 설계 결정 시 검증된 출처.
- [The C4 Model](https://c4model.com/) — Context/Container/Component/Code 다이어그램 표준. 부서간 설계 합의 시 공통 언어.
- [Microservices.io patterns (Chris Richardson)](https://microservices.io/patterns/) — Saga / Outbox / API Gateway 등 패턴 카탈로그.
- [Awesome Software Architecture](https://github.com/mehdihadeli/awesome-software-architecture) — 아키텍처 학습 자료 종합 인덱스.

## 코드 리뷰 / 합의 운영

- [Google Engineering Practices — Code Review](https://google.github.io/eng-practices/review/) — Google 의 PR 리뷰 정책. 본 프로젝트의 single-author write subject 정책 (#48) 의 입력.
- [Conventional Commits](https://www.conventionalcommits.org/) — 커밋 메시지 표준. 본 프로젝트는 한국어 3 섹션 양식 사용 중.
- [How to Write a Git Commit Message (cbeams)](https://cbeams.com/posts/git-commit/) — 메시지 7 규칙 (50자 제목 / 본문 줄 길이 등).
- [Atlassian — Trunk-based development](https://www.atlassian.com/continuous-delivery/continuous-integration/trunk-based-development) — main 단일 줄기 + short-lived feature branch.

## 팀 운영 / 리더십

- [Manager's Path (Camille Fournier)](https://www.oreilly.com/library/view/the-managers-path/9781491973882/) — tech-lead → manager 진입 시 표준. 30-resources/books 미러 권장.
- [Staff Engineer's Path (Tanya Reilly)](https://www.oreilly.com/library/view/the-staff-engineers/9781098118723/) — IC 트랙 leadership 패턴.
- [LeadDev articles](https://leaddev.com/articles) — 엔지니어링 리더십 사례 모음.

## 의사 결정 / 문서화

- [ADR (Architecture Decision Records) — Michael Nygard](https://github.com/joelparkerhenderson/architecture-decision-record) — 결정 노트 양식의 표준. 본 vault 의 `decision-*.md` 의 입력.
- [TLDR (Technical Leadership Decision Record)](https://github.com/joelparkerhenderson/decision-record) — ADR 확장형. 후속 영향 / 비결정 섹션 포함.
- [RFC process (Rust)](https://github.com/rust-lang/rfcs/blob/master/0000-template.md) — RFC 템플릿 표준. 큰 결정의 합의 흐름.

## 추천 운영 / 도구

- `repo-map` 플러그인 활용 — 작업 시작 전 디렉터리 지도 1 회 주입
- `claude-mem` 플러그인 활용 — 5-소스 long-term memory 자동 surface
- vault 의 `_moc/` 주제 hub 를 사이클 시작점으로
