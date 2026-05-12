# yule-agent-vault

> 개인 개발 Wiki + Obsidian Vault + 자동화 에이전트 (yule-studio-agent) 의
> 외부 surface write 대상. 학습 자료 / 프로젝트 문서 / 설계 결정 /
> 에이전트 작업 기록 / 트러블슈팅을 Markdown 으로 관리한다.

![convention](https://img.shields.io/badge/convention-PARA--like-blue)
![filename](https://img.shields.io/badge/filename-kebab--case-green)
![auto-agent](https://img.shields.io/badge/agent-yule--studio--agent-orange)

---

## 구조 한눈에 보기

```
yule-agent-vault/
├── README.md          ← 본 문서 (vault 진입점)
├── CLAUDE.md          ← 에이전트용 운영 규칙
└── obsidian-vault/    ← Obsidian 이 여는 실제 vault
    ├── README.md      ← vault 인덱스 (10 폴더 가지)
    ├── 00-inbox/      ← 분류 전 임시
    ├── 10-projects/   ← 프로젝트별 작업
    ├── 20-areas/      ← 지속 학습 주제
    ├── 30-resources/  ← 책 / 강의 / 공식 문서
    ├── 40-patterns/   ← 재사용 가능한 설계 패턴
    ├── 50-snippets/   ← 코드 조각
    ├── 60-troubleshooting/  ← 에러 해결 기록
    ├── 70-daily/      ← 일일 작업 기록
    ├── 80-templates/  ← Obsidian 템플릿
    └── 90-archive/    ← 보관 (deprecated / migrated)
```

## 시작하기

### 1. Obsidian 으로 열기

```bash
open /Users/<USER>/local-dev/yule-agent-vault/obsidian-vault
```

Obsidian 앱에서 "Open folder as vault" → `obsidian-vault/` 선택.

### 2. 노트 찾기

- **프로젝트 작업** → [obsidian-vault/10-projects](./obsidian-vault/10-projects/)
- **학습 주제** → [obsidian-vault/20-areas](./obsidian-vault/20-areas/)
- **외부 자료** → [obsidian-vault/30-resources](./obsidian-vault/30-resources/)
- **모든 가지** → [obsidian-vault/index.md](./obsidian-vault/index.md)

### 3. 자동화 에이전트로 노트 추가

```bash
yule obsidian sync --session <session-id>
```

자동화 에이전트가 다음 컨벤션으로 vault 에 노트를 작성한다:

- 파일명: `<kind>-<topic-kebab>[-issue-<n>].md`
- 본문: frontmatter + 문서 버전 표 + 표준 섹션
- 위치: kind 별 라우팅 (`10-projects/{project}/decisions/`, `60-troubleshooting/{area}/`, 등)

자세한 매핑: [CLAUDE.md](./CLAUDE.md)

## 디렉터리 책임

| 폴더 | 책임 | 진입점 |
| --- | --- | --- |
| `obsidian-vault/00-inbox/` | 분류 전 임시 저장 | [index](./obsidian-vault/00-inbox/inbox.md) |
| `obsidian-vault/10-projects/` | 프로젝트별 (yule-studio-agent / homelab / spring-sandbox / ...) | [index](./obsidian-vault/10-projects/projects.md) |
| `obsidian-vault/20-areas/` | 지속 학습 주제 (ai-agent / backend / devops / ...) | [index](./obsidian-vault/20-areas/areas.md) |
| `obsidian-vault/30-resources/` | 책 / 강의 / 공식 문서 / 아티클 / 논문 | [index](./obsidian-vault/30-resources/resources.md) |
| `obsidian-vault/40-patterns/` | 재사용 가능한 설계 패턴 | [index](./obsidian-vault/40-patterns/patterns.md) |
| `obsidian-vault/50-snippets/` | 언어 / 도구별 코드 조각 | [index](./obsidian-vault/50-snippets/snippets.md) |
| `obsidian-vault/60-troubleshooting/` | 영역별 에러 해결 기록 | [index](./obsidian-vault/60-troubleshooting/troubleshooting.md) |
| `obsidian-vault/70-daily/` | 일일 작업 기록 | [index](./obsidian-vault/70-daily/daily.md) |
| `obsidian-vault/80-templates/` | Obsidian 노트 템플릿 | [index](./obsidian-vault/80-templates/templates.md) |
| `obsidian-vault/90-archive/` | 보관 (deprecated / migrated / old-projects) | [index](./obsidian-vault/90-archive/archive.md) |

## 컨벤션 한 줄 요약

- **파일명**: 소문자 kebab-case, 날짜 prefix 금지 (날짜는 frontmatter +
  본문 첫 표). 예: `decision-tech-lead-single-write-subject-issue-48.md`
- **본문 첫 줄**: H1 제목 → 문서 버전 표 (`v.N.M.P | 작성일 | 작성자 |
  변경 사항`) → 본문
- **frontmatter**: `title / kind / project / agent / status /
  created_at / related / tags`
- **검증**: `yule-studio-agent` repo 의 `tests/engineering/
  test_obsidian_convention_governance.py` 가 hard rail

자세한 규칙: [CLAUDE.md](./CLAUDE.md) /
[obsidian-vault/10-projects/yule-studio-agent/yule-studio-agent.md](./obsidian-vault/10-projects/yule-studio-agent/yule-studio-agent.md)

## 자동화 에이전트와의 관계

이 vault 는 [yule-studio-agent](https://github.com/yule-studio/yule-studio-agent)
의 외부 write 대상이다. 에이전트가 작성한 노트는 다음 hard rail 을
통과한 페이로드만 들어온다:

1. **PasteGuard** — secret / PII 자동 마스킹
2. **filename_convention.validate_filename** — 파일명 컨벤션 검증
3. **OUTBOUND_VAULT hook** — write 직전 최종 게이트
4. **obsidian-vault-push** (선택) — write 후 자동 git push

운영 정책: yule-studio-agent repo 의 `policies/runtime/vault/naming-convention.md`

## 라이선스 / 사용

개인 vault 이므로 외부 사용 미고려. 본 vault 의 구조 / 컨벤션은
오픈소스 PARA 시스템 + Obsidian 모범 사례 + yule-studio-agent 자동화
요구를 합쳐 자체적으로 다듬은 결과.

---

작성: engineering-agent/tech-lead · 2026-05-12
