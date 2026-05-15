---
title: "Machine Learning — 전통 ML 의 핵심 개념"
kind: knowledge
project: ai
agent: engineering-agent/tech-lead
status: placeholder
created_at: 2026-05-15T00:00:00+09:00
tags: [area, ai, machine-learning, beginner]
---

# Machine Learning — 전통 ML 의 핵심 개념

**[[../ai|↑ ai hub]]** · Phase 1

> Deep Learning 의 부모. **gradient descent / loss / overfitting** 의 직관이 모든 LLM 학습의 기반.

## 1. 핵심 개념

| | 한 줄 |
| --- | --- |
| **Supervised vs Unsupervised** | label 있음 / 없음 |
| **Loss function** | "얼마나 틀렸나" — MSE / cross-entropy |
| **Gradient Descent** | loss 를 줄이는 방향으로 한 발씩 |
| **Learning rate** | 한 발의 크기 |
| **Overfitting** | 학습 데이터만 잘 맞춤. 일반화 X |
| **Regularization** | 오버피팅 방지 (L1, L2, dropout) |
| **Train / Val / Test split** | 학습 / 검증 / 평가 |
| **Bias-Variance Tradeoff** | 너무 단순 vs 너무 복잡 |

## 2. 학습 우선순위

1. **선형 회귀** — y = Wx + b. gradient descent 로 W, b 찾기.
2. **logistic regression** — 분류 + sigmoid + cross-entropy loss.
3. **decision tree / random forest** — 직관적.
4. **kNN / k-means** — 군집 / 거리 기반.
5. **SVM** — 마진 최대화.
6. **gradient boosting (XGBoost / LightGBM)** — kaggle 의 보스.

→ LLM 이전 시대 ML 의 80%. 직접 다 안 만들어도 직관 필요.

## 3. 작성 예정

- `loss-and-gradient-descent.md`
- `supervised-vs-unsupervised.md`
- `overfitting-regularization.md`
- `train-val-test-split.md`
- `bias-variance.md`
- `linear-logistic-regression.md`
- `tree-models.md`
- `cross-entropy-deep-dive.md` — LLM 의 loss

## 4. 외부 자료

- [Andrew Ng — Machine Learning (Coursera)](https://www.coursera.org/learn/machine-learning) — 고전.
- [scikit-learn — tutorials](https://scikit-learn.org/stable/tutorial/index.html)
- [The StatQuest YouTube](https://www.youtube.com/@statquest) — 짧고 명쾌.
- [Hands-On Machine Learning with Scikit-Learn (book)](https://www.oreilly.com/library/view/hands-on-machine-learning/9781492032632/)

## 5. 관련

- [[../ai|ai hub]]
- [[../foundations/foundations]]
- [[../deep-learning/deep-learning]]
