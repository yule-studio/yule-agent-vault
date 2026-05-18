---
title: "Deep Learning — Hub"
kind: knowledge
project: ai
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T00:00:00+09:00
updated_at: 2026-05-19T05:35:00+09:00
tags: [area, ai, deep-learning, neural-network, hub]
home_hub: ai
related:
  - "[[../ai]]"
  - "[[../ai-senior-curriculum]]"
  - "[[concepts]]"
  - "[[backpropagation]]"
  - "[[training-loop-pytorch]]"
  - "[[../machine-learning/machine-learning]]"
  - "[[../foundations/foundations]]"
---

# Deep Learning — Hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-15 | engineering-agent/tech-lead | placeholder — 작성 예정 노트 목록 |
| v.2.0.0 | 2026-05-19 | engineering-agent/tech-lead | placeholder → 본 hub: concepts / backpropagation / training-loop-pytorch + 학습 순서 |

**[[../ai|↑ ai hub]]** · Phase 2

> 신경망 (NN) / backpropagation / autograd 의 이해는 Transformer / LLM 으로 가는 필수 경로.

---

## 1. 영역 정의

본 영역은 신경망 (Neural Network) 의 기본 구성 요소부터 PyTorch 실용 학습까지를 다룬다. Transformer / LLM 은 [[../llm/llm|↑ llm]] 영역.

본 영역이 정의하는 것:
- perceptron / activation / MLP / forward+backward pass
- backpropagation 의 chain rule + computational graph + autograd
- PyTorch Dataset / DataLoader / Model / Loss / Optimizer / Scheduler 6 구성요소
- 표준 학습 loop + 진단 (학습 곡선 / gradient norm / overfit)
- mixed precision / gradient accumulation / checkpoint
- 흔한 오류 (OOM / NaN / device mismatch / data loader 정체)

본 영역이 정의하지 않는 것:
- ML 일반 이론 / metric / CV / leakage — [[../machine-learning/machine-learning]]
- CNN / RNN / Transformer 구체 아키텍처 — 별도 (작성 예정)
- LLM pretraining / fine-tuning — [[../llm/]], [[../fine-tuning/]]
- 모델 serving / 분산 학습 — [[../mlops/]]

---

## 2. 노트 인덱스

| 노트 | 다루는 주제 |
| --- | --- |
| [[concepts]] | perceptron / activation 함수 / MLP / forward / backward / 보편 근사 정리 / depth vs width |
| [[backpropagation]] | chain rule + computational graph + autograd + reverse-mode + memory vs time trade-off |
| [[training-loop-pytorch]] | Dataset / DataLoader / 표준 학습 loop / wandb / mixed precision / checkpoint / 디버깅 |

> 가이드라인 (전체 AI 영역의 competency / 산출물 / 검증): [[../ai-senior-curriculum]] §7.3

---

## 3. 학습 순서

| 단계 | 진입점 |
| --- | --- |
| 1. ML 기초 (loss / gradient / CV) | [[../machine-learning/concepts]] + [[../machine-learning/loss-and-gradient-descent]] + [[../machine-learning/training-and-evaluation]] |
| 2. 신경망 개념 | [[concepts]] |
| 3. backprop 의 동작 | [[backpropagation]] (chain rule + autograd) |
| 4. PyTorch 첫 학습 — MNIST MLP | [[training-loop-pytorch]] |
| 5. 데이터 / 차트 | [[../foundations/data-analysis]] + [[../foundations/data-visualization]] §9 ML 차트 |
| 6. CNN / RNN / Transformer 로 확장 | (작성 예정) |
| 7. LLM 으로 진입 | [[../llm/llm]] |

---

## 4. 작성 예정 (deep notes)

§7.3 architectural level 산출물.

- `optimizers-sgd-adam.md` — SGD / Momentum / Adam / AdamW / LAMB / LARS 깊이
- `normalization.md` — BatchNorm / LayerNorm / GroupNorm / RMSNorm
- `cnn-basics.md` — convolution / pooling / receptive field / 대표 아키텍처 (LeNet / VGG / ResNet / EfficientNet)
- `rnn-lstm-gru.md` — sequence 모델 / vanishing / gate
- `embeddings.md` — word2vec / GloVe / fastText / contextual embedding
- `transfer-learning.md` — pretrained 모델 활용 / fine-tuning 전략
- `model-compression.md` — pruning / quantization / distillation
- `pitfalls.md` — 학습 깨짐 / NaN / OOM 디버깅
- `activation-functions-deep.md` — ReLU / GELU / SwiGLU / Mish 비교

---

## 5. 외부 자료

| 자료 | 영역 |
| --- | --- |
| Andrej Karpathy — "Neural Networks: Zero to Hero" | ★ 필독 (micrograd / makemore / nanoGPT) |
| 3Blue1Brown — Neural Networks | 직관 |
| "Deep Learning" (Goodfellow / Bengio / Courville) | 교과서 |
| Stanford CS231n (Vision) | CNN 깊이 |
| Stanford CS224n (NLP) | RNN / Transformer |
| fast.ai — Practical Deep Learning | top-down |
| PyTorch tutorials | API |
| "Deep Learning with PyTorch" (Stevens) — 무료 PDF | PyTorch 실전 |

---

## 6. 관련

- [[../ai|↑ ai hub]]
- [[../ai-senior-curriculum]]
- [[../machine-learning/machine-learning]]
- [[../foundations/foundations]]
- [[../foundations/data-analysis]]
- [[../foundations/data-visualization]]
- [[../llm/llm]]
- [[../mlops/]]
