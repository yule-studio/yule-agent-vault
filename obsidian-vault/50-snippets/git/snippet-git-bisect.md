---
title: "git bisect — 버그 도입 commit 찾기"
kind: snippet
project: tooling
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T12:25:00+09:00
tags:
  - snippet
  - git
  - bisect
  - debug
---

# git bisect — 버그 도입 commit 찾기

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[git|↑ git]]**

## 목적
"어디서부터 깨졌지?" — N 개 commit 을 이진 검색 (log2N 번) 으로 좁힌다.

## 코드

```bash
# 1. 시작
git bisect start
git bisect bad                        # 현재 (또는 HEAD) = 깨짐
git bisect good v1.2.0                # 정상이던 마지막 태그 / commit
# → git 이 자동으로 중간 commit 으로 이동

# 2. 매번 테스트 후 결과 알려줌
# 빌드 + 테스트 …
git bisect good                       # 이 commit 은 정상
# 또는
git bisect bad                        # 깨짐
git bisect skip                       # 빌드 자체 실패 → 건너뛰기

# 3. 자동화 (★ 가장 강력)
git bisect start HEAD v1.2.0
git bisect run pytest tests/test_bug.py    # 종료코드 0 = good, !=0 = bad
# → git 이 알아서 좁혀서 범인 commit 출력

# 4. 더 정교한 script
git bisect run ./check.sh
# check.sh:
#   #!/bin/bash
#   make build || exit 125            # 125 = skip
#   ./run_test.sh || exit 1
#   exit 0

# 5. 끝내고 원래로
git bisect reset                      # 원래 HEAD 로
git bisect reset abc123               # 특정 commit 으로

# 6. 진행 상황
git bisect log                        # 지금까지의 결정 기록
git bisect log > bisect.log           # 저장
git bisect replay bisect.log          # 재실행

# 7. 시각화
git bisect visualize                  # gitk
git bisect view                       # alias
```

## 사용법
- "어제까진 됐는데 오늘 안 돼" = `bisect start HEAD HEAD@{yesterday}` → 자동화
- flaky 한 테스트 = exit 125 로 skip
- 좁혀진 commit = `git show <commit>` 으로 변경 확인

## 주의
- good / bad 결정이 잘못되면 엉뚱한 commit 으로 수렴 — 신중
- exit code: `0=good`, `1-124, 126-127=bad`, `125=skip`
- `git bisect run npm test` 같은 한 줄도 충분히 강력

## 관련
- [[snippet-git-log]] — 의심 commit 들의 history
- [[snippet-git-search]] — 도입 시점의 코드 변화
