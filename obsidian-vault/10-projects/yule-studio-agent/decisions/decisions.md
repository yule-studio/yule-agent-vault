# decisions/ — 결정 노트

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 decisions README hub |

## 이 폴더가 묶는 것

운영자 또는 자동화 에이전트가 내린 **모든 결정의 단일 기록**. 결정의
배경 / 의미 / 비결정 (의도적으로 미루기로 한 것) / 변경 영향 / 후속
액션을 함께 담는다.

**[[../index|↑ 상위 README]]**

## 파일명 컨벤션

```
decision-<topic-kebab>[-issue-<n>].md
```

날짜 / 작성자 / 버전은 frontmatter + 본문 첫 표 에서만 관리.

## 현재 노트 (위가 최신)

- [[decision-product-vs-marketing-cpo-cmo-separation-issue-126]] —
  product-agent (CPO) vs marketing-agent (CMO) 의 7 결정 박스
  (F15 #126)
- [[decision-engineering-knowledge-share-boundary]] — share_scope
  PUBLIC / TEAM_INTERNAL / RESTRICTED 와 외부 surface 노출 게이트
- [[decision-discord-ci-and-cd-boundary]] — Discord CI 알림 경계 +
  CD 책임 분리
- [[decision-hermes-yule-integration]] — Hermes agent → Yule 흡수
  결정 (issue #59)
- [[decision-tech-lead-single-write-subject-issue-48]] —
  engineering-agent 의 외부 write 주체를 tech-lead 1 명으로 단일화
  (issue #48)

## 운영 규칙

1. 결정 한 줄 요약 + 결정 박스 표 + 각 결정의 의미 + 비결정 (의도적)
   + 변경 영향 + 후속 액션 6 섹션을 반드시 갖춘다.
2. 결정이 superseded 되면 status 갱신 + 후속 노트 링크를 본문 첫 줄에
   기록. 파일은 삭제하지 않는다 (히스토리 보존).
3. 같은 주제의 research 노트와 task-log 가 있다면 frontmatter
   `related` 로 cross-link.
4. 결정의 영향 범위가 부서 전체면 `_moc/` 의 해당 hub 에도 [[link]].

## 관련 hub

- [[../_moc/_moc|_moc/ 전체 인덱스]]
- [[../_moc/f15-corporate-structure]]
- [[../_moc/manifest-migration]]
- [[../_moc/discord-ci-strategy]]
- [[../_moc/hermes-yule-integration]]
