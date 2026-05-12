# 20-areas/ — 지속 학습 주제

> 종료 시점 없는 **지속 관심 영역** 의 개념 정리. 프로젝트와 달리 끝이
> 없고, 시간이 지나며 노트가 누적된다.

**[[../index|↑ vault 인덱스]]**

## 영역

| 영역 | 진입점 | 한 줄 |
| --- | --- | --- |
| ai-agent | [[ai-agent/]] | agent-runtime / llm / multi-agent / prompt-engineering / rag / tool-calling |
| backend | [[backend/]] | api-design / architecture / database / java / jpa / mybatis / redis / spring |
| devops | [[devops/]] | (운영 / 인프라 / CI / CD) |
| frontend | [[frontend/]] | (UI / 상태 관리 / 빌드 도구) |
| computer-science | [[computer-science/computer-science]] | algorithm / data-structure / database / network / OS / software-engineering — 사전형 (이코테 기반) |
| career | [[career/]] | certificates / company-research / interview / portfolio / resume |

## 파일명 컨벤션

```
concept-<topic-kebab>.md
```

또는 영역 안에 더 세부 디렉터리가 있으면 `<area>/<subarea>/<topic>.md`.

## 노트 표준 구조

```markdown
# <개념 이름>

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | YYYY-MM-DD | <역할> | 최초 정리 |

## 정의
한 줄로 "이게 무엇인가".

## 왜 중요한가
어떤 문제를 풀기 위한 개념인가.

## 핵심 메커니즘
2-5 개의 핵심 동작.

## 적용 / 예시
실제로 어떻게 쓰는가 (코드 / 다이어그램).

## 관련 / 참고
- [[다른-개념]]
- 외부 자료 (출처 + 날짜)
```

## 운영 규칙

1. **하나의 노트 = 하나의 개념**. 두 개념 섞이면 분리.
2. 단순 복붙 금지 — **내 해석 / 요약 / 적용 예** 가 본문의 50% 이상.
3. 외부 자료 인용 시 출처 + 날짜 필수.
4. 같은 개념이 여러 영역에 걸치면 **한 곳에 본문, 다른 곳은 [[cross-link]]**.
5. 시간 지나 deprecated 되면 `90-archive/deprecated-notes/` 로.

## 자동화 에이전트 라우팅

ConceptNote → `20-areas/<area>/`

매핑은 frontmatter `area` 필드 또는 노트 제목 키워드 기반.

## 관련

- [[../index|↑ vault 인덱스]]
- [[../30-resources/README|30-resources (외부 자료 요약)]] — 자료를 보고
  요약은 30-resources, 개념 정리는 20-areas
- [[../40-patterns/README|40-patterns (재사용 설계 패턴)]] — 개념이
  패턴으로 정착하면 40-patterns 로 추출
