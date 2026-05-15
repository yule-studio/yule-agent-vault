---
title: "Foundations — Math + Python ML Stack"
kind: knowledge
project: ai
agent: engineering-agent/tech-lead
status: placeholder
created_at: 2026-05-15T00:00:00+09:00
tags: [area, ai, foundations, math, python, beginner]
---

# Foundations — Math + Python ML Stack

**[[../ai|↑ ai hub]]** · Phase 1

> LLM 의 모든 것은 **행렬 곱 + 미분 + Python**. 여기서 막히면 그 위 모두 막힘.

## 1. 영역

| | 핵심 |
| --- | --- |
| **Linear Algebra** | 벡터 / 행렬 / 곱셈 / SVD / eigenvalue |
| **Calculus** | 편미분 / chain rule / gradient |
| **Probability / Statistics** | 분포 / 기대값 / cross-entropy |
| **Python ML stack** | NumPy / Pandas / Matplotlib |
| **PyTorch (또는 JAX)** | Tensor / autograd / nn.Module |

## 2. 학습 우선순위

1. **선형대수의 시각화** — 3Blue1Brown 영상 (Essence of Linear Algebra).
2. **편미분 + chain rule** — Karpathy 의 micrograd (40줄로 backprop).
3. **NumPy + matrix 연산** — Andrej Karpathy 의 노트북 따라.
4. **PyTorch tensor + autograd** — `torch.tensor`, `.backward()`.
5. **Cross-entropy loss + softmax** — 분류의 기본.

## 3. 작성 예정 (deep notes)

- `linear-algebra-basics.md` — vector / matrix / 행렬곱 직관
- `calculus-for-ml.md` — gradient / chain rule / backprop 의 수학
- `probability-basics.md` — 분포 / KL divergence / cross-entropy
- `numpy-essentials.md` — broadcasting / vectorization
- `pytorch-basics.md` — tensor / autograd / nn.Module / DataLoader
- `karpathy-micrograd.md` — backprop 을 40줄로 구현하기

## 4. 외부 자료

- [3Blue1Brown — Essence of Linear Algebra](https://www.3blue1brown.com/topics/linear-algebra) (필독)
- [3Blue1Brown — Essence of Calculus](https://www.3blue1brown.com/topics/calculus)
- [Karpathy — micrograd](https://github.com/karpathy/micrograd) — 첫 작품.
- [PyTorch — Learn the basics](https://pytorch.org/tutorials/beginner/basics/intro.html)
- [fast.ai — Practical Deep Learning](https://course.fast.ai/) — 또 다른 접근.

## 5. 관련

- [[../ai|ai hub]]
- [[../machine-learning/machine-learning]]
- [[../deep-learning/deep-learning]]
