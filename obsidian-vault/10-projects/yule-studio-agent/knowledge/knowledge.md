# knowledge/ — 운영 지식

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 knowledge README hub |

## 이 폴더가 묶는 것

decision / research / task-log 와 달리, **재사용 가능한 운영 지식**.
"이 시스템은 이렇게 동작한다 / 이 컨벤션은 이렇게 쓴다 / 이 도구는
이렇게 운영한다" 의 reference card.

**[[../index|↑ 상위 README]]**

## 하위 영역

- [[plugins/plugins|plugins/]] — F11 11 개 플러그인 운영 카탈로그
  (paste-guard / hookify / repo-map / lsp-preflight / claude-mem /
  tool-call-gate / live-llm-editor / live-research-provider /
  discussion-response / auto-merge-decider / obsidian-vault-push)

## 파일명 컨벤션

```
knowledge-<topic-kebab>.md
```

또는 하위 영역의 경우 `knowledge/<area>/<topic>.md` (area 자체가 명시적
주제를 표현하면 파일명에서 `knowledge-` prefix 생략 가능).

## 운영 규칙

1. knowledge 노트는 **현재 상태의 진실** 을 담는다 — 결정 히스토리는
   `decisions/`, 작업 흐름은 `task-logs/`. 동일 정보를 중복하지 않음.
2. 본문 표준 5 섹션: 무엇인가 / 왜 도입했나 / 어떻게 동작하나 / 운영
   가이드 (켜는 법 / 끄는 법) / 관련 정책·이슈.
3. 시스템 변경 시 본 knowledge 노트를 우선 갱신 (단일 진실 표). 새
   decision 추가 시 본 노트의 status / 관련 섹션 갱신.
4. `knowledge/plugins/` 의 11 노트는 repo `policies/runtime/plugins/`
   와 1:1 미러. repo 가 권위 source — 충돌 시 repo 가 이긴다.

## 관련 hub

- [[../_moc/_moc|_moc/ 전체 인덱스]]
- [[../_moc/plugins-catalog]]
- [[../_moc/manifest-migration]]
