---
title: "PR 템플릿 — Pull Request"
kind: pattern
project: github-conventions
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T19:45:00+09:00
tags:
  - github
  - pr-template
---

# PR 템플릿

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 정리 |

**[[github-conventions|↑ GitHub 공통 작업 규칙]]**

---

## 1. 위치

`.github/PULL_REQUEST_TEMPLATE.md` (repo 루트).

또는 `.github/PULL_REQUEST_TEMPLATE/` 폴더에 여러 양식 가능 (`?template=feature.md`).

---

## 2. 표준 템플릿

```markdown
## 📌 관련 이슈
<!-- 관련있는 이슈 번호(#000)을 적어주세요.
  해당 pull request merge와 함께 이슈를 닫으려면
  closed #Issue_number를 적어주세요 -->

## ✨ 과제 내용
<!-- 과제에 대한 설명을 적어주세요 -->

## 📸 스크린샷(선택)
<!-- 스크린샷이 필요한 과제면 스크린샷을 첨부해주세요 -->

## 📚 레퍼런스 (또는 새로 알게 된 내용) 혹은 궁금한 사항들
<!-- 참고할 사항이 있다면 적어주세요 -->
```

---

## 3. 작성 가이드

### 3.1 관련 이슈

**`closed #N`** 키워드를 쓰면 PR 머지 시 자동으로 이슈 close.

다른 키워드:
- `close #N`, `closes #N`, `closed #N`
- `fix #N`, `fixes #N`, `fixed #N`
- `resolve #N`, `resolves #N`, `resolved #N`

여러 이슈:
```
closed #12, #15
fixes #20
```

참조만 (close X):
```
related to #N
ref #N
```

### 3.2 과제 내용

**무엇** + **왜** + **어떻게**.
```
## ✨ 과제 내용

사용자 목록 페이지에 CSV export 버튼 추가.

- 왜: 관리자가 매주 통계 분석을 위해 사용자 데이터를 수동 복사 → CSV 1-click 으로 단축
- 어떻게:
  - GET /admin/users/export 엔드포인트 추가 (CSV 응답)
  - 권한: ROLE_ADMIN 만
  - 큰 데이터는 streaming (10K+ 행)
- 영향:
  - 새 의존: opencsv 5.7
  - DB read-only — schema 변경 X
```

### 3.3 스크린샷
- UI 변경 = 필수
- before / after 둘 다
- gif > png > 텍스트

### 3.4 레퍼런스 / 새로 배운 내용
- 참고 문서 / blog
- 도전했던 부분 / 함정
- 다음 PR 의 후속 작업

---

## 4. 좋은 PR 의 조건

| 항목 | 권장 |
| --- | --- |
| size | < 400 LoC 변경 (review 용이) |
| commit | 의미 단위로 분할 ([[../../CLAUDE\|커밋 정책]]) |
| 테스트 | 변경마다 (단위 / 통합) |
| 문서 | code 변경 + 문서 변경 같은 PR |
| review | 최소 1 명 approval |
| CI | 모두 green |
| WIP | `draft` 로 시작 |

---

## 5. 커밋 메시지 컨벤션

사용자 메모리 (`feedback_commit_format`):
```
{이모지} #{이슈번호} {유형} commit {N} — {짧은 제목}

변경 이유:
- ...

주요 변경 사항:
- ...

비고:
- ...
```

⚠️ **Co-Authored-By 절대 X**.

PR 머지 방식:
- **Merge commit** — feature 의 commit history 보존 (PR-당-최소-3-commit 정책)
- **Squash** — docs-only 1-commit chore 만
- **Rebase** — clean history 원할 때

자세히 → 사용자 메모리 `feedback_commit_splitting_policy`.

---

## 6. Review 가이드

| 단계 | 무엇 |
| --- | --- |
| 1. 의도 | 이슈 / 설명 읽기 |
| 2. 크기 | 너무 큼? → split 요청 |
| 3. 테스트 | 누락 / 약함? |
| 4. 디자인 | 더 단순한 방법? |
| 5. 디테일 | 변수명 / 에러 처리 / 로깅 |
| 6. 보안 | input 검증 / 권한 / secret |
| 7. 성능 | N+1 / 메모리 / I/O |

코멘트:
- **must-fix** (블로커)
- **nit** (선호)
- **question** (이해 X)
- **praise** (좋은 점)

---

## 7. Draft PR

WIP / RFC 단계 = `draft` 로 생성.
- 디자인 review 받기 위해
- CI 결과 미리 보기
- 의도 공유

`gh pr ready` 또는 GitHub UI 의 "Ready for review" 버튼.

---

## 8. 자동 라벨링 / Assign

`.github/labeler.yml` (actions/labeler):
```yaml
🎨 html&css:
  - "**/*.css"
  - "**/*.scss"
📬 api:
  - "src/**/controller/**"
  - "src/**/api/**"
🗄 schema:
  - "migrations/**"
  - "**/entity/**"
```

또는 PR 본문 / 파일 path 기반 자동 라벨. CODEOWNERS 로 자동 reviewer.

---

## 9. 관련

- [[github-conventions]] — hub
- [[pattern-label-rules]] — PR 의 라벨 priority
- [[pattern-workflow]] — 이슈 → PR → merge 흐름
- 사용자 메모리 `feedback_commit_format` / `feedback_commit_splitting_policy` (.claude/projects/.../memory/ 안)
