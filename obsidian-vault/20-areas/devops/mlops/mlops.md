---
title: "MLOps — ML 모델 배포 / serving / 모니터링 ★"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T10:55:00+09:00
tags: [area, devops, mlops]
---

# MLOps — ML 모델 배포 / serving / 모니터링 ★

**[[../devops|↑ devops]]**

---

## 1. 왜 MLOps

```
ML model = 단순 코드 + data + weights + 학습 환경.

기존 DevOps 와 다름:
  - data 도 version (training set 변경 시 다른 model)
  - reproducibility (같은 학습 = 같은 model?)
  - model 의 drift (시간 지나면 정확도 ↓)
  - resource (GPU) 비싸 / 부족
  - explainability / 공정성
  - A/B test 의 metric 이 business 와 ML
```

→ 시니어 = data 팀 협업 + serve 인프라 + 모니터링.

---

## 2. 4 phase

```
1. Data
   - 수집 / 정제 / version (DVC, LakeFS)
   - feature store (Feast / Tecton)

2. Train
   - experiment tracking (MLflow / W&B)
   - hyperparameter tuning (Optuna, Ray Tune)
   - distributed training (Horovod, DeepSpeed)
   - GPU cluster 관리

3. Serve
   - batch inference (Spark / Airflow)
   - online serving (TorchServe / TF Serving / Triton / vLLM)
   - autoscale (KEDA, GPU)
   - canary

4. Monitor
   - latency
   - error rate
   - prediction drift
   - data drift
   - model quality (label feedback)
   - cost
```

---

## 3. 하위 영역

- [[model-serving]] — TorchServe / TF Serving / Triton / vLLM
- [[ml-pipeline]] — Kubeflow / Argo Workflow / Airflow
- [[feature-store]] — Feast / Tecton
- [[experiment-tracking]] — MLflow / W&B / Neptune
- [[model-registry]] — MLflow Models / Sagemaker
- [[ab-test-models]] — shadow / canary / champion-challenger
- [[gpu-cluster]] — k8s + GPU operator / Slurm
- [[llm-ops]] — LLM 특수 (vLLM / TGI / quantization / RAG)
- [[model-monitoring]] — drift detection / explainability
- [[mlflow]] — Tracking + Models + Projects
- [[data-versioning]] — DVC / LakeFS
- [[pitfalls]]

---

## 4. 도구 stack

```
data:
  S3 / GCS / Azure Blob
  Iceberg / Delta Lake (table format)
  Spark / Trino (query)
  Airflow / Dagster / Prefect (orchestrate)

training:
  PyTorch / TensorFlow / JAX
  HuggingFace
  Ray (distributed)
  Kubeflow training operator
  W&B / MLflow (tracking)

serving:
  TorchServe / TF Serving / Triton
  vLLM (LLM 특화)
  Seldon Core / KServe
  BentoML

monitoring:
  Evidently
  Arize
  WhyLabs
  Fiddler
```

→ stack 큼. 회사 별 선택.

---

## 5. 시니어 결정 단골

```
"GPU 비싸. 어떻게 줄임?"
  → Spot GPU
  → quantization (INT8 / FP8)
  → batching (TF Serving batch)
  → distillation (작은 model)
  → serverless GPU (Modal / RunPod)

"model 정확도 ↓"
  → data drift?
  → concept drift?
  → seasonality?
  → retrain 빈도

"serving latency 큼"
  → batch size 조정
  → KV cache (LLM)
  → CPU inference 도 검토
  → ONNX / TensorRT 변환
```

---

## 6. learn 순서

1. Day 1: [[model-serving]] (TorchServe 또는 vLLM)
2. Day 2: [[ml-pipeline]] (Airflow / Kubeflow)
3. Day 3: [[mlflow]] (tracking + registry)
4. Day 4: [[gpu-cluster]] (k8s GPU)
5. Day 5: [[model-monitoring]] + [[llm-ops]]

---

## 7. 관련

- [[../devops|↑ devops]]
- [[../kubernetes/kubernetes|↗ k8s]]
- [[../performance/performance|↗ performance]]
- [[../finops/finops|↗ FinOps (GPU cost)]]
