---
title: "ai — AI / LLM / RAG / Agent 운영"
kind: knowledge
project: ai
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T04:00:00+09:00
tags:
  - area
  - index
  - ai
---

# ai — AI / LLM / RAG / Agent 운영

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 최초 — 폴더링 골격 |

**[[../areas|↑ 20-areas]]**

> AI / 머신러닝 / LLM 시스템 운영. 기존 `ai-agent/` 는 보존 (Yule 에이전트 특화).

## 하위 영역

| 주제 | 진입점 | 한 줄 |
| --- | --- | --- |
| AI 공통 | [[_common/_common]] | NLP / tokenization / embeddings |
| LLM 모델 | [[llm/llm]] | Claude / GPT / Gemini / Llama / Mistral / Qwen / local |
| 프롬프트 엔지니어링 | [[prompt-engineering/prompt-engineering]] | CoT / few-shot / XML tags / 인젝션 방어 |
| RAG | [[rag/rag]] | chunking / embedding / re-ranking / contextual retrieval / GraphRAG |
| AI Agents | [[agents/agents]] | single / multi / tool use / memory / 프레임워크 |
| 벡터 DB | [[vector-db/vector-db]] | Pinecone / Weaviate / Chroma / Qdrant / pgvector |
| Fine-tuning | [[fine-tuning/fine-tuning]] | LoRA / QLoRA / instruction / RLHF |
| LLMOps | [[mlops/mlops]] | 서빙 / 비용 / 관측 / multi-provider gateway |
| 평가 | [[evaluation/evaluation]] | LLM-as-judge / 골든셋 / hallucination / Ragas / 벤치마크 |
| 안전 / 정렬 | [[safety-alignment/safety-alignment]] | Constitutional AI / red-team / jailbreak / OWASP LLM |
| 논문 | [[papers/papers]] | 핵심 논문 요약 (Attention / GPT / RLHF / 등) |

## 관련
- [[../areas|↑ 20-areas]]
- [[../00-inbox/links/links|↗ 외부 reference 카탈로그]]