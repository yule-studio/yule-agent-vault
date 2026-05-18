---
title: "backpropagation — chain rule / computational graph / autograd"
kind: knowledge
project: ai
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-19T03:30:00+09:00
tags: [ai, deep-learning, backpropagation, autograd, chain-rule]
home_hub: deep-learning
related:
  - "[[deep-learning]]"
  - "[[concepts]]"
  - "[[training-loop-pytorch]]"
  - "[[../machine-learning/loss-and-gradient-descent]]"
---

# backpropagation — chain rule / computational graph / autograd

**[[deep-learning|↑ deep-learning]]**

---

## 1. 목적

본 문서는 신경망의 gradient 계산 알고리즘인 backpropagation 의 정의 / 수학 / 구현 / autograd 의 동작을 정리한다.

본 문서가 정의하는 것:
- chain rule 의 NN 적용
- computational graph 모델
- forward / backward pass 의 동시 추적
- PyTorch autograd 의 내부 동작
- gradient 계산의 메모리 vs 시간 trade-off

본 문서가 정의하지 않는 것:
- NN 의 기본 구조 — [[concepts]]
- optimizer (SGD / Adam / ...) — [[../machine-learning/loss-and-gradient-descent]]
- PyTorch 실용 학습 loop — [[training-loop-pytorch]]

---

## 2. 범위

| 구분 | 포함 |
| --- | --- |
| 대상 | 모든 미분 가능한 NN |
| 수학 깊이 | 핵심 chain rule + 행렬 미분 (논문 수준 아님) |
| 도구 | PyTorch autograd (참조) + NumPy 수동 구현 |

---

## 3. 용어

| 용어 | 정의 |
| --- | --- |
| **forward pass** | 입력 → 출력 + loss 계산. |
| **backward pass** | loss → 모든 파라미터의 gradient 계산. |
| **gradient** | scalar loss 의 vector / matrix 파라미터에 대한 편미분. |
| **chain rule** | 합성 함수의 미분 = 각 단계 미분의 곱. |
| **computational graph** | 연산 → node, 데이터 → edge 로 표현한 DAG. |
| **autograd** | computational graph 추적 + chain rule 자동 적용. |
| **Jacobian** | vector valued function 의 1차 도함수 matrix. |
| **vector-Jacobian product (VJP)** | reverse-mode autodiff 의 핵심 연산. |

---

## 4. 왜 backprop 인가

### 4.1 단순 수치 미분의 한계

```
∂L/∂θ_i ≈ (L(θ + ε·e_i) - L(θ - ε·e_i)) / (2ε)
```

| 한계 | 결과 |
| --- | --- |
| 파라미터 N 개 → 2N 회 forward | NN 의 N = 수백만~수십억 → 불가능 |
| 수치 오차 | 작은 ε 의 부동소수점 오차 |
| 검증용 | gradient checking 에만 사용 |

### 4.2 backprop 의 효율

backprop = **1 회 forward + 1 회 backward** 만으로 모든 파라미터의 gradient 계산.

→ 시간 복잡도 = O(forward 와 같음). NN 학습 자체가 가능한 이유.

---

## 5. chain rule — backprop 의 수학적 핵심

### 5.1 스칼라 chain rule

```
y = f(g(x))
dy/dx = (dy/dg) · (dg/dx)
```

### 5.2 다변수 chain rule

```
L = f(z₁, z₂, ..., z_n)
z_i = g_i(x)
dL/dx = Σ_i (∂L/∂z_i) · (∂z_i/∂x)
```

### 5.3 NN 의 한 layer

```
z = W·x + b
h = σ(z)
L = ...

∂L/∂W = (∂L/∂h) · (∂h/∂z) · (∂z/∂W)
      = (∂L/∂h) · σ'(z) · x^T
```

→ 핵심: **상위 layer 에서 받은 gradient (∂L/∂h)** × **이 layer 의 local 미분 (∂h/∂z · ∂z/∂W)**.

---

## 6. computational graph

### 6.1 모델

연산 → node, 데이터 (tensor) → edge.

```
예: L = (W·x + b - y)²

    x      W      b
    │      │      │
    └──×───┘      │
        │        │
        z ────+──┘
        │
        ŷ ──── y
        │     │
        └──-──┘
            │
          (ŷ-y)
            │
          (ŷ-y)²
            │
            L
```

### 6.2 forward = topological 순회 (입력 → 출력)

각 node 에서 입력 tensor 들로 출력 tensor 계산.

### 6.3 backward = 역방향 topological 순회 (출력 → 입력)

각 node 에서 **상위 gradient × local 미분** 으로 입력에 대한 gradient 계산.

```
∂L/∂L = 1
∂L/∂(ŷ-y) = 2(ŷ-y)
∂L/∂ŷ = ∂L/∂(ŷ-y) · 1 = 2(ŷ-y)
∂L/∂z = ∂L/∂ŷ · 1 = 2(ŷ-y)     # ŷ = z (linear regression)
∂L/∂W = ∂L/∂z · x = 2(ŷ-y) · x
∂L/∂b = ∂L/∂z · 1 = 2(ŷ-y)
∂L/∂x = ∂L/∂z · W = 2(ŷ-y) · W   # x 는 보통 학습 안 함
```

---

## 7. backprop 의 vectorized 형태

### 7.1 단일 dense layer

```
입력  x ∈ R^d_in       (또는 batch [B, d_in])
파라미터  W ∈ R^(d_out × d_in)
           b ∈ R^d_out

forward:
  z = x · W^T + b              # shape [B, d_out]
  h = σ(z)                      # element-wise activation

backward (loss L 에서 ∂L/∂h 받음):
  ∂L/∂z = ∂L/∂h ⊙ σ'(z)          # element-wise
  ∂L/∂b = sum over batch of ∂L/∂z
  ∂L/∂W = (∂L/∂z)^T · x          # outer product, sum over batch
  ∂L/∂x = ∂L/∂z · W              # 이전 layer 로 전달
```

### 7.2 σ' 의 예

| activation | σ(z) | σ'(z) |
| --- | --- | --- |
| sigmoid | 1/(1+e^-z) | σ(z)·(1-σ(z)) |
| tanh | tanh(z) | 1 - tanh²(z) |
| ReLU | max(0, z) | 1 if z>0 else 0 |
| GELU | z · Φ(z) | (analytical approx) |

---

## 8. 손실 + activation 의 결합 — softmax + CE

다중 분류에서 흔히 softmax + cross-entropy 를 결합 사용. 두 단계의 backward 가 매우 단순해진다.

```
forward:
  z = logits ∈ R^K
  p = softmax(z)
  L = -Σ y_c · log(p_c)         (y 는 one-hot)

backward:
  ∂L/∂z = p - y                  # ★ 매우 단순!
```

→ PyTorch `CrossEntropyLoss` 는 내부적으로 softmax + log 결합 (`log_softmax`) + numerical stable 처리.

→ 모델 마지막 layer 는 **logits 만 출력**, loss 계산 시 자동으로 softmax 적용. 직접 softmax 호출 X.

---

## 9. PyTorch autograd 의 내부

### 9.1 동작

```python
import torch

x = torch.tensor([1.0, 2.0], requires_grad=True)
y = x ** 2          # forward — autograd 가 computational graph 기록
z = y.sum()         # forward 끝
z.backward()        # backward — graph 역순회

print(x.grad)       # tensor([2., 4.])  ← ∂z/∂x = 2x
```

### 9.2 핵심 컴포넌트

| 컴포넌트 | 역할 |
| --- | --- |
| `Tensor.requires_grad` | 이 tensor 가 leaf 면 gradient 추적 |
| `Tensor.grad_fn` | 이 tensor 를 만든 backward 함수 |
| `Tensor.grad` | backward 후 누적된 gradient |
| `torch.autograd.Function` | 사용자 정의 forward + backward |
| `loss.backward()` | computational graph 역순회 + .grad 갱신 |

### 9.3 gradient 누적 — `optimizer.zero_grad()` 가 필요한 이유

```python
loss1.backward()    # x.grad = g1
loss2.backward()    # x.grad = g1 + g2     ← 누적!
```

→ 매 step 시작 시 `optimizer.zero_grad()` 로 초기화. RNN BPTT / gradient accumulation 같은 의도된 경우만 안 함.

### 9.4 no_grad / inference_mode

평가 / 추론 시 graph 추적 끄기 → 메모리 절약.

```python
with torch.no_grad():
    val_loss = compute_loss(val_x)        # graph 추적 X

with torch.inference_mode():               # 더 빠름 (PyTorch 1.9+)
    pred = model(x)
```

### 9.5 retain_graph / create_graph

| 옵션 | 사용 |
| --- | --- |
| `retain_graph=True` | 같은 graph 로 여러 번 backward (GAN / 2nd derivative) |
| `create_graph=True` | gradient 의 gradient (Hessian, MAML, double backprop) |

---

## 10. 자동 미분의 모드

| 모드 | 동작 | 효율 |
| --- | --- | --- |
| **forward-mode** | tangent vector 전파 — 1 입력 → 모든 출력 | 입력 적을 때 |
| **reverse-mode (backprop)** | cotangent vector 전파 — 1 출력 → 모든 입력 | 출력 적을 때 (loss=1) |
| **mixed mode** | 합성 | 특수 |

NN 학습은 **출력 = 1 scalar loss, 입력 = 수백만 파라미터** 이므로 **reverse-mode (backprop)** 가 효율.

→ Jacobian-vector product (JVP, forward) vs vector-Jacobian product (VJP, reverse).

---

## 11. 메모리 vs 시간 trade-off

backprop 은 forward 결과 (activation) 를 backward 까지 보존해야 한다.

| 전략 | 메모리 | 시간 |
| --- | --- | --- |
| **standard** | 모든 activation 보존 | forward 1 회 + backward 1 회 |
| **gradient checkpointing** | 일부만 보존, 나머지는 backward 시 재계산 | forward ~2 회 + backward 1 회 |
| **reversible network** | 입력에서 출력 재계산 가능한 구조 | 메모리 매우 적음 + 시간 ↑ |

```python
# PyTorch gradient checkpointing
from torch.utils.checkpoint import checkpoint

class BigModel(nn.Module):
    def forward(self, x):
        # 대형 block 을 checkpoint 로 감쌈
        return checkpoint(self.big_block, x, use_reentrant=False)
```

→ LLM 학습에서 메모리 부족 시 표준 트릭. 메모리 절반 / 시간 1.3 배.

---

## 12. mixed precision (fp16 / bf16) 의 영향

| 측면 | 효과 |
| --- | --- |
| 메모리 절반 | 더 큰 모델 / 큰 batch 가능 |
| 속도 2-3 배 | tensor core 활용 |
| underflow / overflow | loss scaling (fp16 만) / bf16 는 range 가 fp32 와 같아 안전 |

```python
from torch.amp import GradScaler, autocast

scaler = GradScaler()

for x, y in loader:
    optimizer.zero_grad()
    with autocast(device_type='cuda', dtype=torch.float16):
        loss = compute_loss(x, y)
    scaler.scale(loss).backward()        # fp16 backward + loss scale
    scaler.step(optimizer)
    scaler.update()
```

---

## 13. 수동 backprop — Karpathy micrograd 핵심 40 줄

```python
class Value:
    def __init__(self, data, _children=(), _op=''):
        self.data = data
        self.grad = 0.0
        self._backward = lambda: None
        self._prev = set(_children)
        self._op = _op

    def __add__(self, other):
        other = other if isinstance(other, Value) else Value(other)
        out = Value(self.data + other.data, (self, other), '+')
        def _backward():
            self.grad  += 1.0 * out.grad
            other.grad += 1.0 * out.grad
        out._backward = _backward
        return out

    def __mul__(self, other):
        other = other if isinstance(other, Value) else Value(other)
        out = Value(self.data * other.data, (self, other), '*')
        def _backward():
            self.grad  += other.data * out.grad
            other.grad += self.data * out.grad
        out._backward = _backward
        return out

    def tanh(self):
        t = math.tanh(self.data)
        out = Value(t, (self,), 'tanh')
        def _backward():
            self.grad += (1 - t*t) * out.grad
        out._backward = _backward
        return out

    def backward(self):
        # topological sort
        topo, visited = [], set()
        def build(v):
            if v not in visited:
                visited.add(v)
                for child in v._prev:
                    build(child)
                topo.append(v)
        build(self)
        # backward 역순 호출
        self.grad = 1.0
        for v in reversed(topo):
            v._backward()
```

→ Andrej Karpathy "micrograd" 의 핵심. 40 줄로 NN 학습 가능한 autograd 구현.

---

## 14. 디버깅 — gradient checking

backprop 코드의 정확성 검증.

```python
def numerical_gradient(f, x, eps=1e-5):
    grad = torch.zeros_like(x)
    for i in range(x.numel()):
        x_plus  = x.clone(); x_plus.view(-1)[i]  += eps
        x_minus = x.clone(); x_minus.view(-1)[i] -= eps
        grad.view(-1)[i] = (f(x_plus) - f(x_minus)) / (2 * eps)
    return grad

# 비교
analytical_grad = ...  # 직접 구현한 backward
numerical_grad = numerical_gradient(f, x)

relative_error = (analytical_grad - numerical_grad).abs() / \
                 (analytical_grad.abs() + numerical_grad.abs() + 1e-8)
assert relative_error.max() < 1e-5
```

→ custom autograd Function 작성 시 필수. PyTorch `torch.autograd.gradcheck()` 가 자동화.

---

## 15. 흔한 실패 모드

| 실패 | 원인 | 정정 |
| --- | --- | --- |
| `loss.backward()` 에러: leaf 가 아님 | requires_grad=True 안 함 또는 중간 결과를 leaf 로 착각 | 학습 가능 파라미터 = `nn.Parameter()` |
| gradient 가 NaN | log(0) / sqrt(negative) / div by 0 | epsilon 추가 / logsumexp / clamp |
| `optimizer.zero_grad()` 잊음 | gradient 누적 | 매 step 시작 시 호출 |
| backward 후 graph 사용 시 에러 | 메모리 절약을 위해 graph 폐기 | `retain_graph=True` (드물게) |
| `with torch.no_grad()` 없이 eval | 메모리 폭주 | no_grad / inference_mode |
| custom Function 에서 backward 잘못 | shape / chain rule 오류 | `torch.autograd.gradcheck()` |
| in-place 연산 후 backward 에러 | computational graph 가 깨짐 | non-inplace 연산 (`x = x + 1` 대신 `x.add_(1)` 금지) |
| 메모리 부족 | activation 너무 큼 | gradient checkpointing |
| backward 두 번째 시 에러 | graph 가 이미 폐기됨 | `retain_graph=True` 또는 graph 재구성 |

---

## 16. 참고

- [[deep-learning|↑ deep-learning hub]]
- [[concepts]]
- [[training-loop-pytorch]]
- [[../machine-learning/loss-and-gradient-descent]]
- Karpathy — "micrograd" (https://github.com/karpathy/micrograd)
- Karpathy — "Neural Networks: Zero to Hero" (video series)
- Goodfellow — "Deep Learning" §6.5
- Olah — "Calculus on Computational Graphs: Backpropagation" (blog, 2015)
- PyTorch — autograd docs (https://pytorch.org/docs/stable/notes/autograd.html)
