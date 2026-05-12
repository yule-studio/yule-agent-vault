# 50-snippets/ — 코드 조각

> 짧고 (≤50 줄) 자주 다시 쓰는 **언어 / 도구별 코드 조각**. 패턴이
> 아니라 즉시 복붙 가능한 실용 조각.

**[[../index|↑ vault 인덱스]]**

## 언어 / 도구

| 카테고리 | 진입점 | 예시 |
| --- | --- | --- |
| bash | [[bash/]] | `find` / `xargs` / 환경변수 조작 |
| docker | [[docker/]] | Dockerfile / docker-compose 조각 |
| github-actions | [[github-actions/]] | workflow yaml 조각 |
| java | [[java/]] | Optional 활용 / Stream 패턴 / record |
| python | [[python/]] | dataclass / async / typing |
| spring | [[spring/]] | `@RestController` / `@Transactional` 조각 |
| sql | [[sql/]] | 윈도우 함수 / CTE / 인덱스 힌트 |
| typescript | [[typescript/]] | generic 패턴 / 타입 유틸 |

## 파일명 컨벤션

```
snippet-<topic-kebab>.md
```

예: `snippet-bash-xargs-parallel.md`, `snippet-spring-transactional-rollback.md`.

## 노트 표준 구조

```markdown
# <코드 조각 제목>

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | YYYY-MM-DD | <역할> | 최초 |

## 목적
이 조각이 푸는 한 문장.

## 코드
```<lang>
// 실제 조각
```

## 사용법
- 호출 예
- 환경 / 의존성

## 주의
- 함정 / 가장 흔한 실수

## 관련
- [[40-patterns/.../pattern-X]] (이 조각이 구현하는 패턴)
```

## 운영 규칙

1. **≤50 줄**. 더 크면 패턴이거나 example — `40-patterns/` 또는
   `30-resources/examples/` 로.
2. **즉시 복붙 가능** 해야 함. 의존성 / 환경 변수 명시.
3. 보안 위험 (예: shell injection / SQL injection) 가능한 조각은
   주의 섹션에 명시.
4. 언어 / 도구가 deprecated 되면 `90-archive/deprecated-notes/` 로.

## 자동화 에이전트 라우팅

Snippet → `50-snippets/<language-or-tool>/`

## 관련

- [[../index|↑ vault 인덱스]]
- [[../40-patterns/README|40-patterns]] — 패턴 (조각보다 큼)
- [[../60-troubleshooting/README|60-troubleshooting]] — 에러 해결과 함께
  쓰일 조각이면 cross-link
