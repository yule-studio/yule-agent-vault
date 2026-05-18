---
title: "loss-and-gradient-descent — 손실 함수 + 최적화 알고리즘"
kind: knowledge
project: ai
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-19T02:30:00+09:00
tags: [ai, machine-learning, loss, gradient-descent, optimization]
home_hub: machine-learning
related:
  - "[[machine-learning]]"
  - "[[concepts]]"
  - "[[training-and-evaluation]]"
  - "[[../deep-learning/concepts]]"
  - "[[../deep-learning/backpropagation]]"
---

# loss-and-gradient-descent — 손실 함수 + 최적화 알고리즘

**[[machine-learning|↑ machine-learning]]**

---

## 1. 목적

본 문서는 ML 모델의 학습이 작동하는 두 축 — **손실 함수 (loss)** 와 **gradient descent (최적화)** — 의 정의 / 변형 / 선택 기준을 정리한다.

본 문서가 정의하는 것:
- 손실 함수의 정의 + task 별 표준 손실
- gradient 의 의미 + 직접 계산 vs autograd
- batch / mini-batch / stochastic gradient descent 의 차이
- learning rate / momentum / adaptive optimizer 의 동작
- 학습이 깨지는 원인 (vanishing / exploding / NaN / 진동)

본 문서가 정의하지 않는 것:
- 신경망의 backprop chain rule — [[../deep-learning/backpropagation]]
- 학습 곡선 분석 — [[training-and-evaluation]] §6

---

## 2. 범위

| 구분 | 포함 |
| --- | --- |
| 대상 | 미분 가능한 모든 모델 (선형 / logistic / 신경망 / NN 외 일부) |
| 제외 | 트리 모델의 split 기준 (entropy / gini) — 별도 |
| 도구 | scikit-learn / PyTorch / NumPy |

---

## 3. 용어

| 용어 | 정의 |
| --- | --- |
| **loss / cost** | 모델 출력 vs 정답의 차이를 1 숫자로. 학습 = loss 최소화. |
| **objective** | 학습이 최적화하려는 함수. loss + regularization term. |
| **gradient (∇L)** | loss 의 파라미터에 대한 편미분 vector. |
| **gradient descent** | gradient 의 반대 방향으로 파라미터 갱신. |
| **learning rate (η)** | 한 step 의 크기. |
| **step / iteration** | 1 회 파라미터 갱신. |
| **epoch** | 전체 데이터를 한 번 모두 통과. |
| **batch size** | 1 step 에 사용하는 sample 수. |
| **convergence** | loss 가 더 안 줄어드는 상태. |
| **local minimum / saddle point** | gradient = 0 이지만 global 이 아닐 수 있음. |

---

## 4. 손실 함수 — task 별 표준

### 4.1 회귀 (regression)

#### MSE (Mean Squared Error)

```
L = (1/N) Σ (ŷ - y)²
```

| 특성 | 결과 |
| --- | --- |
| 큰 오차에 가중치 ↑ (제곱) | outlier 에 민감 |
| 미분 부드러움 | gradient descent 안정 |
| 단위 | 원 단위² (해석 시 √MSE = RMSE) |

#### MAE (Mean Absolute Error)

```
L = (1/N) Σ |ŷ - y|
```

| 특성 | 결과 |
| --- | --- |
| 모든 오차 동일 가중 | outlier 에 둔감 |
| 0 에서 미분 불연속 | subgradient 사용 |
| 단위 | 원 단위 |

#### Huber loss

MSE 와 MAE 의 결합. 작은 오차 = MSE, 큰 오차 = MAE.

```
L = { 0.5(ŷ-y)²              if |ŷ-y| ≤ δ
    { δ(|ŷ-y| - 0.5δ)         otherwise
```

→ outlier 가 있지만 대부분 정상인 데이터에 적합.

### 4.2 이진 분류

#### Binary Cross-Entropy (BCE) / log loss

```
L = -(1/N) Σ [y·log(p) + (1-y)·log(1-p)]
```

| 특성 | 결과 |
| --- | --- |
| p = sigmoid(z) 와 결합 | logistic regression / NN 표준 |
| 모델이 확신 + 틀림 → 매우 큰 loss | calibration 강제 |
| numerically stable 버전 | `BCEWithLogitsLoss` (PyTorch) |

#### Hinge loss

```
L = max(0, 1 - y·ŷ)         (y ∈ {-1, +1})
```

→ SVM 의 표준. 마진 안에 있으면 0, 밖이면 선형 페널티.

### 4.3 다중 분류

#### Categorical Cross-Entropy

```
L = -(1/N) Σ Σ y_c · log(p_c)
        (y_c 는 one-hot 또는 soft label)
```

| 특성 | 결과 |
| --- | --- |
| softmax 와 결합 | NN 의 마지막 layer 표준 |
| numerically stable | `CrossEntropyLoss` (PyTorch — softmax 내장) |
| label smoothing 호환 | `(1-ε)·one-hot + ε/K` |

#### Focal loss

```
L = -(1-p)^γ · log(p)
```

| 특성 | 결과 |
| --- | --- |
| 쉬운 sample (high p) 의 loss 감소 | 어려운 sample 에 집중 |
| 클래스 불균형 | 객체 검출 (RetinaNet) 표준 |

### 4.4 sequence / LLM

#### Next-token cross-entropy

```
L = -(1/T) Σ log p(x_t | x_<t)
```

→ LLM 의 표준. T 개 token 각각에 대한 cross-entropy.

#### Perplexity

```
PPL = exp(L)
```

→ 평가 지표 (loss 의 지수 변환). 낮을수록 좋음.

### 4.5 선택 기준

| 상황 | 권장 loss |
| --- | --- |
| 회귀 + outlier 없음 | MSE / RMSE |
| 회귀 + outlier 많음 | MAE / Huber |
| 이진 분류 | BCE (sigmoid + BCE = BCEWithLogits) |
| 다중 분류 (one-hot) | Cross-Entropy (softmax + CE 결합) |
| 다중 분류 + 클래스 불균형 | Focal loss / weighted CE |
| 회귀이지만 분위 (quantile) | Quantile loss |
| 순위 학습 | Pairwise (RankNet) / Listwise (LambdaRank) |
| LLM pretraining | Next-token CE |
| contrastive | InfoNCE / triplet loss |

---

## 5. gradient — 의미와 계산

### 5.1 1차원 직관

```
L(θ) 가 컵 모양 곡선이면:
  - gradient = dL/dθ
  - gradient > 0 → 우측 가니 loss ↑ → 좌측으로 가야 함
  - gradient < 0 → 좌측 가니 loss ↑ → 우측으로 가야 함

→ θ_{t+1} = θ_t - η · gradient
```

### 5.2 다차원

여러 파라미터 vector `θ = [w₁, w₂, ..., w_n, b]` 의 경우 각각의 편미분:

```
∇L = [∂L/∂w₁, ∂L/∂w₂, ..., ∂L/∂w_n, ∂L/∂b]
```

→ 모든 파라미터를 동시에 갱신.

### 5.3 직접 계산 vs autograd

| 방법 | 동작 |
| --- | --- |
| **직접 계산 (analytical)** | 수학으로 미분 식 도출 후 코드로 (선형 회귀 등 단순 모델) |
| **수치 미분 (numerical)** | (L(θ+ε) - L(θ-ε)) / 2ε — 검증용 (느림, 오차 큼) |
| **autograd** | computational graph 추적해서 chain rule 자동 — PyTorch / TensorFlow / JAX 표준 |

```python
# PyTorch autograd 예
import torch

w = torch.tensor([1.0, 2.0], requires_grad=True)
x = torch.tensor([3.0, 4.0])
y_true = torch.tensor(20.0)

y_pred = (w * x).sum()              # forward
loss = (y_pred - y_true) ** 2       # forward

loss.backward()                      # autograd 가 ∂loss/∂w 계산
print(w.grad)                        # tensor([ -2*(y_pred-y_true)*x_i ])
```

상세: [[../deep-learning/backpropagation]].

---

## 6. gradient descent 변형

### 6.1 Batch GD (full-batch)

```
gradient = (1/N) Σ_{i=1}^N ∇L_i(θ)
θ = θ - η · gradient
```

| 특성 | 결과 |
| --- | --- |
| 전체 데이터로 1 step | 정확한 gradient |
| 메모리 N 배 필요 | 큰 데이터 불가 |
| step 적음 | 수렴 빠르지만 1 step 의 비용 큼 |

### 6.2 Stochastic GD (SGD, batch_size=1)

```
for i in range(N):
    gradient = ∇L_i(θ)
    θ = θ - η · gradient
```

| 특성 | 결과 |
| --- | --- |
| 1 sample 마다 갱신 | gradient noisy → local minimum 탈출에 도움 |
| 매우 빠름 | 큰 데이터 OK |
| 수렴 진동 | 작은 learning rate 필요 |

### 6.3 Mini-batch GD (현재 표준, batch_size = 32 ~ 512)

```
for batch in batches:
    gradient = (1/|batch|) Σ_{i ∈ batch} ∇L_i(θ)
    θ = θ - η · gradient
```

| 특성 | 결과 |
| --- | --- |
| GPU 병렬 활용 | 빠른 학습 + 안정 |
| batch size 작음 → noisy / 일반화 ↑ | small batch 가 더 잘 일반화 (논쟁 있음) |
| batch size 큼 → 안정 / 빠른 epoch | 매우 큰 batch 는 lr 조정 필요 (LARS / LAMB) |

→ 모던 표준: 32 ~ 512. LLM 같은 큰 모델은 4k+ (gradient accumulation 으로).

---

## 7. learning rate (η) — 가장 중요한 hyperparameter

### 7.1 너무 크면 / 너무 작으면

```
큼:        ╱╲    ╱╲    ╱╲          → 진동 / NaN
          ╱  ╲  ╱  ╲  ╱  ╲
작음:    ╲                            → 수렴 느림 / local minimum
           ╲___
적절:      ╲___                       → 부드러운 수렴
              ╲___
                  ╲___
```

### 7.2 학습률 탐색

| 방법 | 동작 |
| --- | --- |
| **manual** | 1e-1, 1e-2, 1e-3, ... log scale |
| **LR finder (Smith 2017)** | lr 을 점진 증가하며 loss 측정 — 가장 빠른 감소 직전이 적정 |
| **scheduler** | epoch 진행에 따라 lr 변경 |

### 7.3 LR scheduler

| 종류 | 동작 |
| --- | --- |
| **StepLR** | 매 N epoch 마다 곱하기 γ |
| **MultiStepLR** | 지정 epoch 에 곱하기 γ |
| **ExponentialLR** | 매 epoch γ 곱 |
| **CosineAnnealingLR** | cosine 곡선으로 감소 — 모던 표준 |
| **ReduceLROnPlateau** | val loss 정체 시 감소 — 안정적 |
| **OneCycleLR** | 처음 증가 → 정점 → 감소 (Smith) — superconvergence |
| **WarmUp + Cosine** | 처음 작은 lr 부터 증가 (linear warm-up) 후 cosine — LLM / Transformer 표준 |

```python
# PyTorch 예 — CosineAnnealing + Warm-up
from torch.optim.lr_scheduler import OneCycleLR

optimizer = torch.optim.AdamW(model.parameters(), lr=1e-3)
scheduler = OneCycleLR(optimizer, max_lr=3e-3, total_steps=10000)

for batch in dataloader:
    optimizer.zero_grad()
    loss = compute_loss(batch)
    loss.backward()
    optimizer.step()
    scheduler.step()
```

---

## 8. 모던 optimizer

### 8.1 SGD + Momentum

```
v_{t+1} = μ·v_t + ∇L(θ_t)
θ_{t+1} = θ_t - η·v_{t+1}
```

| 효과 | 결과 |
| --- | --- |
| 누적 gradient 의 관성 | local 진동 완화 / 빠른 수렴 |
| μ (momentum) 보통 0.9 | (필요시 0.99) |

→ 비전 / 표 데이터에 여전히 강력. ResNet 학습의 표준.

### 8.2 AdaGrad

```
G_t = G_{t-1} + (∇L)²
θ = θ - (η / √G_t) · ∇L
```

| 효과 | 결과 |
| --- | --- |
| 차원별 lr 자동 적응 | 희소 feature 에 강함 |
| G 누적 → lr 영원히 감소 | 학습 멈춤 위험 |

### 8.3 RMSProp

```
G_t = β·G_{t-1} + (1-β)·(∇L)²
θ = θ - (η / √G_t) · ∇L
```

→ AdaGrad 의 G 누적을 EMA 로 — 학습 멈춤 방지.

### 8.4 Adam (현재 가장 흔한 default)

momentum + RMSProp 결합 + bias correction.

```
m_t = β₁·m_{t-1} + (1-β₁)·∇L          (1차 momentum)
v_t = β₂·v_{t-1} + (1-β₂)·(∇L)²        (2차 momentum)
m̂_t = m_t / (1 - β₁^t)                  (bias correction)
v̂_t = v_t / (1 - β₂^t)
θ = θ - η · m̂_t / (√v̂_t + ε)
```

| hyperparameter | 표준 |
| --- | --- |
| β₁ | 0.9 |
| β₂ | 0.999 |
| ε | 1e-8 |
| η | 1e-3 ~ 1e-4 |

### 8.5 AdamW (LLM / Transformer 표준)

Adam + 정확한 weight decay 분리.

```
θ = θ - η · (m̂_t / (√v̂_t + ε) + λ·θ)
                                  ↑
                          weight decay (L2 가 아닌 직접)
```

→ Transformer / LLM 학습의 default. β₁=0.9, β₂=0.95, weight_decay=0.1 흔함.

### 8.6 비교

| optimizer | lr 민감도 | 일반화 | 사용 |
| --- | --- | --- | --- |
| SGD + momentum | 높음 | 우수 (잘 튜닝 시) | 비전 / 표 데이터 |
| Adam | 낮음 | 보통 | NLP / 빠른 실험 |
| AdamW | 낮음 | 좋음 (weight decay) | LLM / Transformer |
| LAMB / LARS | (특수) | 매우 큰 batch | 분산 학습 |

---

## 9. weight initialization

학습 시작 시 파라미터 초기값이 학습 성공을 좌우.

| 방법 | 사용 |
| --- | --- |
| **Xavier / Glorot** | sigmoid / tanh — variance preserving |
| **He (Kaiming)** | ReLU 계열 — variance preserving |
| **truncated normal** | Transformer / BERT |
| **Orthogonal** | RNN |
| **0 초기화 (편향만)** | 가중치는 절대 0 초기화 X (symmetry 깨지지 않음) |

PyTorch 의 기본 layer 는 합리적 default 있음. custom layer 만 신경.

---

## 10. 학습이 깨지는 5 가지 원인

### 10.1 NaN / Inf

| 원인 | 정정 |
| --- | --- |
| lr 너무 큼 → exploding gradient | lr 감소 / gradient clipping (`clip_grad_norm_`) |
| log(0) | softmax + log 결합 (logsumexp / log_softmax) |
| 0 으로 나누기 | epsilon 추가 (`1e-8`) |
| fp16 overflow | mixed precision + loss scaling |

### 10.2 진동 (oscillation)

| 원인 | 정정 |
| --- | --- |
| lr 너무 큼 | lr 감소 |
| batch size 너무 작음 | batch size ↑ 또는 momentum ↑ |
| 데이터 정규화 안 됨 | feature scaling (StandardScaler / 0-1 normalize) |

### 10.3 vanishing gradient

| 원인 | 정정 |
| --- | --- |
| 깊은 NN + sigmoid / tanh | ReLU / GELU 사용 |
| 잘못된 초기화 | Xavier / He / Kaiming |
| 매우 깊은 NN | residual connection / batch norm / layer norm |

### 10.4 exploding gradient

| 원인 | 정정 |
| --- | --- |
| RNN 장기 시퀀스 | gradient clipping |
| 매우 깊은 NN | residual / norm |

### 10.5 local minimum / saddle point

| 원인 | 정정 |
| --- | --- |
| loss landscape 가 험함 | momentum / Adam |
| 작은 batch → noisy gradient | batch 작게 (역설적으로 도움) |
| 같은 minimum 반복 | warm restart (cosine) / 다른 seed |

---

## 11. PyTorch 표준 학습 step

```python
import torch

model = MyModel()
optimizer = torch.optim.AdamW(model.parameters(), lr=1e-3, weight_decay=0.01)
scheduler = torch.optim.lr_scheduler.CosineAnnealingLR(optimizer, T_max=10)
criterion = torch.nn.CrossEntropyLoss()

for epoch in range(num_epochs):
    model.train()
    for x, y in train_loader:
        x, y = x.to(device), y.to(device)

        # 1. forward
        logits = model(x)
        loss = criterion(logits, y)

        # 2. backward
        optimizer.zero_grad()             # 이전 gradient 초기화
        loss.backward()                    # autograd → gradient 계산

        # 3. (optional) gradient clipping
        torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)

        # 4. step (optimizer 가 파라미터 갱신)
        optimizer.step()

    scheduler.step()                       # epoch 마다 lr 조정

    # 5. validation
    model.eval()
    with torch.no_grad():
        val_loss = ...
```

상세: [[../deep-learning/training-loop-pytorch]].

---

## 12. 흔한 실패 모드

| 실패 | 원인 | 정정 |
| --- | --- | --- |
| loss 가 NaN | lr / log(0) / fp16 | §10.1 |
| loss 가 안 줄어듦 | lr 너무 작음 / 데이터 안 정규화 | lr 증가 / scaling |
| loss 가 epoch 마다 진동 | lr 너무 큼 | lr 감소 / scheduler |
| train loss ↓ val loss ↑ | overfitting | regularization (weight decay / dropout) |
| 다른 seed 에서 결과 다름 | local minimum 다름 | 여러 seed 평균 + 보고 |
| `optimizer.zero_grad()` 잊음 | gradient 누적 | 매 step 시작 시 호출 |
| `model.eval()` 안 한 채 validation | dropout / batchnorm 학습 모드 | eval mode 전환 |
| `with torch.no_grad()` 없이 평가 | 메모리 폭주 (gradient 계산) | no_grad 또는 inference_mode |

---

## 13. 참고

- [[machine-learning|↑ machine-learning hub]]
- [[concepts]]
- [[training-and-evaluation]]
- [[../deep-learning/concepts]]
- [[../deep-learning/backpropagation]]
- [[../deep-learning/training-loop-pytorch]]
- "Deep Learning" (Goodfellow) §4, §8
- Smith — "Cyclical Learning Rates" (2017) — LR finder / OneCycle
- Kingma, Ba — "Adam" (2015)
- Loshchilov, Hutter — "Decoupled Weight Decay Regularization" (AdamW, 2019)
- PyTorch optim docs — https://pytorch.org/docs/stable/optim.html
