---
title: "Foundations — Math + Python ML Stack Hub"
kind: knowledge
project: ai
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T00:00:00+09:00
updated_at: 2026-05-19T05:40:00+09:00
tags: [area, ai, foundations, math, python, hub]
home_hub: ai
related:
  - "[[../ai]]"
  - "[[../ai-senior-curriculum]]"
  - "[[data-analysis]]"
  - "[[data-visualization]]"
  - "[[../machine-learning/machine-learning]]"
  - "[[../deep-learning/deep-learning]]"
---

# Foundations — Math + Python ML Stack Hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-15 | engineering-agent/tech-lead | placeholder — 작성 예정 노트 목록 |
| v.2.0.0 | 2026-05-19 | engineering-agent/tech-lead | placeholder → 본 hub: data-analysis (NumPy/Pandas) + data-visualization (matplotlib/seaborn) + 학습 순서 |

**[[../ai|↑ ai hub]]** · Phase 1

> ML / DL 의 모든 것은 **행렬 곱 + 미분 + Python**. 본 영역은 이 토대.

---

## 1. 영역 정의

본 영역은 ML / DL 학습을 위한 **수학 + Python ML stack + 데이터 분석 / 시각화** 의 인덱스다.

본 영역이 정의하는 것:
- 수학 기초 (linear algebra / calculus / probability) 의 ML 적용
- Python ML stack (NumPy / Pandas / Matplotlib / PyTorch)
- 데이터 분석 절차 (EDA / 결측 / 이상치 / 통계)
- 차트 종류 / 라이브러리 / ML 시각화 패턴

본 영역이 정의하지 않는 것:
- ML 모델 / 알고리즘 — [[../machine-learning/machine-learning]]
- 신경망 / backprop / PyTorch 학습 loop — [[../deep-learning/deep-learning]]
- LLM / Transformer — [[../llm/llm]]

---

## 2. 노트 인덱스

| 노트 | 다루는 주제 |
| --- | --- |
| [[data-analysis]] | NumPy + Pandas — array / DataFrame / groupby / merge / 결측 / 이상치 / 시계열 / 메모리 |
| [[data-visualization]] | matplotlib / seaborn / plotly — 차트 선택 / 객체 모델 / 통계 차트 / ML 시각화 (학습 곡선 / confusion / ROC / SHAP) |

> 가이드라인 (전체 AI 영역의 competency / 산출물 / 검증): [[../ai-senior-curriculum]] §7.1

---

## 3. 학습 순서

| 단계 | 진입점 |
| --- | --- |
| 1. NumPy + Pandas | [[data-analysis]] |
| 2. matplotlib + seaborn | [[data-visualization]] |
| 3. 수학 기초 | 3Blue1Brown 영상 (Linear Algebra / Calculus) + (작성 예정) |
| 4. PyTorch tensor + autograd | (작성 예정) `pytorch-basics.md` |
| 5. micrograd 직접 구현 | [[../deep-learning/backpropagation]] §13 |
| 6. ML 로 진입 | [[../machine-learning/machine-learning]] |

---

## 4. 작성 예정 (deep notes)

§7.1 architectural level 산출물.

- `linear-algebra-basics.md` — vector / matrix / 행렬곱 / SVD / eigenvalue 직관
- `calculus-for-ml.md` — gradient / chain rule / Jacobian / Hessian
- `probability-basics.md` — 분포 (Gaussian / Bernoulli / Multinomial) / KL divergence / cross-entropy 유도
- `pytorch-basics.md` — tensor / autograd / nn.Module / DataLoader 기초
- `karpathy-micrograd.md` — backprop 40 줄 구현 분석
- `notebooks-best-practices.md` — Jupyter 효율 / 셀 의존성 / 재현성

---

## 5. 외부 자료

| 자료 | 영역 |
| --- | --- |
| 3Blue1Brown — "Essence of Linear Algebra" / "Essence of Calculus" | 직관 |
| Khan Academy — Linear Algebra / Calculus / Probability | 입문 |
| "Mathematics for Machine Learning" (Deisenroth / Faisal / Ong) — 무료 PDF | 통합 교과서 |
| "Python for Data Analysis" 3rd (Wes McKinney) | Pandas |
| NumPy / Pandas / Matplotlib / Seaborn 공식 docs | API |
| Karpathy — "micrograd" / "Zero to Hero" | autograd 직관 |
| PyTorch — Learn the Basics | PyTorch 입문 |

---

## 6. 관련

- [[../ai|↑ ai hub]]
- [[../ai-senior-curriculum]]
- [[data-analysis]]
- [[data-visualization]]
- [[../machine-learning/machine-learning]]
- [[../deep-learning/deep-learning]]
