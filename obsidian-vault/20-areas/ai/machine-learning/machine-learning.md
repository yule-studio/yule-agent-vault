---
title: "Machine Learning — Hub"
kind: knowledge
project: ai
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T00:00:00+09:00
updated_at: 2026-05-19T05:30:00+09:00
tags: [area, ai, machine-learning, hub]
home_hub: ai
related:
  - "[[../ai]]"
  - "[[../ai-senior-curriculum]]"
  - "[[concepts]]"
  - "[[training-and-evaluation]]"
  - "[[loss-and-gradient-descent]]"
  - "[[../deep-learning/deep-learning]]"
  - "[[../foundations/foundations]]"
---

# Machine Learning — Hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-15 | engineering-agent/tech-lead | placeholder — 작성 예정 노트 목록 |
| v.2.0.0 | 2026-05-19 | engineering-agent/tech-lead | placeholder → 본 hub: concepts / training-and-evaluation / loss-and-gradient-descent + 학습 순서 + ai-senior-curriculum link |

**[[../ai|↑ ai hub]]** · Phase 1

> 전통 ML 의 핵심 개념 영역. Deep Learning 의 부모이며 **gradient descent / loss / overfitting** 의 직관이 모든 모델 학습의 기반.

---

## 1. 영역 정의

본 영역은 전통 머신러닝 (선형 / logistic / tree / SVM / k-NN / boosting) 과 ML 의 일반 학습 / 평가 이론을 다룬다. 신경망 / Transformer / LLM 은 [[../deep-learning/deep-learning|↑ deep-learning]] / [[../llm/llm|↑ llm]] 영역.

본 영역이 정의하는 것:
- ML 의 5 학습 종류 (supervised / unsupervised / self-supervised / semi-supervised / reinforcement)
- loss / gradient / optimizer / scheduler
- train / val / test 분리 + cross-validation + leakage 방지
- 평가 metric / 모델 선택
- bias-variance / overfitting / regularization

본 영역이 정의하지 않는 것:
- 신경망 / backprop / autograd — [[../deep-learning/deep-learning]]
- Transformer / Attention / LLM — [[../llm/llm]]
- NumPy / Pandas / Matplotlib — [[../foundations/foundations]]
- MLOps / serving / 모니터링 — [[../mlops/]]

---

## 2. 노트 인덱스

| 노트 | 다루는 주제 |
| --- | --- |
| [[concepts]] | ML 핵심 개념 — 5 학습 종류 / loss / gradient / hypothesis / 평가 / overfitting / metric |
| [[loss-and-gradient-descent]] | 손실 함수 (MSE / CE / Focal / Huber) + optimizer (SGD / Adam / AdamW) + lr scheduler |
| [[training-and-evaluation]] | train/val/test split / k-fold / stratified / time-series CV / hyperparameter tuning / leakage 방지 |

> 가이드라인 (전체 AI 영역의 competency / 산출물 / 검증): [[../ai-senior-curriculum]] §7.2

---

## 3. 학습 순서

| 단계 | 진입점 |
| --- | --- |
| 1. 기초 도구 | [[../foundations/data-analysis]] (NumPy / Pandas) + [[../foundations/data-visualization]] (matplotlib / seaborn) |
| 2. 개념 | [[concepts]] |
| 3. loss / optimizer | [[loss-and-gradient-descent]] |
| 4. 학습 / 평가 절차 | [[training-and-evaluation]] |
| 5. 첫 모델 — 선형 회귀 → logistic regression → random forest → gradient boosting | scikit-learn 튜토리얼 |
| 6. 신경망으로 확장 | [[../deep-learning/concepts]] → [[../deep-learning/training-loop-pytorch]] |

---

## 4. 작성 예정 (deep notes)

§7.2 매트릭스의 architectural level 산출물.

- `bias-variance.md` — bias-variance trade-off 깊이
- `regularization-deep.md` — L1 / L2 / dropout / data aug / early stopping
- `class-imbalance.md` — focal loss / oversampling (SMOTE) / weighted loss
- `tree-models.md` — decision tree / random forest / XGBoost / LightGBM / CatBoost
- `linear-logistic-regression.md` — 가장 단순 모델 분석
- `clustering.md` — k-means / DBSCAN / hierarchical
- `dimensionality-reduction.md` — PCA / t-SNE / UMAP
- `model-interpretation.md` — SHAP / LIME / partial dependence
- `feature-engineering.md` — encoding / scaling / interaction
- `pitfalls.md` — 흔한 실패 + 디버깅

---

## 5. 외부 자료

| 자료 | 영역 |
| --- | --- |
| Andrew Ng — Coursera ML Specialization | 입문 |
| "Hands-On Machine Learning with Scikit-Learn, Keras, and TensorFlow" (Géron, 3rd ed) | 실전 |
| "Pattern Recognition and Machine Learning" (Bishop) | 이론 |
| "The Elements of Statistical Learning" (Hastie / Tibshirani / Friedman) | 통계 깊이 |
| scikit-learn user guide | API |
| StatQuest YouTube | 직관 |
| Andrew Ng — "Machine Learning Yearning" (무료 PDF) | 디버깅 / 평가 |

---

## 6. 관련

- [[../ai|↑ ai hub]]
- [[../ai-senior-curriculum]]
- [[../foundations/foundations]]
- [[../foundations/data-analysis]]
- [[../foundations/data-visualization]]
- [[../deep-learning/deep-learning]]
- [[../llm/llm]]
- [[../mlops/]]
