---
title: "training-loop-pytorch — 실용 PyTorch 학습 가이드"
kind: knowledge
project: ai
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-19T04:00:00+09:00
tags: [ai, deep-learning, pytorch, training, practice]
home_hub: deep-learning
related:
  - "[[deep-learning]]"
  - "[[concepts]]"
  - "[[backpropagation]]"
  - "[[../machine-learning/loss-and-gradient-descent]]"
  - "[[../machine-learning/training-and-evaluation]]"
  - "[[../foundations/data-analysis]]"
---

# training-loop-pytorch — 실용 PyTorch 학습 가이드

**[[deep-learning|↑ deep-learning]]**

---

## 1. 목적

본 문서는 PyTorch 로 신경망을 처음부터 학습 / 평가 / 저장하는 표준 절차를 정리한다.

본 문서가 정의하는 것:
- Dataset / DataLoader / Model / Loss / Optimizer / Scheduler 6 구성 요소
- 표준 학습 루프 (train epoch + val epoch + checkpoint)
- 학습 진단 (loss 곡선 / metric 추적 / gradient 진단)
- 흔한 오류 (device mismatch / OOM / data loader 정체 / NaN) 의 디버깅
- GPU 효율화 (mixed precision / gradient accumulation / pin_memory)
- 모델 저장 / 로드 / inference 전환

본 문서가 정의하지 않는 것:
- NN 의 구조 정의 — [[concepts]]
- 손실 / optimizer 의 수학 — [[../machine-learning/loss-and-gradient-descent]]
- backprop 의 chain rule — [[backpropagation]]
- 데이터 전처리 / EDA — [[../foundations/data-analysis]]

---

## 2. 범위

| 구분 | 포함 |
| --- | --- |
| 대상 | PyTorch 2.x 기준 (PyTorch Lightning / Hugging Face Trainer 미사용) |
| 도구 | torch / torch.utils.data / torch.cuda.amp / torch.optim |
| 제외 | 분산 학습 (DDP / FSDP) — 별도 |
| 제외 | LLM fine-tuning (peft / TRL) — 별도 |

---

## 3. 용어

| 용어 | 정의 |
| --- | --- |
| **Dataset** | sample 1 개를 반환하는 객체 (`__getitem__` / `__len__`). |
| **DataLoader** | Dataset 으로부터 batch 를 만들어 반환하는 iterator. |
| **Model** | `nn.Module` 의 subclass. forward 메서드 보유. |
| **device** | `'cpu'` / `'cuda'` / `'mps'` / `'cuda:0'` 등. tensor 가 살아있는 위치. |
| **state_dict** | 모델 / optimizer 의 학습된 상태 dict (parameter / buffer). |
| **checkpoint** | state_dict + epoch + 추가 metadata 를 저장한 파일. |
| **step** | 1 회 optimizer.step (1 batch). |
| **epoch** | 전체 train set 1 회 통과. |

---

## 4. 표준 학습 루프 — 6 구성 요소

```
1. Dataset / DataLoader (데이터)
2. Model (NN)
3. Loss function (목표)
4. Optimizer (파라미터 갱신)
5. Scheduler (lr 조정 — 선택)
6. Training loop (train epoch + val epoch + checkpoint)
```

### 4.1 전체 구조

```python
import torch
from torch import nn
from torch.utils.data import Dataset, DataLoader
from torch.optim import AdamW
from torch.optim.lr_scheduler import CosineAnnealingLR

device = "cuda" if torch.cuda.is_available() else "cpu"

# 1. Dataset
train_ds = MyDataset(split="train")
val_ds   = MyDataset(split="val")

# 2. DataLoader
train_loader = DataLoader(train_ds, batch_size=64, shuffle=True,
                          num_workers=4, pin_memory=True)
val_loader   = DataLoader(val_ds,   batch_size=128, shuffle=False,
                          num_workers=4, pin_memory=True)

# 3. Model
model = MyModel().to(device)

# 4. Loss
criterion = nn.CrossEntropyLoss()

# 5. Optimizer + Scheduler
optimizer = AdamW(model.parameters(), lr=1e-3, weight_decay=0.01)
scheduler = CosineAnnealingLR(optimizer, T_max=num_epochs)

# 6. Training loop
best_val = float("inf")
for epoch in range(num_epochs):
    train_loss = train_one_epoch(model, train_loader, optimizer, criterion, device)
    val_loss, val_acc = evaluate(model, val_loader, criterion, device)
    scheduler.step()

    print(f"epoch {epoch}: train_loss={train_loss:.4f} val_loss={val_loss:.4f} val_acc={val_acc:.4f}")

    if val_loss < best_val:
        best_val = val_loss
        torch.save({
            "epoch": epoch,
            "model": model.state_dict(),
            "optimizer": optimizer.state_dict(),
            "scheduler": scheduler.state_dict(),
            "val_loss": val_loss,
        }, "checkpoint_best.pt")
```

---

## 5. Dataset 작성

### 5.1 기본 형태

```python
from torch.utils.data import Dataset

class MyDataset(Dataset):
    def __init__(self, split="train"):
        self.x = ...   # numpy / tensor / file path list
        self.y = ...

    def __len__(self):
        return len(self.x)

    def __getitem__(self, idx):
        x_i = self.x[idx]
        y_i = self.y[idx]
        return torch.tensor(x_i, dtype=torch.float32), torch.tensor(y_i, dtype=torch.long)
```

### 5.2 이미지 — torchvision

```python
from torchvision import datasets, transforms

transform = transforms.Compose([
    transforms.Resize((224, 224)),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225]),
])

train_ds = datasets.ImageFolder("data/train", transform=transform)
val_ds   = datasets.ImageFolder("data/val",   transform=transform)
```

### 5.3 텍스트 — Hugging Face datasets

```python
from datasets import load_dataset
from transformers import AutoTokenizer

raw = load_dataset("imdb")
tokenizer = AutoTokenizer.from_pretrained("bert-base-uncased")

def tokenize(batch):
    return tokenizer(batch["text"], padding="max_length", truncation=True, max_length=256)

train_ds = raw["train"].map(tokenize, batched=True)
train_ds.set_format("torch", columns=["input_ids", "attention_mask", "label"])
```

### 5.4 lazy loading (큰 데이터)

```python
class LargeFileDataset(Dataset):
    def __init__(self, file_paths):
        self.paths = file_paths    # 메타만 메모리에

    def __getitem__(self, idx):
        path = self.paths[idx]
        x = np.load(path)           # 실제 로드는 batch 시점
        return torch.from_numpy(x), label[idx]
```

→ 디스크 IO 가 병목이면 `num_workers > 0` + `pin_memory=True`.

---

## 6. DataLoader 옵션

| 옵션 | 의미 | 권장 |
| --- | --- | --- |
| `batch_size` | 한 batch 의 sample 수 | 32-512 (GPU 메모리에 맞춰) |
| `shuffle` | epoch 마다 순서 섞기 | train=True, val=False |
| `num_workers` | 데이터 로드 병렬 worker 수 | CPU 코어의 절반 ~ 전체 |
| `pin_memory` | host 메모리 고정 → GPU 전송 빠름 | GPU 사용 시 True |
| `prefetch_factor` | worker 당 미리 적재할 batch 수 | 2 (기본) |
| `persistent_workers` | epoch 사이 worker 재사용 | num_workers > 0 시 True |
| `drop_last` | 마지막 부분 batch 버림 | BatchNorm 시 True 권장 |
| `collate_fn` | batch 구성 함수 | 가변 길이 / padding 시 |

```python
loader = DataLoader(
    ds,
    batch_size=64,
    shuffle=True,
    num_workers=4,
    pin_memory=True,
    persistent_workers=True,
    drop_last=True,
)
```

---

## 7. train epoch 함수

```python
def train_one_epoch(model, loader, optimizer, criterion, device):
    model.train()                              # ★ dropout / batchnorm 학습 모드
    total_loss = 0.0
    for x, y in loader:
        x, y = x.to(device, non_blocking=True), y.to(device, non_blocking=True)

        # 1. forward
        logits = model(x)
        loss = criterion(logits, y)

        # 2. backward
        optimizer.zero_grad(set_to_none=True)
        loss.backward()

        # 3. (optional) gradient clipping — 폭주 방지
        torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)

        # 4. step
        optimizer.step()

        total_loss += loss.item() * x.size(0)

    return total_loss / len(loader.dataset)
```

| 항목 | 설명 |
| --- | --- |
| `model.train()` | dropout 활성 / batchnorm 학습 통계 사용 |
| `non_blocking=True` | pin_memory + cuda 와 결합 시 async 전송 |
| `set_to_none=True` | zero_grad 가 메모리 약간 효율적 (PyTorch 1.7+) |
| `loss.item()` | scalar 만 가져오기 (graph 분리) |

---

## 8. validation / evaluation 함수

```python
@torch.no_grad()
def evaluate(model, loader, criterion, device):
    model.eval()                               # ★ dropout off / batchnorm 평가 모드
    total_loss = 0.0
    correct = 0
    for x, y in loader:
        x, y = x.to(device), y.to(device)
        logits = model(x)
        loss = criterion(logits, y)
        total_loss += loss.item() * x.size(0)
        correct += (logits.argmax(dim=1) == y).sum().item()

    return total_loss / len(loader.dataset), correct / len(loader.dataset)
```

| 항목 | 설명 |
| --- | --- |
| `@torch.no_grad()` | gradient 추적 X — 메모리 / 속도 |
| `model.eval()` | dropout / batchnorm 평가 모드 |
| `model.train()` 으로 복귀 | evaluate 끝나면 자동 안 됨 — 호출자가 신경 |

---

## 9. 학습 진단 — 무엇을 추적

| 지표 | 측정 | 의미 |
| --- | --- | --- |
| train loss | epoch / step | 학습 진행 |
| val loss | epoch | 일반화 / overfit 진단 |
| metric (accuracy / F1 / AUC) | val epoch | task 별 |
| learning rate | step | scheduler 동작 확인 |
| gradient norm | step | exploding / vanishing 진단 |
| GPU 사용률 | nvidia-smi / wandb | 효율 |
| epoch 시간 | epoch | 병목 진단 |

### 9.1 wandb / tensorboard

```python
import wandb

wandb.init(project="my-project", config={
    "lr": 1e-3, "batch_size": 64, "model": "ResNet50",
})

for epoch in range(num_epochs):
    train_loss = train_one_epoch(...)
    val_loss, val_acc = evaluate(...)
    wandb.log({
        "epoch": epoch,
        "train_loss": train_loss,
        "val_loss": val_loss,
        "val_acc": val_acc,
        "lr": optimizer.param_groups[0]["lr"],
    })
```

### 9.2 gradient norm 모니터링

```python
total_norm = 0.0
for p in model.parameters():
    if p.grad is not None:
        total_norm += p.grad.data.norm(2).item() ** 2
total_norm = total_norm ** 0.5

wandb.log({"grad_norm": total_norm})
```

| total_norm | 진단 |
| --- | --- |
| 매우 큼 (>1000) | exploding → clip / lr ↓ |
| 매우 작음 (<1e-4) | vanishing → 초기화 / activation / lr ↑ |
| 안정 | OK |

---

## 10. checkpoint 저장 / 로드

### 10.1 저장

```python
torch.save({
    "epoch": epoch,
    "model": model.state_dict(),
    "optimizer": optimizer.state_dict(),
    "scheduler": scheduler.state_dict(),
    "val_loss": val_loss,
    "config": {"lr": 1e-3, "batch_size": 64},
}, "checkpoint.pt")
```

### 10.2 로드

```python
ckpt = torch.load("checkpoint.pt", map_location=device)
model.load_state_dict(ckpt["model"])
optimizer.load_state_dict(ckpt["optimizer"])
scheduler.load_state_dict(ckpt["scheduler"])
start_epoch = ckpt["epoch"] + 1
```

### 10.3 inference 만 (가벼움)

```python
torch.save(model.state_dict(), "model_only.pt")

# 로드
model = MyModel()
model.load_state_dict(torch.load("model_only.pt"))
model.eval()
```

→ optimizer state 없음. resume 불가, inference 만.

### 10.4 best 모델 + early stopping

```python
class EarlyStopper:
    def __init__(self, patience=5):
        self.patience = patience
        self.best = float("inf")
        self.counter = 0
        self.stop = False

    def step(self, val_loss):
        if val_loss < self.best:
            self.best = val_loss
            self.counter = 0
        else:
            self.counter += 1
            if self.counter >= self.patience:
                self.stop = True

stopper = EarlyStopper(patience=10)
for epoch in range(num_epochs):
    ...
    stopper.step(val_loss)
    if stopper.stop:
        print(f"Early stopping at epoch {epoch}")
        break
```

---

## 11. GPU 효율 — mixed precision / gradient accumulation

### 11.1 mixed precision (fp16 / bf16)

```python
from torch.amp import GradScaler, autocast

scaler = GradScaler()

for x, y in loader:
    x, y = x.to(device, non_blocking=True), y.to(device, non_blocking=True)
    optimizer.zero_grad(set_to_none=True)

    with autocast(device_type='cuda', dtype=torch.float16):
        logits = model(x)
        loss = criterion(logits, y)

    scaler.scale(loss).backward()
    scaler.unscale_(optimizer)
    torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
    scaler.step(optimizer)
    scaler.update()
```

| dtype | 특성 |
| --- | --- |
| fp32 | 표준 — 안전, 메모리/속도 보통 |
| fp16 | 메모리 절반 / 속도 2배, underflow 위험 → loss scaling 필수 |
| bf16 | 메모리 절반 / 속도 2배, range = fp32 → loss scaling 불필요 (A100 / TPU) |

### 11.2 gradient accumulation — 가상 batch 크기 증가

GPU 메모리 부족 시 작은 batch 를 누적해서 큰 batch 효과.

```python
accumulation_steps = 4    # 가상 batch = batch_size * 4

optimizer.zero_grad(set_to_none=True)
for i, (x, y) in enumerate(loader):
    x, y = x.to(device), y.to(device)

    with autocast(device_type='cuda'):
        logits = model(x)
        loss = criterion(logits, y) / accumulation_steps   # ★ 평균 위해 나눔

    scaler.scale(loss).backward()

    if (i + 1) % accumulation_steps == 0:
        scaler.step(optimizer)
        scaler.update()
        optimizer.zero_grad(set_to_none=True)
```

---

## 12. 흔한 오류와 디버깅

### 12.1 device mismatch

```
RuntimeError: Expected all tensors to be on the same device, but found at least two devices, cuda:0 and cpu!
```

| 원인 | 정정 |
| --- | --- |
| Model 은 `.to(device)`, 입력은 안 함 | `x = x.to(device)` |
| `nn.Module` 안 직접 만든 tensor | `register_buffer` 사용 또는 `to` 호출 |

### 12.2 CUDA out of memory (OOM)

| 정정 (우선순위) | 효과 |
| --- | --- |
| `batch_size` 감소 | 즉시 효과 |
| gradient accumulation | 가상 큰 batch 유지 |
| mixed precision (fp16/bf16) | 메모리 절반 |
| gradient checkpointing | activation 메모리 절약 |
| `torch.cuda.empty_cache()` | 흔히 효과 없음 (PyTorch 자체 cache) |
| 더 작은 모델 / sequence | 마지막 수단 |

### 12.3 NaN loss

| 원인 | 정정 |
| --- | --- |
| lr 너무 큼 | lr 감소 |
| 데이터에 NaN / Inf | `torch.isnan(x).any()` 검사 |
| log(0) / sqrt(neg) | epsilon 추가 / clamp |
| fp16 underflow | bf16 로 전환 또는 loss scaling 확인 |
| gradient explode | clip_grad_norm |

### 12.4 데이터 로더 정체 (GPU 0% 사용률 자주)

| 원인 | 정정 |
| --- | --- |
| `num_workers=0` | `num_workers=4 ~ 8` |
| 디스크 IO 병목 | SSD / NVMe / 데이터 캐싱 |
| 복잡한 transform | augmentation 단순화 / 사전 처리 |
| `pin_memory=False` | True |

### 12.5 결과가 재현 안 됨

```python
import torch, random, numpy as np

def set_seed(seed):
    random.seed(seed)
    np.random.seed(seed)
    torch.manual_seed(seed)
    torch.cuda.manual_seed_all(seed)
    torch.backends.cudnn.deterministic = True
    torch.backends.cudnn.benchmark = False
```

→ 완전 재현은 cudnn 의 non-deterministic operation 으로 제한 — 약간의 차이는 남음.

---

## 13. inference 전환

학습 끝난 모델을 production 으로.

### 13.1 PyTorch native

```python
model = MyModel()
model.load_state_dict(torch.load("model.pt"))
model.eval()
model.to(device)

with torch.inference_mode():
    pred = model(x.to(device))
```

### 13.2 TorchScript (배포)

```python
scripted = torch.jit.script(model)
scripted.save("model.ts")

# 로드 (Python / C++ 모두 가능)
loaded = torch.jit.load("model.ts")
```

### 13.3 ONNX (frame work 무관)

```python
torch.onnx.export(
    model, dummy_input, "model.onnx",
    input_names=["input"], output_names=["output"],
    dynamic_axes={"input": {0: "batch"}, "output": {0: "batch"}},
    opset_version=17,
)
```

→ ONNXRuntime / TensorRT / OpenVINO 등 어디서나 추론.

---

## 14. 학습 효율 점검 체크 항목

| 항목 | 점검 |
| --- | --- |
| `model.train()` / `model.eval()` 적절히 전환 |  |
| `optimizer.zero_grad()` 매 step (또는 accumulation 끝) |  |
| `loss.backward()` 후 `optimizer.step()` |  |
| validation 시 `@torch.no_grad()` |  |
| device 전송 (`.to(device)`) 일관 |  |
| `pin_memory=True` + `num_workers > 0` (GPU 시) |  |
| gradient clipping (RNN / Transformer) |  |
| mixed precision (큰 모델) |  |
| checkpoint 정기 저장 |  |
| seed 고정 (디버깅 / 비교) |  |

---

## 15. 참고

- [[deep-learning|↑ deep-learning hub]]
- [[concepts]]
- [[backpropagation]]
- [[../machine-learning/loss-and-gradient-descent]]
- [[../machine-learning/training-and-evaluation]]
- [[../foundations/data-analysis]]
- [[../foundations/data-visualization]]
- PyTorch tutorial — https://pytorch.org/tutorials/beginner/basics/intro.html
- PyTorch amp docs — https://pytorch.org/docs/stable/amp.html
- "Deep Learning with PyTorch" (Eli Stevens, 2020) — 무료 PDF
- Hugging Face course — https://huggingface.co/course
