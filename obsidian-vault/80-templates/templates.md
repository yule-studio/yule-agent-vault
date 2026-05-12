# 80-templates/ — Obsidian 노트 템플릿

> Obsidian Templater / Templates 코어 플러그인이 사용할 노트 양식.
> 새 노트 만들 때 hot-key 한 번으로 표준 골격 삽입.

**[[../index|↑ vault 인덱스]]**

## 권장 템플릿 (생성 예정)

| 템플릿 | 용도 | 라우팅 |
| --- | --- | --- |
| `template-decision.md` | 결정 노트 | `10-projects/<project>/decisions/` |
| `template-research.md` | 리서치 노트 | `10-projects/<project>/research/` |
| `template-task-log.md` | 작업 로그 | `10-projects/<project>/task-logs/` |
| `template-concept.md` | 개념 정리 | `20-areas/<area>/` |
| `template-resource.md` | 외부 자료 요약 | `30-resources/<category>/` |
| `template-pattern.md` | 설계 패턴 | `40-patterns/<category>/` |
| `template-snippet.md` | 코드 조각 | `50-snippets/<lang>/` |
| `template-troubleshooting.md` | 트러블 해결 | `60-troubleshooting/<area>/` |
| `template-daily.md` | 일일 노트 | `70-daily/<year>/` |

## 템플릿 표준 구조

모든 템플릿은 다음을 포함:

1. **frontmatter** — kind / status / created_at / agent / tags
2. **H1 제목** — `# <{{title}}>`
3. **문서 버전 표** — `| v.1.0.0 | <%tp.date.now()> | <역할> | 최초 |`
4. **kind 별 표준 섹션** — 폴더 README 의 노트 표준 구조 그대로

## 운영 규칙

1. 템플릿은 **새로 만들 때만** 사용. 기존 노트의 컨벤션 강제는
   `filename_convention` 검증으로.
2. 템플릿 갱신 시 폴더 README 의 "노트 표준 구조" 와 sync 유지.
3. Obsidian Templater 의 `<% %>` 구문 사용 (날짜 자동 채움 등).

## Obsidian 설정 위치

- Settings → Core plugins → Templates → Template folder location =
  `80-templates`
- 또는 Community plugin **Templater** 설치 + Template folder =
  `80-templates`

## 관련

- [[../index|↑ vault 인덱스]]
- 각 kind 별 표준 구조는 해당 폴더 README 참조
