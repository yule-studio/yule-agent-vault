# CLAUDE.md

## 역할
이 저장소는 개발자 개인 Wiki이자 Obsidian Vault이다.
개발 학습 자료, 프로젝트 문서, 설계 결정, 에이전트 작업 기록, 트러블슈팅 기록을 Markdown으로 관리한다.

## 기본 원칙
- 애매한 문서는 먼저 `00-inbox/`에 저장한다.
- 특정 프로젝트와 직접 관련된 문서는 `10-projects/{project-name}/`에 저장한다.
- 지속적으로 참고할 기술 개념은 `20-areas/`에 저장한다.
- 책, 강의, 공식 문서, 아티클 요약은 `30-resources/`에 저장한다.
- 재사용 가능한 설계/구현 방식은 `40-patterns/`에 저장한다.
- 코드 조각은 `50-snippets/`에 저장한다.
- 에러 해결 기록은 `60-troubleshooting/`에 저장한다.
- 일일 작업 기록은 `70-daily/`에 저장한다.
- 오래되었거나 더 이상 사용하지 않는 문서는 `90-archive/`로 이동한다.

## 문서 작성 규칙
- 모든 문서는 Markdown으로 작성한다.
- 문서 제목은 H1(`#`) 하나만 사용한다.
- 날짜는 `YYYY-MM-DD` 형식을 사용한다.
- 파일명은 소문자 kebab-case를 사용한다.
- 하나의 문서는 하나의 주제만 다룬다.
- 단순 복붙 자료가 아니라, 요약과 내 해석을 포함한다.

## 파일명 규칙
- 개념 정리: `concept-{topic}.md`
- 리서치: `{date}-research-{topic}.md`
- 결정 기록: `{date}-decision-{topic}.md`
- 작업 로그: `{date}-tasklog-{topic}.md`
- 트러블슈팅: `{date}-troubleshooting-{topic}.md`
- 코드 조각: `snippet-{topic}.md`

## 에이전트 Export 규칙
에이전트가 문서를 생성할 때는 아래 기준을 따른다.

- ResearchPack → `10-projects/{project}/research/`
- DecisionRecord → `10-projects/{project}/decisions/`
- TaskLog → `10-projects/{project}/task-logs/`
- MeetingNote → `10-projects/{project}/meeting-notes/`
- ConceptNote → `20-areas/{area}/`
- Troubleshooting → `60-troubleshooting/{area}/`
- Snippet → `50-snippets/{language-or-tool}/`

## 문서 기본 구조

### ResearchPack
```md
# {title}

## 배경
## 수집한 자료
## 핵심 요약
## 판단
## 다음 액션