---
title: "DL 핵심 개념 — perceptron / MLP / activation / forward / backward"
kind: knowledge
project: ai
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-19T03:00:00+09:00
tags: [ai, deep-learning, concepts, neural-network]
home_hub: deep-learning
related:
  - "[[deep-learning]]"
  - "[[backpropagation]]"
  - "[[training-loop-pytorch]]"
  - "[[../machine-learning/concepts]]"
  - "[[../machine-learning/loss-and-gradient-descent]]"
---

# DL 핵심 개념 — perceptron / MLP / activation / forward / backward

**[[deep-learning|↑ deep-learning]]**

---

## 1. 목적

본 문서는 신경망 (Neural Network, NN) 의 기본 구성 요소 — perceptron / 활성화 함수 / 다층 / forward pass / backward pass — 를 정의한다.

본 문서가 정의하는 것:
- perceptron 의 정의 + 한계
- 활성화 함수 종류 + 선택 기준
- MLP (Multi-Layer Perceptron) 의 구조
- forward pass (예측) 와 backward pass (gradient) 의 흐름
- 신경망의 표현력 / 보편 근사 정리
- depth vs width 의 trade-off

본 문서가 정의하지 않는 것:
- backprop 의 수학 (chain rule / Jacobian) — [[backpropagation]]
- PyTorch 학습 loop — [[training-loop-pytorch]]
- CNN / RNN / Transformer 의 구조 — 각 별도 노트
- 손실 함수 / optimizer — [[../machine-learning/loss-and-gradient-descent]]

---

## 2. 범위

| 구분 | 포함 |
| --- | --- |
| 대상 | feedforward NN (MLP) 기본 |
| 도구 | PyTorch (참조) / NumPy (수동 구현) |
| 제외 | 특수 아키텍처 (CNN / RNN / Transformer) |

---

## 3. 용어

| 용어 | 정의 |
| --- | --- |
| **neuron / unit** | 입력의 weighted sum + activation = 1 unit. |
| **weight (W)** | 입력과 곱해지는 학습 가능한 파라미터. |
| **bias (b)** | weighted sum 에 더하는 학습 가능한 스칼라. |
| **layer** | 동일한 입력을 받는 unit 집합. |
| **activation function (σ)** | weighted sum 을 비선형 변환하는 함수. |
| **forward pass** | 입력 → 출력 의 계산. |
| **backward pass** | loss → gradient 의 계산 (역방향). |
| **logit** | 마지막 layer 의 활성화 이전 값 (분류 시 softmax 이전). |
| **embedding** | 이산 token / category 를 연속 vector 로 매핑. |
| **parameter count** | 모델 안 학습 가능한 수 (W + b 모두). |
| **deep** | layer ≥ 3 ~ 정의에 따라 다름. 실용 기준 "비선형 + 다층" 이면 deep. |

---

## 4. perceptron — 가장 단순한 unit

### 4.1 정의

```
입력  x = [x₁, x₂, ..., x_n]
weight  W = [w₁, w₂, ..., w_n]
bias  b

z = W·x + b = Σ w_i·x_i + b
output = σ(z)             # σ = activation function
```

### 4.2 시각적 해석

```
x₁ ─→ ─×w₁─┐
x₂ ─→ ─×w₂─┤
x₃ ─→ ─×w₃─┼─→ + ──→ +b ──→ σ(·) ──→ output
...        │
x_n ─→ ─×w_n─┘
```

### 4.3 단일 perceptron 의 한계

선형 분류만 가능. XOR 같은 비선형 문제 풀 수 없음.

```
AND: x₁=0,x₂=0 → 0      직선 1개로 0/1 분리 가능
     x₁=0,x₂=1 → 0
     x₁=1,x₂=0 → 0
     x₁=1,x₂=1 → 1

XOR: x₁=0,x₂=0 → 0      직선 1개로 분리 불가
     x₁=0,x₂=1 → 1                ↑
     x₁=1,x₂=0 → 1      Minsky&Papert (1969) 의 perceptron 한계
     x₁=1,x₂=1 → 0                ↓
                                 다층 + 비선형 → MLP 가 해결
```

→ Minsky / Papert (1969) 의 "Perceptrons" 가 NN 의 첫 번째 겨울 (1970s) 의 원인. 다층 + 비선형 activation 으로 해결됨.

---

## 5. 활성화 함수 (activation)

### 5.1 왜 필요

비선형 activation 없으면 깊은 NN 도 사실상 1 layer 와 같다. `W₂(W₁x + b₁) + b₂ = W'x + b'` 의 선형 변환.

→ 비선형 activation 이 "층마다 다른 추상화" 를 가능하게 한다.

### 5.2 종류

#### Sigmoid

```
σ(z) = 1 / (1 + e^(-z))
```

| 특성 | 결과 |
| --- | --- |
| 출력 0~1 | 확률 해석 가능 (이진 분류 마지막 layer) |
| z 가 크면 / 작으면 gradient ≈ 0 | vanishing gradient — 깊은 NN 학습 어려움 |
| 모든 출력 양수 | gradient 가 같은 부호 → 학습 비효율 |
| 현재 hidden layer 에선 거의 사용 X | 마지막 layer 의 이진 분류만 |

#### Tanh

```
tanh(z) = (e^z - e^(-z)) / (e^z + e^(-z))
```

| 특성 | 결과 |
| --- | --- |
| 출력 -1~1 | 중심이 0 → sigmoid 보다 학습 효율 ↑ |
| vanishing gradient | 여전히 깊은 NN 어려움 |
| RNN / LSTM 에 흔히 | gate 의 활성화 |

#### ReLU (Rectified Linear Unit)

```
ReLU(z) = max(0, z)
```

| 특성 | 결과 |
| --- | --- |
| 계산 매우 빠름 | training 빠름 |
| z > 0 에서 gradient = 1 | vanishing 없음 → 깊은 NN 학습 가능 |
| z < 0 에서 gradient = 0 | **dead ReLU** — 영원히 학습 안 됨 |
| 현재 hidden 의 default | CNN / MLP 의 표준 |

#### Leaky ReLU

```
LeakyReLU(z) = max(α·z, z)         (α ≈ 0.01)
```

→ dead ReLU 방지. z < 0 에서도 작은 gradient.

#### ELU / SELU / Mish / Swish

```
ELU(z)   = { z          if z ≥ 0
           { α(e^z - 1) if z < 0
Swish(z) = z · sigmoid(βz)
Mish(z)  = z · tanh(softplus(z))
```

→ ReLU 변형. 일부 task 에서 ReLU 보다 약간 좋음. compute 비용 ↑.

#### GELU (Gaussian Error Linear Unit)

```
GELU(z) = z · Φ(z)                  (Φ = Gaussian CDF)
        ≈ 0.5·z·(1 + tanh(√(2/π)·(z + 0.044715·z³)))
```

| 특성 | 결과 |
| --- | --- |
| 부드러운 ReLU 변형 | gradient 항상 정의 |
| Transformer / BERT / GPT 표준 | 현재 LLM 의 default |

#### Softmax (마지막 layer 만)

```
softmax(z)_i = e^(z_i) / Σ_j e^(z_j)
```

| 특성 | 결과 |
| --- | --- |
| 다중 분류 마지막 layer | 출력의 합이 1 (확률) |
| numerically stable 구현 | logsumexp 사용 |
| Cross-Entropy 와 결합 | PyTorch `CrossEntropyLoss` 가 자동 처리 |

### 5.3 선택 매트릭스

| 위치 | 권장 |
| --- | --- |
| hidden layer (일반 MLP / CNN) | ReLU (default) / Leaky ReLU |
| hidden layer (Transformer / BERT / GPT) | GELU |
| RNN / LSTM gate | sigmoid / tanh (구조적 의미) |
| 마지막 layer — 회귀 | (활성화 없음 또는 ReLU) |
| 마지막 layer — 이진 분류 | sigmoid |
| 마지막 layer — 다중 분류 | softmax (또는 logits + CE 직접) |

---

## 6. MLP (Multi-Layer Perceptron)

### 6.1 구조

```
[input]
  x ∈ R^d_in
   │
   ▼
[hidden layer 1]
  z₁ = W₁·x + b₁          (W₁ ∈ R^(h₁ × d_in))
  h₁ = σ(z₁)
   │
   ▼
[hidden layer 2]
  z₂ = W₂·h₁ + b₂          (W₂ ∈ R^(h₂ × h₁))
  h₂ = σ(z₂)
   │
   ▼
... 더 많은 layer
   │
   ▼
[output layer]
  z_L = W_L·h_{L-1} + b_L
  ŷ = σ_L(z_L)            (회귀: identity / 이진: sigmoid / 다중: softmax)
```

### 6.2 PyTorch 코드

```python
import torch
import torch.nn as nn

class MLP(nn.Module):
    def __init__(self, d_in, hidden_sizes, d_out):
        super().__init__()
        layers = []
        prev = d_in
        for h in hidden_sizes:
            layers.append(nn.Linear(prev, h))
            layers.append(nn.ReLU())
            prev = h
        layers.append(nn.Linear(prev, d_out))
        self.net = nn.Sequential(*layers)

    def forward(self, x):
        return self.net(x)

model = MLP(d_in=784, hidden_sizes=[256, 128], d_out=10)
```

### 6.3 파라미터 개수

`Linear(d_in, d_out)` 의 파라미터 = `d_in × d_out + d_out` (W + b).

```
MLP(784 → 256 → 128 → 10):
  784·256 + 256 = 200,960
  256·128 + 128 = 32,896
  128·10 + 10   = 1,290
  -------- 합계 = 235,146
```

→ 모델 크기는 파라미터 수 × bytes (fp32 = 4 bytes, fp16 = 2 bytes). LLM 의 "7B" / "70B" 가 파라미터 수.

---

## 7. forward pass

### 7.1 정의

입력 → 출력 의 일련의 계산. 학습 / 평가 모두에서 실행.

### 7.2 단계별

```
1. x = batch 입력 (shape: [B, d_in])
2. z₁ = x @ W₁ᵀ + b₁          (W₁ shape: [h₁, d_in])
3. h₁ = ReLU(z₁)
4. z₂ = h₁ @ W₂ᵀ + b₂
5. h₂ = ReLU(z₂)
6. logits = h₂ @ W_L^T + b_L
7. (학습) loss = CE(logits, y_true)
   (평가) ŷ = argmax(softmax(logits))
```

### 7.3 batch 의 의미

shape `[B, d_in]` — B 는 batch size. 한 번에 B 개 입력 병렬 계산.

→ GPU 의 matrix 곱셈 성능 활용. batch size ↑ = 효율 ↑ (메모리 한도 내).

---

## 8. backward pass — 흐름만

forward 의 결과 loss 에서 시작해 chain rule 로 각 파라미터의 gradient 계산.

```
loss
  │ ∂L/∂logits
  ▼
logits
  │ ∂L/∂W_L (= ∂L/∂logits · h₂)
  │ ∂L/∂b_L
  │ ∂L/∂h₂ (다음 layer 로 전달)
  ▼
h₂
  │ ∂L/∂z₂ (= ∂L/∂h₂ ⊙ ReLU'(z₂))
  ▼
z₂
  │ ∂L/∂W₂, ∂L/∂b₂, ∂L/∂h₁
  ▼
...
  ▼
입력 (x 에 대한 gradient 는 학습 안 함)
```

상세: [[backpropagation]].

### 8.1 autograd 의 역할

PyTorch / TensorFlow / JAX 의 autograd 가 위 chain rule 을 자동 수행. 사용자는 forward 만 작성.

```python
loss.backward()    # autograd 가 모든 ∂L/∂W, ∂L/∂b 자동 계산
                   # 결과는 W.grad, b.grad 에 저장
```

---

## 9. 보편 근사 정리 (Universal Approximation Theorem)

> "1 hidden layer 의 충분히 넓은 NN 은 모든 연속 함수를 임의의 정확도로 근사할 수 있다." — Cybenko (1989), Hornik (1991)

| 의미 | 결과 |
| --- | --- |
| 이론적 표현력 보장 | NN 이 "이런 함수는 표현 못해" 라는 한계 X |
| 실용적 한계 | 충분한 width / 충분한 데이터 / 충분한 학습 시간 필요 |
| width vs depth | 같은 표현력에 depth 가 width 보다 효율 (특정 함수족) |

→ 1989 의 정리는 "NN 이 강력하다" 의 이론적 근거. 하지만 실용은 깊이 (depth) 에 의존.

---

## 10. depth vs width

### 10.1 비교

| 구조 | 특성 |
| --- | --- |
| **shallow + wide** (1 layer, 1M units) | 표현력 OK, 효율 ↓, overfit 쉬움 |
| **deep + narrow** (10 layer × 64 units) | 같은 표현력에 적은 파라미터, hierarchical feature 학습 |
| **deep + wide** (modern) | 가장 강력, 계산 비용 ↑ |

### 10.2 깊이의 장점

| 장점 | 결과 |
| --- | --- |
| hierarchical 추상화 | 입력 layer 는 edge → 중간은 texture → 상위는 객체 (CNN) |
| 파라미터 효율 | 특정 함수에 적은 파라미터 |
| transfer learning | 하위 layer 재사용 |

### 10.3 깊이의 함정

| 함정 | 정정 |
| --- | --- |
| vanishing gradient | ReLU / 정규화 / residual connection |
| 학습 어려움 | batch norm / layer norm / 적절한 초기화 |
| overfitting | dropout / weight decay / 데이터 augment |

→ ResNet (2015) 의 residual connection (`y = F(x) + x`) 이 1000+ layer 학습을 가능하게 함.

---

## 11. 모델 / 파라미터 / 메모리

### 11.1 메모리 계산

```
모델 메모리 = 파라미터 수 × bytes per parameter
            + activation 메모리 (forward 결과 보존, backward 용)
            + gradient 메모리 (≈ 파라미터 메모리)
            + optimizer state (Adam: 2 × 파라미터 메모리)
```

### 11.2 예 — 235k MLP

| 항목 | fp32 | fp16 |
| --- | --- | --- |
| 파라미터 | 0.94 MB | 0.47 MB |
| gradient | 0.94 MB | 0.47 MB |
| Adam state (m, v) | 1.88 MB | 0.94 MB |
| activation | batch dependent | batch dependent |
| **총** | ≈ 4 MB | ≈ 2 MB |

→ 충분히 작음. CPU 도 OK.

### 11.3 예 — LLM 7B 파라미터

| 항목 | fp16 |
| --- | --- |
| 파라미터 | 14 GB |
| gradient | 14 GB |
| Adam state | 28 GB |
| activation (batch=1, seq=2048) | ~5 GB |
| **총 (학습)** | ~60+ GB |
| **inference 만** | ~14 GB (gradient/optimizer X) |

→ 학습은 80GB A100 / H100 필요. inference 는 24GB GPU (4090) 도 가능.

---

## 12. 흔한 실패 모드

| 실패 | 원인 | 정정 |
| --- | --- | --- |
| 비선형 activation 잊음 | linear stack = 1 layer 와 같음 | ReLU / GELU 추가 |
| sigmoid hidden + 깊은 NN | vanishing gradient | ReLU / 정규화 / residual |
| dead ReLU | z < 0 항상 (잘못된 초기화 / lr) | Leaky ReLU / GELU / lr 조정 |
| softmax + sigmoid 혼용 | 마지막 layer 잘못 | task 별 표 (§5.3) |
| `argmax` 후 backprop | 미분 불가 | softmax + CE (logits 직접) |
| `Linear(d_in, d_out)` 이 `(d_out, d_in)` | shape 혼동 | PyTorch 는 [out, in] |
| forward 만 빨라지길 원하는데 batch=1 | GPU 활용 X | batch ↑ |
| 모델 크기 메모리 vs activation 메모리 혼동 | gradient checkpointing 미사용 | activation 메모리 측정 + checkpoint |
| 모든 layer 같은 lr | layer 별 lr 도움 되는 경우 | LR groups / discriminative fine-tuning |

---

## 13. 참고

- [[deep-learning|↑ deep-learning hub]]
- [[backpropagation]]
- [[training-loop-pytorch]]
- [[../machine-learning/concepts]]
- [[../machine-learning/loss-and-gradient-descent]]
- [[../foundations/foundations]]
- 3Blue1Brown — "Neural Networks" 시리즈
- "Deep Learning" (Goodfellow / Bengio / Courville) §6 — feedforward NN
- Andrej Karpathy — "Zero to Hero" 시리즈 (micrograd / makemore / nanoGPT)
- Cybenko (1989), Hornik (1991) — Universal Approximation
- He et al. (2015) — ResNet
- Hendrycks, Gimpel (2016) — GELU
