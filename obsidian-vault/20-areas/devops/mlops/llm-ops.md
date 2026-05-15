---
title: "LLM Ops — RAG / vLLM / quantization / cost"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T10:59:00+09:00
tags: [devops, mlops, llm]
---

# LLM Ops — RAG / vLLM / quantization / cost

**[[mlops|↑ mlops]]**

---

## 1. LLM 의 특수성

```
일반 ML model:
  small (< 1GB), 빠른 inference, low memory.

LLM (7B-70B+):
  - 14GB-140GB+ memory
  - 큰 batch / KV cache
  - GPU 필수 (또는 매우 느림)
  - tokens / second 가 throughput
  - long context (128K+) 의 memory 폭주
  - quantization 큰 효과
```

---

## 2. host vs API

```
A. SaaS API (OpenAI / Anthropic / Gemini / Bedrock)
   + 운영 부담 0
   + 최신 model
   - 비쌈 (긴 prompt × N user)
   - PII / 보안
   - vendor lock-in
   - rate limit

B. self-host (vLLM / TGI / Triton)
   + cost (대규모면 저렴)
   + data 안 나감
   + custom fine-tune
   - GPU infra
   - 운영 부담
   - 최신 model 따라가기

C. hybrid
   - 일반 = SaaS
   - 민감 / 대규모 = self-host
```

→ 시니어 결정: traffic / data 민감 / 비용.

---

## 3. cost 비교 예

```
GPT-4o ($0.005 input / $0.015 output per 1k token):
  1M req × 1k input + 500 output 평균
  = $0.005 × 1M + $0.0075 × 1M = $12,500
  매월

self-host Llama-3.1-70B:
  4x A100 80GB = $32/h × 24 × 30 = $23,040
  유사 throughput
  → 일정 traffic 이상이면 self-host 저렴

break-even = ~30M token/mo
```

---

## 4. RAG (Retrieval-Augmented Generation) ★

```
LLM 은 학습 cutoff + 회사 data 모름.

RAG:
  1. user query
  2. embedding 으로 변환
  3. vector DB 에서 유사 docs 검색
  4. docs + query → LLM 에 prompt
  5. LLM 응답

→ LLM 이 회사 / 최신 data 알도록.
```

```python
# 흐름
query = "회사의 휴가 정책은?"

# 1. embedding
embedding = embed_model.encode(query)

# 2. vector search
docs = vector_db.search(embedding, top_k=5)

# 3. prompt
prompt = f"""
You are a helpful assistant. Answer based on context.

Context:
{docs}

Question: {query}
"""

# 4. LLM
response = llm.generate(prompt)
```

---

## 5. vector DB

| | 무엇 |
| --- | --- |
| **Pinecone** | SaaS, 표준 |
| **Weaviate** | OSS + SaaS, GraphQL |
| **Qdrant** | OSS + SaaS, Rust |
| **Milvus** | OSS, scale |
| **Chroma** | 단순, Python 친화 |
| **pgvector** | Postgres extension (★ 간단) |
| **Elasticsearch dense vector** | 검색 + RAG |
| **OpenSearch k-NN** | AWS |

→ 소규모 = pgvector. 대규모 = Pinecone / Qdrant.

```sql
-- pgvector 예
CREATE EXTENSION vector;

CREATE TABLE docs (
    id BIGSERIAL,
    content TEXT,
    embedding vector(1536)    -- OpenAI 의 dim
);

CREATE INDEX ON docs USING ivfflat (embedding vector_cosine_ops);

-- 검색
SELECT content
FROM docs
ORDER BY embedding <=> '[0.1, 0.2, ...]'::vector
LIMIT 5;
```

---

## 6. quantization (★)

```
70B model 의 size:
  FP32 : 280GB
  FP16 :  140GB
  INT8 :   70GB
  INT4 :   35GB

도구:
  - GPTQ (Hugging Face transformers)
  - AWQ (성능 ↑)
  - bitsandbytes (4-bit)
  - llama.cpp (CPU GGUF)
  - vLLM (자동)

quality:
  4-bit = 거의 같은 quality
  2-bit = 약간 손실
```

→ memory 절감 + 작은 GPU 에 fit.

---

## 7. context length

```
GPT-4: 128K token
Claude: 200K
Gemini: 1M+
Llama-3.1: 128K

긴 context 의 cost:
  - prompt × tokens (input cost)
  - KV cache GPU memory ↑ (linear)
  - 처리 시간 ↑

전략:
  - RAG (관련 docs 만)
  - summary / chunk
  - sliding window
  - hierarchical
```

---

## 8. prompt engineering 운영

```
prompt 도 코드처럼 관리:
  - git version
  - A/B test
  - 평가 (eval)
  - regression test

도구:
  - LangSmith
  - Helicone
  - Weights & Biases Prompts
  - PromptLayer
  - Langfuse
```

---

## 9. agent / function calling

```
LLM 이 도구 호출:
  user: "오늘 서울 날씨?"
  LLM: 도구 호출 → get_weather("Seoul")
  도구: {"temp": 22, "rain": false}
  LLM: "서울은 22도 입니다."

도구:
  - OpenAI function calling
  - LangChain agents
  - LangGraph
  - Anthropic tool use
  - AutoGen (Microsoft)
  - CrewAI
```

→ 단순 generation → 복잡 workflow.

---

## 10. evaluation

```
LLM 의 quality 측정:

방법:
  - human eval (gold standard, 비쌈)
  - LLM-as-judge (GPT-4 가 grade)
  - rule-based (regex / keyword)
  - benchmark (MMLU, HumanEval)
  - 자체 dataset

도구:
  - DeepEval
  - LangSmith eval
  - W&B
  - Promptfoo
  - Arize
```

---

## 11. safety / alignment

```
production LLM 의 risk:
  - PII leak
  - prompt injection
  - jailbreak
  - hallucination
  - 편향 / 차별
  - 저작권

대응:
  - input guard (NeMo Guardrails / Llama Guard)
  - output filter
  - rate limit
  - audit log (모든 입출력)
  - human review (high-risk)
```

---

## 12. caching

```
LLM 응답 cache:
  - same prompt → cached (시간 지나도 OK 면)
  - embedding cache (RAG 의 retrieval)
  - prefix cache (vLLM)

cost ↓ 큰 효과 (cache hit 30-70%).
```

---

## 13. monitoring (LLM 특화)

```
metric:
  - tokens / second (throughput)
  - time to first token (TTFT) (★ UX)
  - latency p99
  - GPU memory / utilization
  - request 의 cost ($)
  - error / refusal rate
  - hallucination rate (eval)
  - prompt 평균 길이
  - cache hit rate
```

---

## 14. infrastructure

```
GPU 옵션:
  A100 80GB:   most flexible
  H100 80GB:   가장 빠름
  L40S 48GB:   inference 적합
  RTX 4090 24GB: dev / 작은 model

cloud:
  AWS p5.48xlarge (8× H100)  $$$/h
  Lambda Labs, RunPod, Modal: 저렴
  CoreWeave, Crusoe: bare-metal GPU

on-prem:
  대규모 → bare-metal H100 cluster
```

---

## 15. 함정

1. **SaaS API 만 의존 — 비용 폭주** — 일정 traffic 이상 self-host 검토.
2. **RAG without chunking strategy** — 큰 doc 가 부적합.
3. **context length 최대 = cost 최대** — RAG 활용.
4. **prompt injection 미검증** — 사용자가 system prompt override.
5. **PII in prompt** — SaaS 에 전송.
6. **eval 없이 prompt 변경** — 회귀.
7. **cold start GPU** — min replica.
8. **fine-tune 보다 RAG / prompt 먼저** — 비용 vs 효과.

---

## 16. 관련

- [[mlops|↑ mlops]]
- [[model-serving]]
- [[../security-ops/security-ops|↗ security PII]]
- [[../finops/kubernetes-cost|↗ GPU cost]]
