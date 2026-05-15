---
title: "ai — Claude Code 같은 LLM coding agent 직접 만들기 학습 hub"
kind: knowledge
project: ai
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T04:00:00+09:00
tags: [area, index, ai, llm, learning-path, hub]
---

# ai — LLM coding agent 직접 만들기 학습 hub

| 문서 버전 | 작성일 | 주요 변경 사항 |
| --- | --- | --- |
| v.1.0.0 | 2026-05-13 | 최초 — 운영 관점의 11 폴더 골격 |
| v.2.0.0 | 2026-05-15 | "Claude Code 류 직접 만들기" 학습 로드맵 + foundations / ML / DL / transformer / training / inference / coding-agent / infra / datasets / benchmarks 추가 |

**[[../areas|↑ 20-areas]]**

> **목표**: Claude Code / Cursor / Cline 같은 LLM coding agent 를 직접 만들 수 있는 수준의 이해.
> **접근**: 운영 (LLM API 사용) 만이 아닌 **밑바닥부터 (모델 / 학습 / 추론 / 에이전트 시스템)** 의 학습 로드맵.

## 1. 학습 로드맵 — 5 Phase

### Phase 1 — Foundations (수학 + 코드)
LLM 의 모든 것은 행렬 곱 + 미분 + Python.

1. **[[foundations/foundations]]** — math (linear algebra, probability, calculus) + python ML stack
2. **[[machine-learning/machine-learning]]** — supervised / loss / gradient descent / overfitting

### Phase 2 — Deep Learning + NLP
신경망의 동작 원리 + 텍스트의 표현.

3. **[[deep-learning/deep-learning]]** — neural network / backprop / CNN / RNN
4. **[[_common/_common]]** — NLP 기초 (tokenization / embeddings)
5. **[[transformer-internals/transformer-internals]]** — Attention is All You Need + decoder-only architecture

### Phase 3 — LLM 학습
모델을 직접 학습 / 파인튜닝.

6. **[[training-pretraining/training-pretraining]]** — pretraining objective / scaling law / Chinchilla
7. **[[fine-tuning/fine-tuning]]** — SFT / LoRA / QLoRA / instruction tuning / RLHF / DPO
8. **[[datasets/datasets]]** — 데이터 수집 / cleaning / labeling / synthetic data
9. **[[infrastructure/infrastructure]]** — GPU / 분산 학습 (FSDP / DeepSpeed) / TPU

### Phase 4 — 추론 + 운영
학습한 모델을 서비스로.

10. **[[inference/inference]]** — sampling (temperature/top-k/top-p) / KV cache / quantization / speculative decoding / vLLM / TGI
11. **[[llm/llm]]** — Claude / GPT / Gemini / Llama / Mistral / Qwen 모델별 특성
12. **[[prompt-engineering/prompt-engineering]]** — CoT / few-shot / XML tags / system prompt
13. **[[rag/rag]]** — chunking / embedding / re-ranking / contextual retrieval
14. **[[vector-db/vector-db]]** — Pinecone / Weaviate / Chroma / Qdrant / pgvector
15. **[[mlops/mlops]]** — 서빙 / 비용 / 관측 / multi-provider gateway

### Phase 5 — Coding Agent
LLM 을 코딩 에이전트로.

16. **[[agents/agents]]** — single / multi-agent / planning / ReAct
17. **[[coding-agent-internals/coding-agent-internals]]** — Claude Code / Cursor / Cline 의 동작 원리 (tool use / repo nav / diff / sandbox / iterate)
18. **[[evaluation/evaluation]]** — HumanEval / MBPP / SWE-bench / LLM-as-judge
19. **[[benchmarks/benchmarks]]** — 코딩 / 추론 / 안전 벤치마크
20. **[[safety-alignment/safety-alignment]]** — RLHF / Constitutional AI / red-team / prompt injection
21. **[[papers/papers]]** — 핵심 논문 (Attention / GPT / InstructGPT / Llama / SWE-agent / 등)

## 2. 영역 표 (folder 별)

| 카테고리 | 진입점 | 단계 |
| --- | --- | --- |
| **Foundations** | [[foundations/foundations]] | Phase 1 |
| ML 기초 | [[machine-learning/machine-learning]] | Phase 1 |
| DL 기초 | [[deep-learning/deep-learning]] | Phase 2 |
| NLP 공통 | [[_common/_common]] | Phase 2 |
| Transformer | [[transformer-internals/transformer-internals]] | Phase 2 |
| Pretraining | [[training-pretraining/training-pretraining]] | Phase 3 |
| Fine-tuning | [[fine-tuning/fine-tuning]] | Phase 3 |
| 데이터셋 | [[datasets/datasets]] | Phase 3 |
| 인프라 | [[infrastructure/infrastructure]] | Phase 3 |
| Inference | [[inference/inference]] | Phase 4 |
| LLM 모델 | [[llm/llm]] | Phase 4 |
| Prompt | [[prompt-engineering/prompt-engineering]] | Phase 4 |
| RAG | [[rag/rag]] | Phase 4 |
| Vector DB | [[vector-db/vector-db]] | Phase 4 |
| LLMOps | [[mlops/mlops]] | Phase 4 |
| Agents | [[agents/agents]] | Phase 5 |
| **Coding Agent** | [[coding-agent-internals/coding-agent-internals]] | Phase 5 |
| Evaluation | [[evaluation/evaluation]] | Phase 5 |
| 벤치마크 | [[benchmarks/benchmarks]] | Phase 5 |
| 안전 / 정렬 | [[safety-alignment/safety-alignment]] | Phase 5 |
| 논문 | [[papers/papers]] | 보조 |

## 3. "Claude Code 같은 거" 의 정의

목표를 명확히 — Claude Code 같은 coding agent 는 다음 4 가지 결합:

1. **강력한 base LLM** — coding-trained foundation model (Claude / GPT-5 / DeepSeek-Coder / Qwen-Coder 등).
2. **Tool use** — file edit / read / shell / search 등의 native function calling.
3. **에이전트 루프** — plan → tool call → observation → next step. iterative.
4. **CLI / IDE 통합** — terminal UI + 로컬 파일시스템 + git.

→ 즉 학습 영역 = (base LLM 자체) + (tool use 시스템) + (agent runtime) + (인터페이스).

내가 처음부터 다 만들기는 비현실적이지만 **각 부분의 동작 원리 + 일부 본인 구현 가능 / 일부 기존 모델 fine-tune** 의 조합으로 가능.

## 4. 현실적 단계 — "직접 만들기" 의 의미

| 단계 | 의미 | 난이도 |
| --- | --- | --- |
| **L1 — Wrapper** | OpenAI / Anthropic API + tool calling | 쉬움 |
| **L2 — Agent Framework** | LangChain / DSPy 위에 자체 에이전트 | 보통 |
| **L3 — 자체 Fine-tune** | Llama / Qwen-Coder + 도메인 fine-tune | 보통 |
| **L4 — 자체 Pretraining** | 작은 LLM 직접 pretrain (1~7B) | 어려움 + 비용 |
| **L5 — 새 아키텍처 / 자체 모델** | 학술 연구 영역 | 매우 어려움 |

→ 대부분 **L1~L3** 에서 의미 있는 결과. L4 는 학습용 (작은 모델). L5 는 별도 학계.

## 5. 외부 자료 — 학습 순서대로

### Phase 1 (Foundations)
- [3Blue1Brown — Linear Algebra / Calculus](https://www.3blue1brown.com/)
- [Karpathy — Neural Networks: Zero to Hero](https://karpathy.ai/zero-to-hero.html) — 필독.

### Phase 2 (DL + Transformer)
- [Attention Is All You Need (2017)](https://arxiv.org/abs/1706.03762) — Transformer 원논문.
- [Jay Alammar — The Illustrated Transformer](http://jalammar.github.io/illustrated-transformer/)
- [Karpathy — nanoGPT](https://github.com/karpathy/nanoGPT) — GPT 직접 구현.
- [Karpathy — Let's build GPT (4시간 영상)](https://www.youtube.com/watch?v=kCc8FmEb1nY)

### Phase 3 (LLM 학습)
- [Stanford CS324: Large Language Models](https://stanford-cs324.github.io/winter2022/)
- [InstructGPT (2022)](https://arxiv.org/abs/2203.02155) — RLHF.
- [LLaMA / Llama 2 / Llama 3](https://ai.meta.com/blog/) papers.
- [DPO (2023)](https://arxiv.org/abs/2305.18290) — RLHF 의 단순 대안.
- [HuggingFace — Trainer / TRL](https://huggingface.co/docs)

### Phase 4 (추론)
- [vLLM](https://docs.vllm.ai/) / [TGI](https://huggingface.co/docs/text-generation-inference)
- [llama.cpp](https://github.com/ggerganov/llama.cpp) — local CPU/GPU.

### Phase 5 (Agent + Coding)
- [Anthropic — Building effective agents (2024)](https://www.anthropic.com/research/building-effective-agents)
- [SWE-agent (2024)](https://arxiv.org/abs/2405.15793)
- [ReAct (2022)](https://arxiv.org/abs/2210.03629)
- [SWE-bench](https://www.swebench.com/)
- [Claude Code 의 시스템 프롬프트 / 공식 문서](https://docs.anthropic.com/en/docs/claude-code)
- [Cursor / Cline / aider 의 오픈 코드](https://github.com/cline/cline) — 직접 본다.

## 6. 추천 학습 순서 (현실적 ~12개월 plan)

```
Month 1-2:  Phase 1 — math + python + numpy/pytorch
Month 3-4:  Phase 2 — neural net + transformer (Karpathy 영상 따라 nanoGPT 직접 구현)
Month 5-6:  Phase 3 — fine-tuning Llama 3 8B (LoRA, instruction tuning)
Month 7-8:  Phase 4 — vLLM 로컬 서빙 + RAG 구축
Month 9-10: Phase 5 — agent framework + tool use 자체 구현
Month 11-12: 자체 coding agent prototype (Llama + tool use + repo)
```

→ 욕심내지 말기. Karpathy 의 nanoGPT 따라하기 1번이 큰 시작.

## 7. 관련

- [[../areas|↑ 20-areas]]
- [[../../00-inbox/links/links|↗ 외부 reference 카탈로그]]
- [[../ai-agent/ai-agent]] — 우리 yule-studio-agent 의 에이전트 시스템 (운영 측)
