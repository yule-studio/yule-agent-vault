# 20-areas/ — 지속 학습 주제 (12 영역)

> 종료 시점 없는 **지속 관심 영역** 의 개념 정리. 프로젝트와 달리 끝이
> 없고, 시간이 지나며 노트가 누적된다. **이론 (CS)** 과 **실무 (영역별)**
> 분리.

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.2.0.0 | 2026-05-13 | engineering-agent/tech-lead | 12 영역 폴더링 골격 (CS 이론 / 실무 분리, 언어·프레임워크별 세분화) |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 — 6 영역 |

**[[../index|↑ obsidian-vault 인덱스]]**

## 12 영역

### 이론

| 영역 | 진입점 | 한 줄 |
| --- | --- | --- |
| 🧮 computer-science | [[computer-science/computer-science]] | 알고리즘 / DS / DB-theory / network / OS / SE / 컴퓨터 구조 / 컴파일러 / 분산 시스템 / 보안 이론 / 디자인 패턴 / 하드웨어 |

### 실무 — 코어

| 영역 | 진입점 | 한 줄 |
| --- | --- | --- |
| 🔧 backend | [[backend/backend]] | _common + Java·Spring / Python·Django/FastAPI / Node·Express/NestJS / Go / Kotlin / Rust / Ruby / PHP / .NET |
| 🎨 frontend | [[frontend/frontend]] | React / Next / Vue / Nuxt / Svelte / SolidJS / Angular + 상태·빌드·테스트·UI·디자인 시스템 |
| 📱 mobile | [[mobile/mobile]] | iOS Swift / Android Kotlin / RN / Flutter / KMP + 아키텍처 / 배포 |
| 🚀 devops | [[devops/devops]] | Linux / Docker / K8s / CI·CD / Cloud (AWS/GCP/Azure/OCI) / IaC / 관측 / SRE / Platform Eng |
| 🤖 ai | [[ai/ai]] | LLM / RAG / Agents / Vector DB / Fine-tuning / MLOps / Evaluation / Safety / Papers |
| 📊 data | [[data/data]] | Data Engineering / Warehouse / Pipeline / Stream / Quality / Modeling / DS / Viz |

### 실무 — 보조

| 영역 | 진입점 | 한 줄 |
| --- | --- | --- |
| 🔒 security | [[security/security]] | Web / App / Network / Cloud / Pentesting / Crypto / Compliance / IR |
| 🗄 database | [[database/database]] | PostgreSQL / MySQL / Redis / Mongo / Elasticsearch / SQLite / Cassandra + 설계 / 튜닝 |

### 사람 / 비즈니스

| 영역 | 진입점 | 한 줄 |
| --- | --- | --- |
| 📦 product | [[product/product]] | PRD / Discovery / Roadmap / OKR / Analytics / Pricing / UR / AB |
| 🎓 career | [[career/]] | certificates / company-research / interview / portfolio / resume |
| 💬 soft-skills | [[soft-skills/soft-skills]] | communication / mentoring / leadership / negotiation / time / learning |

### 레거시 (보존)

| 영역 | 한 줄 |
| --- | --- |
| ai-agent | Yule 자체 에이전트 특화 (agent-runtime / llm / multi-agent / prompt / rag / tool-calling). 일반 AI 는 `ai/` 로 |

---

## 설계 원칙

1. **이론 vs 실무 분리** — `computer-science/` (이론) vs `<영역>/` (실무).
2. **최대 3 단 깊이** — `<area>/<sub-area>/<topic>.md`. 더 깊으면 폴더 추가.
3. **누적 가능한 5-8 자산** — getting-started / project-structure / configuration /
   db-connection / auth / testing / deployment / troubleshooting / comparison.
4. **언어·프레임워크 분리** — backend / frontend / mobile 모두 언어×프레임워크별 폴더.
5. **`_common/` 패턴** — 언어 무관 공통 개념.
6. **폴더명 = 인덱스 파일명** — `backend/backend.md`.
7. **확장 시 기존 깨지 않음** — 새 폴더만 추가.
8. **CS ↔ 실무 cross-link** — frontmatter `related` + 본문 [[link]].

---

## 파일명 컨벤션

```
<kind>-<topic-kebab>[-issue-<n>].md     # 사전 / decision / research / task-log
<folder-name>.md                          # 폴더 인덱스
```

날짜는 frontmatter `created_at` + 본문 문서 버전 표만. 파일명 X.

---

## 노트 표준 구조

```markdown
# <개념 이름>

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | YYYY-MM-DD | <역할> | 최초 정리 |

## 정의 / 왜 중요한가 / 핵심 메커니즘 / 적용 예 / 관련

(사전 수준은 14 섹션 — algorithm 노트 참조)
```

---

## 관련

- [[../index|↑ obsidian-vault 인덱스]]
- [[../00-inbox/links/links|↗ 외부 reference 카탈로그]]
- [[../10-projects/projects|↗ 10-projects (프로젝트)]]
