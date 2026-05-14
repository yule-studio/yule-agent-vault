---
title: "라벨 규칙 + Priority Boost"
kind: pattern
project: github-conventions
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T19:50:00+09:00
tags:
  - github
  - label
  - priority
---

# 라벨 규칙 + Priority Boost

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 17 라벨 + priority_boost JSON |

**[[github-conventions|↑ GitHub 공통 작업 규칙]]**

---

## 1. 의도

라벨은 단순 분류 + **자동 우선순위 계산의 입력**.
에이전트가 backlog 정렬 / 작업 분배 시 `priority_boost` 합산.

```
issue.priority = base + Σ label.priority_boost
```

---

## 2. 라벨 표 (17 종)

| 라벨 | priority_boost | 의미 |
| --- | --- | --- |
| `🏗 infrastructure` | **+30** | 기반 인프라 |
| `⚙ setting` | **+25** | 개발 환경 세팅 |
| `🐞 bugfix` | **+25** | 버그 수정 |
| `📦 domain` | **+25** | 도메인 모델 |
| `🗄 schema` | **+25** | 스키마 / 마이그레이션 |
| `🎯 core` | **+25** | 코어 비즈니스 로직 |
| `🌏 deploy` | **+20** | 배포 |
| `🔐 auth` | **+20** | 인증 / 인가 |
| `📬 api` | **+15** | API 통신 |
| `✨ feature` | **+10** | 기능 개발 |
| `🥰 accessibility` | **+5** | 웹접근성 |
| `✅ test` | **+5** | 테스트 |
| `🔨 refactor` | **0** | 리팩토링 |
| `🙋‍♂️ question` | **0** | 질문 / 정보 요청 |
| `💻 crossbrowsing` | **-5** | 브라우저 호환성 |
| `📃 docs` | **-5** | 문서 |
| `🎨 html&css` | **-10** | 마크업 / 스타일링 (표면) |

---

## 3. JSON (에이전트 입력)

```json
{
  "labels": {
    "⚙ setting": {
      "priority_boost": 25,
      "reason": "개발 환경 세팅"
    },
    "✨ feature": {
      "priority_boost": 10,
      "reason": "기능 개발"
    },
    "🌏 deploy": {
      "priority_boost": 20,
      "reason": "배포"
    },
    "🎨 html&css": {
      "priority_boost": -10,
      "reason": "마크업/스타일링 (표면)"
    },
    "🐞 bugfix": {
      "priority_boost": 25,
      "reason": "버그 수정"
    },
    "💻 crossbrowsing": {
      "priority_boost": -5,
      "reason": "브라우저 호환성"
    },
    "📃 docs": {
      "priority_boost": -5,
      "reason": "문서"
    },
    "📬 api": {
      "priority_boost": 15,
      "reason": "API 통신"
    },
    "🔨 refactor": {
      "priority_boost": 0,
      "reason": "리팩토링"
    },
    "🙋‍♂️ question": {
      "priority_boost": 0,
      "reason": "질문/정보 요청"
    },
    "🥰 accessibility": {
      "priority_boost": 5,
      "reason": "웹접근성"
    },
    "✅ test": {
      "priority_boost": 5,
      "reason": "테스트"
    },
    "🏗 infrastructure": {
      "priority_boost": 30,
      "reason": "기반 인프라"
    },
    "📦 domain": {
      "priority_boost": 25,
      "reason": "도메인 모델"
    },
    "🗄 schema": {
      "priority_boost": 25,
      "reason": "스키마/마이그레이션"
    },
    "🔐 auth": {
      "priority_boost": 20,
      "reason": "인증/인가"
    },
    "🎯 core": {
      "priority_boost": 25,
      "reason": "코어 비즈니스 로직"
    }
  }
}
```

---

## 4. 카테고리

### 4.1 ⬆️ 높은 우선 (+20 이상)
**시스템의 기반 / 차단 요인**. 이 작업이 막히면 다른 모두 막힘.
- 🏗 infrastructure, ⚙ setting, 🐞 bugfix, 📦 domain, 🗄 schema, 🎯 core
- 🌏 deploy, 🔐 auth

→ backlog 의 최상단. 즉시 분배.

### 4.2 ⬆️ 중간 (+5 ~ +15)
**기능 / 품질**. 정상 작업 흐름.
- 📬 api, ✨ feature
- 🥰 accessibility, ✅ test

### 4.3 ➖ 중립 (0)
**선호 / 정보**. 즉시 액션 X.
- 🔨 refactor, 🙋‍♂️ question

### 4.4 ⬇️ 낮은 (-5 ~ -10)
**표면 / 후속**. 다른 일 끝난 후.
- 💻 crossbrowsing, 📃 docs, 🎨 html&css

---

## 5. 합산 예

### 5.1 "버그 — 결제 API 의 인증 실패"
- 🐞 bugfix (+25)
- 📬 api (+15)
- 🔐 auth (+20)
- **합계 +60** — 최고 우선

### 5.2 "기능 — 새 API 추가"
- ✨ feature (+10)
- 📬 api (+15)
- **합계 +25**

### 5.3 "CSS 색상 변경"
- 🎨 html&css (-10)
- **합계 -10** — 후순위

### 5.4 "스키마 마이그레이션 + 코어 로직"
- 🗄 schema (+25)
- 🎯 core (+25)
- 📦 domain (+25)
- **합계 +75** — 압도적 우선

---

## 6. 라벨 그룹 (의미별)

### 6.1 종류 (kind)
- 🐞 bugfix, ✨ feature, 🔨 refactor, 📃 docs, ✅ test, 🙋‍♂️ question

### 6.2 영역 (area)
- 🎨 html&css, 📬 api, 🗄 schema, 🔐 auth, 🎯 core, 📦 domain, 💻 crossbrowsing, 🥰 accessibility

### 6.3 활동 (action)
- 🌏 deploy, ⚙ setting, 🏗 infrastructure

이슈에 **각 그룹에서 1개씩** 권장 (kind + area + (action)).

---

## 7. 자동 라벨링 (선택)

`.github/labeler.yml` (actions/labeler):
```yaml
🎨 html&css:
  - "**/*.css"
  - "**/*.scss"
  - "**/*.html"

📬 api:
  - "src/**/controller/**"
  - "src/**/api/**"

🗄 schema:
  - "migrations/**"
  - "src/**/entity/**"

✅ test:
  - "**/test/**"
  - "**/*.test.ts"
  - "**/*Test.java"

📃 docs:
  - "docs/**"
  - "**/*.md"

🐞 bugfix:
  # 자동 라벨링 어려움 — title prefix 또는 사용자 수동
```

GitHub Action:
```yaml
- uses: actions/labeler@v5
  with:
    repo-token: ${{ secrets.GITHUB_TOKEN }}
```

---

## 8. 라벨 일관성 (`.github/labels.yml`)

repo 마다 라벨 동기화:
```yaml
- name: "🏗 infrastructure"
  color: "0e8a16"
  description: "기반 인프라"

- name: "⚙ setting"
  color: "0e8a16"
  description: "개발 환경 세팅"

- name: "🐞 bugfix"
  color: "d73a4a"
  description: "버그 수정"

- name: "✨ feature"
  color: "a2eeef"
  description: "기능 개발"

# ... (모든 17 라벨)
```

`github-label-sync` 또는 `crazy-max/ghaction-github-labeler` 로 적용:
```yaml
- uses: crazy-max/ghaction-github-labeler@v5
  with:
    yaml-file: .github/labels.yml
```

---

## 9. priority 계산 알고리즘 (의사 코드)

```python
def calculate_priority(issue) -> int:
    base = 0
    if issue.has_label("blocker") or issue.milestone_due_soon():
        base += 50

    boost = sum(LABEL_RULES[label].priority_boost for label in issue.labels
                if label in LABEL_RULES)

    if issue.age_days > 30:
        boost -= 5    # 오래된 이슈는 우선순위 ↓

    return base + boost
```

에이전트는 `priority` 내림차순으로 작업.

---

## 10. 라벨 검증 / CI

PR 라벨 검증:
```yaml
# 모든 PR 에 kind 라벨 1 개 이상
- uses: mheap/github-action-required-labels@v5
  with:
    mode: minimum
    count: 1
    labels: "🐞 bugfix, ✨ feature, 🔨 refactor, 📃 docs, ✅ test, ⚙ setting"
```

→ 잘못된 라벨 / 누락 PR 차단.

---

## 11. 관련

- [[github-conventions]] — hub
- [[pattern-workflow]] — priority 기반 작업 분배
- [[pattern-pr-template]] — PR 의 라벨
- 모든 이슈 템플릿의 "priority_boost" 섹션
