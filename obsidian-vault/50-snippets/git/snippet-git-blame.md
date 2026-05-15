---
title: "git blame — 한 줄의 출처"
kind: snippet
project: tooling
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T12:00:00+09:00
tags:
  - snippet
  - git
  - blame
---

# git blame — 한 줄의 출처

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[git|↑ git]]**

## 목적
이 한 줄을 누가 / 언제 / 왜 추가했는지를 본다.

## 코드

```bash
# 1. 파일 전체
git blame src/api.py

# 2. 특정 줄 범위
git blame -L 100,120 src/api.py
git blame -L 100,+20 src/api.py        # 100 줄부터 20 줄
git blame -L :function_name: src/api.py

# 3. rename / move 따라가기
git blame -C src/api.py                 # 같은 파일 안의 copy
git blame -C -C src/api.py              # 다른 파일에서 copy
git blame -C -C -C src/api.py           # commit 생성 시점에서까지

# 4. 공백 변경 무시 (포맷팅 commit 건너뛰기)
git blame -w src/api.py                 # 공백 무시
git blame --ignore-rev <fmt-commit>     # 특정 commit 의 변경 무시
git config blame.ignoreRevsFile .git-blame-ignore-revs

# 5. 이름 / 날짜 형식
git blame --date=short src/api.py
git blame -e src/api.py                 # 이메일 표시

# 6. 한 줄의 history (line evolution)
git log -L 100,120:src/api.py
git log -L :function_name:src/api.py    # 함수 변화 전체

# 7. blame 의 commit 자세히
git show <blame-commit>                 # 그 commit 의 변경 + 메시지
git log -1 <blame-commit>               # 메시지만
```

## 사용법
- "왜 이 코드?" = blame → 그 commit 의 PR / 이슈 번호 → 의도 파악
- 포맷팅 변경이 blame 을 더럽힌다면 = `.git-blame-ignore-revs` 활용 (GitHub 도 인식)

## 주의
- blame 은 "마지막으로 수정한 사람" → 실제 작성자가 아닐 수 있음 (옮겨졌거나 포맷팅 됐거나)
- `-w` / `--ignore-rev` 없이는 prettier / black 같은 포맷터 한 번으로 모든 줄이 그 commit 으로 잡힘
- IDE / GitHub 의 blame view 가 더 친화적일 때 많음

## 관련
- [[snippet-git-log]] — `-L` 로 라인 history
- [[snippet-git-search]] — 함수 / 문자열 도입 시점
