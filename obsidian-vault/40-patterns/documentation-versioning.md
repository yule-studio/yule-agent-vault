---
title: "Documentation Versioning — semver for docs"
kind: knowledge
project: vault
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T09:00:00+09:00
tags:
  - patterns
  - documentation
  - versioning
  - governance
---

# Documentation Versioning — semver for docs

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 — vault 노트의 버전 bump 규칙 (semver) |

> Obsidian vault 안 모든 노트의 frontmatter 다음 **문서 버전 표** 가 따르는 표준 규칙.
> 본 정책은 `policies/runtime/vault/documentation-versioning.md` (yule-studio-agent) 와 1:1 미러.

---

## 1. 왜 semver

**왜 필요**
- 노트가 진화할 때 reader 가 "무엇이 바뀐 의미인지" 즉시 알아야 함.
- 매번 Major 만 올리면 (1.0 → 2.0 → 3.0) 작은 보강도 큰 변경처럼 보임 → 신뢰 ↓.
- 매번 Patch 만 올리면 큰 구조 변경도 작게 보임 → reader navigation 충격.

**안 하면 무슨 문제**
- 변경 규모와 버전 mismatch — "v3.0.0 인데 typo 만?" / "v1.0.1 인데 폴더 split 됨?"
- 봇이 작성할 때 일관성 X (어떤 봇은 0.1 올림, 어떤 봇은 1.0 올림).

---

## 2. 정책 — Bump 규칙

| Bump | 의미 | 예시 |
| --- | --- | --- |
| **MAJOR** (X.0.0) | 폴더 / 구조 / navigation 변경 — reader 의 mental model 변경 | 단일 파일 → 폴더 split / 다른 도메인 흡수 / Phase 재구성 / 노트 위치 이동 |
| **MINOR** (X.Y.0) | 새 노트 / 새 흐름 / 새 섹션 추가 — 의미 확장 | 새 implementation 노트 추가 / design-decisions 새 결정 / security 새 layer / 함정 카테고리 추가 |
| **PATCH** (X.Y.Z) | 수정 / 보강 / 함정 추가 — 의미 변화 X | 함정 5→10개 보강 / link 정정 / typo / ASCII → mermaid / "왜" 깊이 추가 |

### 2.1 판별 질문

1. **reader 의 navigation 이 바뀌나?** → MAJOR
2. **새 의미 / 새 흐름 / 새 노트가 추가됐나?** → MINOR
3. **기존 내용 보강 / 수정만?** → PATCH

---

## 3. 형식

```markdown
# <한 줄 제목>

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v2.1.0 | 2026-05-15 | engineering-agent/tech-lead | social-login 흡수 |
| v2.0.0 | 2026-05-14 | engineering-agent/tech-lead | 폴더 split (8 sub-folder) |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |
```

### 3.1 표기 규칙

- `v` 소문자 prefix.
- Major.Minor.Patch — 마침표 구분.
- 옛 버전 위에 새 버전 (descending — 최신이 위).
- 같은 날 여러 bump 가능 — 버전만 다르면 됨.
- 옛 버전 row 영구 보존 (history) — 삭제 X.

### 3.2 옛 형식 (deprecated)

```
v.1.0.0  ❌ — 점 prefix 사용 안 함
v1.0     ❌ — Patch 자리 누락
1.0.0    ❌ — v prefix 누락
```

---

## 4. 적용 예 — signup.md 의 history 재해석

**옛 표 (잘못된 적용)**
```
v.3.0.0 — auth 통합
v.2.0.0 — 폴더 split (12 detail)
v.1.0.0 — 단일 파일
```

→ 모두 Major. 매번 너무 큼.

**새 정책 적용 (올바른 해석)**
```
v4.2.0 — social-login 흡수 (Minor — 새 흐름 추가)
v4.1.0 — mermaid 전환 (실제로는 Patch v4.0.1 이 맞음)
v4.0.0 — Option A 폴더 8개 split (Major — 구조 변경)
v3.0.0 — auth 통합 (Major — 도메인 흡수)
v2.0.0 — 폴더 split (Major — 구조 변경)
v1.0.0 — 단일 파일 (최초)
```

→ Major / Minor / Patch 가 변경 규모와 일치.

---

## 5. 봇의 의무

vault 에 노트를 쓰는 모든 agent (engineering-agent / planning-agent / 등) 는 다음 의무:

| 의무 | 무엇 |
| --- | --- |
| 새 노트 작성 시 | v1.0.0 시작 |
| 노트 수정 시 | 변경 규모 판별 → Major/Minor/Patch bump |
| 옛 버전 row 삭제 X | history 영구 보존 |
| 변경 사항 한 줄 명시 | "주요 변경 사항" 컬럼에 의미 |

자세히: `policies/runtime/vault/documentation-versioning.md` (yule-studio-agent).

---

## 6. 함정 모음

### 함정 1 — 매번 Major 올림
"big bump" 환상 — 변경 규모와 mismatch.
→ §2.1 판별 질문 적용.

### 함정 2 — 옛 버전 row 삭제
history 손실 — 변경 추적 X.
→ 영구 보존, descending 정렬.

### 함정 3 — 변경 사항 vague ("update", "fix")
reader 가 무엇 변경됐는지 모름.
→ 한 줄 구체 ("social-login 흡수", "함정 5→10개 보강").

### 함정 4 — `v.1.0.0` 점 prefix
이전 vault 의 옛 형식. 표준은 `v1.0.0`.

### 함정 5 — 같은 날 여러 bump 무시
한 PR 에 여러 변경 → 한 버전 row 로 압축 X.
→ 각 logical 변경마다 row (단 같은 날 OK).

### 함정 6 — 의미 변화 없는데 Minor 올림
typo / link 정정 = Patch.
→ 의미 추가 없으면 Patch.

### 함정 7 — Patch 가 폴더 변경 포함
구조 변경 = Major.
→ §2 표 참고.

---

## 7. 관련

- `policies/runtime/vault/documentation-versioning.md` — canonical bot policy (yule-studio-agent)
- `policies/runtime/vault/naming-convention.md` — vault 파일명 규칙 (companion)
- [[../../40-patterns/patterns|↑ patterns hub]]
- 외부 — semver.org (semantic versioning 2.0.0)
