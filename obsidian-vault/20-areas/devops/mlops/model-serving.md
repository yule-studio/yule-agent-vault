---
title: "Model serving — TorchServe / Triton / vLLM"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T10:57:00+09:00
tags: [devops, mlops, serving]
---

# Model serving — TorchServe / Triton / vLLM

**[[mlops|↑ mlops]]**

---

## 1. 종류

```
batch inference:
  - 정기 (일별 / 시간별)
  - 큰 volume, latency 무관
  - Spark / Ray / Dask
  - airflow 로 schedule

online serving:
  - 실시간 (10-100ms)
  - REST / gRPC API
  - autoscaling
  - GPU / CPU

streaming:
  - Kafka / Flink 에서 inference
  - low latency
```

---

## 2. 도구 비교

| | 종류 | 강점 |
| --- | --- | --- |
| **TorchServe** | PyTorch | PyTorch native |
| **TF Serving** | TensorFlow | TF native, 빠름 |
| **NVIDIA Triton** | 다 framework | GPU 최적, dynamic batching |
| **KServe** | k8s | autoscale + canary |
| **Seldon Core** | k8s | A/B, explanation |
| **BentoML** | flexible | Python 친화 |
| **vLLM** (★) | LLM | PagedAttention, throughput 최고 |
| **TGI** (HuggingFace) | LLM | 쉬움 |
| **Modal / Replicate** | SaaS | serverless GPU |
| **Sagemaker** | AWS SaaS | managed |
| **Ray Serve** | scale | distributed |

→ classical ML = KServe / Triton. LLM = vLLM / TGI.

---

## 3. vLLM 예 (LLM ★)

```bash
# 설치
pip install vllm

# 서버 시작
python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Llama-3.1-8B-Instruct \
    --port 8000 \
    --max-model-len 4096 \
    --tensor-parallel-size 2 \
    --gpu-memory-utilization 0.95
```

```python
# client (OpenAI 호환)
from openai import OpenAI
client = OpenAI(base_url="http://localhost:8000/v1", api_key="dummy")

response = client.chat.completions.create(
    model="meta-llama/Llama-3.1-8B-Instruct",
    messages=[{"role": "user", "content": "Hello"}]
)
```

특징:
- **PagedAttention** — KV cache 효율 ↑
- **continuous batching** — 다른 길이 요청 동시 처리
- **OpenAI API 호환** — drop-in replacement
- **multi-GPU** (tensor parallel)

→ 일반 inference 보다 5-10x throughput.

---

## 4. Triton (★ classical ML 최강)

```
# config.pbtxt
name: "resnet50"
platform: "tensorrt_plan"
max_batch_size: 32

input [
  { name: "input", data_type: TYPE_FP32, dims: [3, 224, 224] }
]
output [
  { name: "output", data_type: TYPE_FP32, dims: [1000] }
]

dynamic_batching {
    preferred_batch_size: [4, 8, 16]
    max_queue_delay_microseconds: 100
}

instance_group [
  { count: 2, kind: KIND_GPU }
]
```

```bash
docker run --gpus all -p 8000:8000 -p 8001:8001 -p 8002:8002 \
    -v /models:/models \
    nvcr.io/nvidia/tritonserver:24.05-py3 \
    tritonserver --model-repository=/models
```

기능:
- multi-framework (TF/PyTorch/ONNX/TensorRT)
- dynamic batching
- ensemble (model 들 chain)
- model versioning
- gRPC + HTTP

---

## 5. KServe (★ k8s 표준)

```yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata: {name: my-model, namespace: ml}
spec:
  predictor:
    minReplicas: 1
    maxReplicas: 10
    scaleTarget: 10        # 10 req/s 마다 1 pod
    scaleMetric: rps
    
    model:
      modelFormat: {name: pytorch}
      storageUri: "s3://my-bucket/models/v1"
      resources:
        limits: {nvidia.com/gpu: 1, memory: 16Gi}

  # canary
  canaryTrafficPercent: 20
  
  # explainer (옵션)
  explainer:
    alibi:
      type: AnchorTabular
      storageUri: "s3://.../explainer"
```

→ k8s native + autoscale + canary 자동.

---

## 6. GPU on k8s

```yaml
# NVIDIA GPU Operator install
helm install gpu-operator nvidia/gpu-operator \
    -n gpu-operator --create-namespace

# pod 에서 GPU 사용
apiVersion: v1
kind: Pod
metadata: {name: inference}
spec:
  containers:
    - name: serve
      image: my-model:1.0
      resources:
        limits:
          nvidia.com/gpu: 1     # 1 GPU
          memory: 16Gi
  tolerations:
    - key: nvidia.com/gpu
      operator: Exists
      effect: NoSchedule
```

→ GPU Operator 가 driver / runtime / device plugin 자동.

---

## 7. autoscaling (★)

```
HPA: CPU / memory — GPU 에 부적합
KEDA: custom metric (queue depth / RPS)
KServe: 자동 (rps target)

LLM 의 특수:
  - cold start 매우 큼 (model load 30s-5min)
  - scale to 0 = first request 매우 느림
  - 항상 min replica 권장 (warm pool)
```

```yaml
# KEDA + GPU
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata: {name: inference}
spec:
  scaleTargetRef: {name: inference}
  minReplicaCount: 1            # warm
  maxReplicaCount: 20
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prom:9090
        metricName: pending_requests
        threshold: "10"
        query: sum(pending_requests{service="inference"})
```

---

## 8. quantization (★ cost 줄이기)

```
FP32 → FP16 → INT8 → INT4 → ...

memory 절반 / 4분의 1
inference 빠름
정확도 약간 ↓ (acceptable)

도구:
  - TensorRT (NVIDIA)
  - ONNX Runtime
  - GPTQ (LLM)
  - AWQ (LLM)
  - bitsandbytes (LLM)
```

```python
# bitsandbytes 4-bit
from transformers import AutoModelForCausalLM, BitsAndBytesConfig

bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_compute_dtype=torch.float16,
    bnb_4bit_quant_type="nf4",
    bnb_4bit_use_double_quant=True
)
model = AutoModelForCausalLM.from_pretrained(
    "meta-llama/Llama-3.1-70B-Instruct",
    quantization_config=bnb_config
)
# 70B model 이 40GB → 35GB → 24GB
```

---

## 9. batching 전략

```
1. static batching
   - N req 모이면 처리
   - latency 변동 큼

2. dynamic batching (★ Triton)
   - max_delay 안에서 자동 batch
   - latency / throughput balance

3. continuous batching (★ vLLM)
   - 다른 길이의 LLM 요청 동시
   - 가장 효율 (LLM)
```

---

## 10. model load

```
큰 model (70B):
  - 30s-5min load
  - 매번 load → cold start 폭주
  
해결:
  - 한 번 load + 항상 메모리 (warm pool)
  - shared volume (PV) 의 model 캐시
  - init container 가 model download → emptyDir
  - tensor parallelism 여러 GPU 에 split
```

---

## 11. monitoring (★)

```
metric:
  - latency p50/p95/p99
  - throughput (RPS)
  - error rate
  - queue depth
  - GPU utilization
  - GPU memory
  - VRAM 사용
  - tokens / second (LLM)
  
quality:
  - prediction distribution drift
  - data drift (input feature)
  - label feedback (online learning)
  - human eval
```

---

## 12. A/B / canary

```
방법:

A. shadow
   prod traffic → 새 model 도 받음 (응답 안 함)
   결과 로깅 → 비교

B. canary
   10% traffic → 새 model
   90% → 기존
   metric OK → 100%

C. champion-challenger
   champion (current) vs N challenger
   주기적으로 swap

D. multi-armed bandit
   ML 이 자동 traffic 분배
```

---

## 13. 함정

1. **GPU 비쌈 — 24/7 max replica** — autoscale 검토.
2. **cold start LLM** — min replica.
3. **model 의 input 검증 없음** — adversarial input.
4. **monitoring 없음** — silent drift.
5. **version 관리 없음** — rollback 불가.
6. **batching 안 함** — throughput 매우 낮음.
7. **CPU inference 만 검토 X** — 작은 model 은 CPU 도 충분.
8. **memory leak** — GPU memory 누적 → OOM.

---

## 14. 관련

- [[mlops|↑ mlops]]
- [[gpu-cluster]]
- [[llm-ops]]
- [[../finops/kubernetes-cost|↗ GPU cost]]
