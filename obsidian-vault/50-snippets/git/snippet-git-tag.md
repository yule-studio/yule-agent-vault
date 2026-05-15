---
title: "git tag — 릴리스 / 버전 마킹"
kind: snippet
project: tooling
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T12:10:00+09:00
tags:
  - snippet
  - git
  - tag
  - release
---

# git tag — 릴리스 / 버전 마킹

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[git|↑ git]]**

## 목적
특정 commit 에 "이게 v1.2.0 이다" 같은 라벨을 영구히 박는다. release / deploy 의 기준점.

## 코드

```bash
# 1. 만들기 — lightweight (단순 포인터)
git tag v1.2.0                         # HEAD 에
git tag v1.1.0 abc123                  # 특정 commit 에

# 2. annotated (메시지 + 작성자 + 날짜 ★ 권장)
git tag -a v1.2.0 -m "Release 1.2.0"
git tag -a v1.2.0 abc123 -m "..."

# 3. signed
git tag -s v1.2.0 -m "..."             # GPG signed

# 4. 보기
git tag                                # 전부
git tag -l "v1.*"                      # 패턴
git tag --sort=-creatordate            # 최신 순
git show v1.2.0                        # 태그 + commit 내용

# 5. push (★ 자동 안 됨)
git push origin v1.2.0                 # 특정 태그
git push origin --tags                 # 전부 — 신중

# 6. 삭제
git tag -d v1.2.0                      # 로컬
git push origin :refs/tags/v1.2.0      # 원격
git push origin --delete v1.2.0        # 원격 (대안)

# 7. 옮기기 (이미 있는 태그)
git tag -f v1.2.0 def456               # 강제 덮어쓰기
git push origin v1.2.0 --force         # 위험 — 동료가 이미 fetch 했으면 충돌

# 8. 태그 기반 작업
git switch v1.2.0                      # detached HEAD (읽기)
git switch -c hotfix/v1.2.1 v1.2.0     # 태그 기반 브랜치

# 9. 마지막 태그
git describe --tags                    # 가장 가까운 태그 + 추가 commit 수
git describe --tags --abbrev=0         # 태그 이름만

# 10. 태그 간 변경 (CHANGELOG)
git log v1.1.0..v1.2.0 --oneline
git log v1.1.0..v1.2.0 --pretty=format:'- %s'
```

## 사용법
- semver — `v<major>.<minor>.<patch>`
- release 자동화 (CI) = tag push 가 트리거 (`on: push: tags: 'v*'`)

## 주의
- `git push` 만으로는 태그 push 안 됨 → `--tags` 또는 명시
- 한번 push 된 태그는 절대 변경 금지 (CI / docker image 매핑 깨짐)
- annotated 태그가 표준 — lightweight 는 작성자 / 메시지 없음

## 관련
- [[../../40-patterns/github-conventions/github-conventions]] — release 워크플로우
- [[snippet-git-log]] — 태그 간 changelog
