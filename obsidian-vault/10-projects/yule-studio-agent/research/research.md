# research/ — 리서치 노트

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 research README hub |

## 이 폴더가 묶는 것

외부 자료 / 내부 코드 / 기존 시스템에 대한 **조사 결과**. 결정으로
이어지는 입력이지만 결정 자체는 아니다 — 결정은 `decisions/` 로 따로
간다.

**[[../index|↑ 상위 README]]**

## 파일명 컨벤션

```
research-<topic-kebab>[-issue-<n>].md
```

## 현재 노트 (위가 최신)

- [[research-engineering-knowledge-surface-strengthening]] —
  knowledge surface 강화 방안 조사 (#69 후속)
- [[research-discord-ci-strategy]] — Discord CI 알림 전략 비교
- [[research-harness-team-patterns]] — Harness 회사형 팀 패턴 분석
  (engineering-agent 골격 입력)
- [[research-hermes-agent-architecture-deep-dive]] — Hermes agent
  아키텍처 deep-dive (Yule 흡수 결정 입력)

## 운영 규칙

1. 조사 자료는 반드시 **출처 + 날짜** 명시. 출처 없는 단언은 research
   로 보지 않는다 (`policies/runtime/common/safety.md`).
2. 본문 표준 6 섹션: 조사 배경 / 조사 방법 / 핵심 발견 / 비교 표 /
   한계 / 다음 단계 입력 (어느 결정으로 갈지).
3. research 가 결정으로 변환되면 `decisions/` 에 노트를 추가하고 본 노트
   frontmatter `related` 에 추가.
4. PII / secret 은 PasteGuard 마스킹 통과 후만 본문에 포함.

## 관련 hub

- [[../_moc/_moc|_moc/ 전체 인덱스]]
- [[../_moc/engineering-knowledge-surface]]
- [[../_moc/discord-ci-strategy]]
- [[../_moc/hermes-yule-integration]]
