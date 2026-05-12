---
title: "LLMOps — 비용 / latency / 캐시 / monitoring"
kind: knowledge
project: agent-references
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-12T20:45:00+09:00
tags:
  - reference
  - links
  - ai
  - llmops
---

# LLMOps — 비용 / latency / 캐시 / monitoring

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 15 링크 |

**[[ai|↑ AI Engineering]]**

> production LLM 운영. 토큰 비용 추적 / latency 최적화 / 모델 라우팅.

## Reference 링크

- [Awesome LLMOps](https://github.com/tensorchord/Awesome-LLMOps) — LLM 운영 자료 인덱스
- [Chip Huyen — Building LLM applications for production](https://huyenchip.com/2023/04/11/llm-engineering.html) — 권위 글
- [Eugene Yan — LLM patterns](https://eugeneyan.com/writing/llm-patterns/) — production LLM 패턴
- [LangFuse (LLM observability)](https://langfuse.com/docs) — open-source LLM 관측
- [Helicone](https://docs.helicone.ai/) — LLM API 프록시 + 모니터링
- [OpenLLMetry (OpenTelemetry for LLM)](https://www.traceloop.com/docs/openllmetry/introduction) — LLM OTel
- [LiteLLM](https://docs.litellm.ai/) — 여러 LLM provider 통합 인터페이스
- [OpenRouter](https://openrouter.ai/docs) — multi-provider API 게이트웨이
- [Portkey AI](https://docs.portkey.ai/) — AI 게이트웨이 + fallback
- [Prompt caching (Anthropic)](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching) — 비용 절감
- [Batch processing (Anthropic)](https://docs.anthropic.com/en/docs/build-with-claude/batch-processing) — 50% 할인
- [Token counter (tiktoken)](https://github.com/openai/tiktoken) — OpenAI 토크나이저
- [LLM Cost Estimator](https://yourgpt.ai/tools/openai-and-other-llm-api-pricing-calculator) — 비용 계산기
- [Datasette LLM](https://llm.datasette.io/) — CLI 기반 LLM 운영
- [Langfuse + Open-source eval pipeline](https://langfuse.com/docs/scores/overview) — 평가 통합