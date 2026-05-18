---
title: "ai-agent — LLM 에이전트 운영 영역 Hub"
kind: knowledge
project: ai-agent
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-19T00:50:00+09:00
tags: [area, ai-agent, llm, hub]
home_hub: areas
related:
  - "[[../areas]]"
  - "[[../ai/ai]]"
  - "[[../ai/llm/llm]]"
  - "[[../ai/rag/rag]]"
---

# ai-agent — LLM 에이전트 운영 영역 Hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-19 | engineering-agent/tech-lead | 최초 — 6 영역 통합 hub |

**[[../areas|↑ 20-areas]]**

> 본 영역은 LLM 을 사용해 자율적으로 작업을 수행하는 **에이전트 시스템의 운영** 을 다룬다. LLM 자체의 학습 / 구조 / 평가는 [[../ai/ai|↑ ai]] 영역.

---

## 1. 영역 정의

본 영역은 LLM 을 호출 / 도구 사용 / 메모리 / 다중 에이전트 협업으로 조합해 자율적 작업 수행 시스템을 만드는 패턴의 인덱스다.

본 영역이 정의하는 것:
- 6 영역의 진입점 (agent runtime / llm 호출 / multi-agent / prompt-eng / RAG / tool calling)
- 영역 간 책임 분리
- 학습 / 적용 진입점
- 인접 영역 (ai / backend / devops) 과의 cross-link

본 영역이 정의하지 않는 것:
- LLM 학습 / 구조 / 평가 — [[../ai/ai]]
- vector DB 자체 — [[../ai/vector-db/]]
- 에이전트 host 인프라 — [[../devops/devops]]

---

## 2. 영역 분류

| 디렉토리 | 다루는 주제 |
| --- | --- |
| [[agent-runtime/|agent-runtime]] | 에이전트의 실행 loop / 상태 / 메모리 / lifecycle / 중단·재개 |
| [[llm/|llm]] | LLM provider 호출 (OpenAI / Anthropic / Bedrock / Gemini / 로컬 LLM) + 토큰 / 비용 / 캐시 |
| [[multi-agent/|multi-agent]] | 다중 에이전트 협업 / 역할 분리 / 메시지 전달 / 합의 |
| [[prompt-engineering/|prompt-engineering]] | system prompt / few-shot / chain-of-thought / 구조화된 출력 |
| [[rag/|rag]] | retrieval-augmented generation — 문서 인덱싱 / 검색 / 컨텍스트 주입 |
| [[tool-calling/|tool-calling]] | function calling / 도구 정의 / 실행 / 안전성 |

---

## 3. 학습 진입점 권장

| 목적 | 진입점 |
| --- | --- |
| LLM 처음 호출 | [[llm/|llm]] → [[prompt-engineering/|prompt-engineering]] |
| 도구 사용 에이전트 작성 | [[tool-calling/|tool-calling]] → [[agent-runtime/|agent-runtime]] |
| 문서 기반 QA / 챗봇 | [[rag/|rag]] + [[llm/|llm]] |
| 자율 작업 에이전트 (Claude Code 류) | [[agent-runtime/|agent-runtime]] → [[tool-calling/|tool-calling]] → [[multi-agent/|multi-agent]] |
| 다중 에이전트 협업 | [[multi-agent/|multi-agent]] → [[../ai/safety-alignment/]] |
| 비용 / 토큰 절감 | [[llm/|llm]] + [[../ai/llm/llm]] |
| LLM 자체 학습 / 구조 | [[../ai/ai|↑ ai]] (별도 영역) |

---

## 4. 영역 간 책임 분리

| 본 영역 | 인접 영역 | 분리 기준 |
| --- | --- | --- |
| LLM provider 호출 (운영 관점) | LLM 학습 / 구조 → [[../ai/llm/llm]] | 사용 vs 원리 |
| RAG 의 검색 / 컨텍스트 주입 | vector DB 자체 → [[../ai/vector-db/]] | 응용 vs 저장 |
| 에이전트 runtime | 에이전트 host 인프라 → [[../devops/devops]] | 응용 vs 인프라 |
| tool calling 의 도구 정의 | 도구 자체 구현 (HTTP API / SQL / shell) | 인터페이스 vs 구현 |
| prompt 작성 | LLM 의 학습 결과 / 한계 → [[../ai/foundations/]] | 적용 vs 이해 |

---

## 5. 핵심 결정 — 새 에이전트 프로젝트 시작

| 결정 | 옵션 |
| --- | --- |
| LLM provider | OpenAI / Anthropic / Bedrock / Gemini / 로컬 (Ollama / vLLM) |
| 에이전트 프레임워크 | LangChain / LangGraph / LlamaIndex / Anthropic SDK 직접 / 자체 구현 |
| 메모리 | conversation buffer / vector memory / SQL state |
| 도구 정의 형식 | OpenAI function calling / Anthropic tool use / MCP |
| RAG 인덱싱 | chunk size / overlap / embedding model |
| vector store | pgvector / Pinecone / Qdrant / Weaviate / Chroma |
| 평가 | hand-curated set / LLM-as-judge / RAGAS |
| 모니터링 | LangSmith / Helicone / 자체 logging |

---

## 6. 본 영역의 유지보수

| 트리거 | 갱신 항목 |
| --- | --- |
| 새 패턴 / framework 등장 | §2 + 신규 디렉토리 |
| 새 LLM provider 진입 | §5 |
| 인접 영역과 책임 변경 | §4 |

---

## 7. 관련

- [[../areas|↑ 20-areas]]
- [[agent-runtime/|agent-runtime]]
- [[llm/|llm]]
- [[multi-agent/|multi-agent]]
- [[prompt-engineering/|prompt-engineering]]
- [[rag/|rag]]
- [[tool-calling/|tool-calling]]
- [[../ai/ai|↑ ai (LLM 자체)]]
- [[../ai/llm/llm|↗ LLM 학습 / 구조]]
- [[../ai/vector-db/]]
- [[../ai/safety-alignment/]]
- [[../backend/backend|↗ backend hub]]
- [[../devops/devops|↗ devops hub]]
