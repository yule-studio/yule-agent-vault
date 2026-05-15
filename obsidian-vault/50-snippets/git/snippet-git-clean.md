---
title: "git clean — untracked 정리"
kind: snippet
project: tooling
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T12:20:00+09:00
tags:
  - snippet
  - git
  - clean
---

# git clean — untracked 정리

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[git|↑ git]]**

## 목적
빌드 산출물 / `.pyc` / 임시 폴더 같은 **untracked 파일** 을 일괄 삭제. **복구 불가능** 하므로 신중.

## 코드

```bash
# 1. 미리보기 (dry-run, 항상 먼저 ★)
git clean -n                          # = --dry-run
git clean -nd                         # 디렉토리도 포함 표시

# 2. 실제 삭제
git clean -f                          # untracked 파일만
git clean -fd                         # 디렉토리도 (가장 흔함)
git clean -fdx                        # .gitignore 무시된 것도 (node_modules / .venv / build/)
git clean -fdX                        # .gitignore 된 것만 (X 대문자)

# 3. interactive
git clean -i                          # 메뉴로 선택

# 4. 특정 경로만
git clean -fd src/
git clean -fd -- "*.pyc"

# 5. 같이 쓰는 패턴: 완전 초기화
git reset --hard HEAD
git clean -fdx
# → 마치 새로 clone 한 상태

# 6. 안전 옵션 — 사람 실수 방지
git config --global clean.requireForce true   # -f 없이는 거부 (기본)
```

## 사용법
- 빌드 캐시 / 패키지 매니저 부산물 정리 = `clean -fdx`
- 의심 가는 파일 확인 = `clean -n` 먼저
- CI 의 clean checkout 흉내 = `reset --hard && clean -fdx`

## 주의
- **복구 불가** — reflog 도 untracked 는 안 잡음
- `.env` / 로컬 secret / `.idea` / `.vscode` 등이 untracked + ignore X 면 같이 날아감 → 항상 `-n` 먼저
- `-x` 는 `.gitignore` 의 보호도 무시 → 의도 명확할 때만

## 관련
- [[snippet-git-undo]] — reset --hard 와 조합
- [[snippet-git-status-stage]] — clean 전 상태 확인
