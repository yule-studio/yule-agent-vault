---
title: "CI/CD pitfalls"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T04:46:00+09:00
tags: [devops, cicd, pitfalls]
---

# CI/CD pitfalls

**[[cicd|↑ cicd]]**

---

1. **secret echo / log** → 누설.
2. **secret 평문 git commit** → 영구 leak.
3. **fork PR + secret** → GitHub 자동 X but 잘 모르고 사용.
4. **flaky test** → CI 무시 → 가짜 green.
5. **cache 너무 큼** (1GB+) → 매번 다운로드 비용.
6. **모든 stage 직렬** → 느림 → 병렬.
7. **manual deploy 만** → release 늦음 + 사람 실수.
8. **rollback 자동화 X** → 실패 시 사람 개입 늦음.
9. **`latest` image tag deploy** → 재현 X.
10. **CVE scan 안 함** → 알려진 취약점.
11. **SAST / DAST 없음** → 코드 / runtime 보안.
12. **production deploy 권한 모두 공개** → 누구나 prod.
13. **monorepo path filter X** → 매번 모든 service rebuild.
14. **GitHub Actions free minute 초과** → 비용 spike.
15. **self-hosted runner 보안 X** → public PR 의 코드 실행.
16. **secret rotation 안 함** → 영구 secret.
17. **change log 없음** → 무엇 배포됐는지 X.
18. **canary metric 없음** → 자동 rollback 못 함.
19. **timeout 무한** → stuck job 누적.
20. **branch protection X** → 직접 main push.

---

## 관련

- [[cicd|↑ cicd]]
- [[pipeline-patterns]]
- [[secret-in-pipeline]]
- [[release-strategies]]
