---
title: "API 문서화 — Hub (Swagger / OpenAPI / Postman)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T16:30:00+09:00
tags:
  - backend
  - java-spring
  - api-docs
  - swagger
  - openapi
  - hub
---

# API 문서화 — Hub (Swagger / OpenAPI / Postman)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | hub — 도구 선택 / 환경별 노출 정책 |

**[[../java-spring|↑ Java Spring]]**

> Spring Boot 3 + Java 21 에서 API 문서 생성 / 배포 / 보안.

---

## 1. 도구 선택

| 도구 | 특징 | 권장 |
| --- | --- | --- |
| **springdoc-openapi** | Spring Boot 3 표준 / OpenAPI 3.0 자동 생성 | ✅ 1순위 |
| springfox | OpenAPI 2.0 (Swagger 2) / 2024 부터 거의 비활성 | ❌ legacy |
| Postman Collection | API client 도 함께 | 보조 |
| Bruno / Hoppscotch | 오픈소스 Postman 대안 | 선택 |
| **REST Docs** | 테스트 기반 문서화 (snippet) | 엄격 정확도 / 큰 팀 |

→ 본 vault 는 **springdoc-openapi** 표준.

---

## 2. 핵심 정책 — 환경별 노출

| 환경 | Swagger UI | OpenAPI yaml | 비고 |
| --- | --- | --- | --- |
| local / dev | ✅ 모두 공개 | ✅ | 자유 사용 |
| staging | ✅ basic auth | ✅ | QA / 외부 검토 |
| **production** | ⚠️ **차단 또는 internal only** | ⚠️ | 보안 — API 구조 노출 X |

> **함정**: 운영 Swagger UI 가 그대로 열려 있어 → 모든 endpoint / 인증 메커니즘 노출 → 공격자 친절 가이드. **반드시 차단** 또는 VPN / internal IP 만.

---

## 3. 노트 진행

| 노트 | 상태 |
| --- | --- |
| [[swagger-springdoc]] | ✅ — springdoc-openapi 본편 |
| rest-docs (예정) | 🟡 — 테스트 기반 |
| postman-bruno (예정) | 🟡 |
| api-versioning (예정) | 🟡 — v1 / v2 / header 기반 |

---

## 4. springdoc-openapi 가 자동으로 처리하는 것

- `@RestController` / `@RequestMapping` → path
- `@RequestBody` (record / class) → schema
- `@PathVariable`, `@RequestParam` → parameter
- `@ResponseStatus` → response code
- Bean Validation (`@NotBlank`, `@Size`, ...) → schema constraint
- `@Tag`, `@Operation`, `@Schema`, `@Parameter` 으로 보강

→ Controller 코드만 작성해도 90% 의 문서 자동 생성. 인증 / 예시 / 보안 등이 명시적 보강 영역.

---

## 5. 무엇을 어디서

| 문서 영역 | 위치 |
| --- | --- |
| springdoc 설정 / 의존성 / yaml export / Swagger UI 보안 | [[swagger-springdoc]] |
| Controller / DTO 의 @Operation / @Schema | [[swagger-springdoc]] §6 |
| 인증 헤더 / Bearer / OAuth2 표현 | [[swagger-springdoc]] §7 |
| OpenAPI yaml 의 빌드 / CI / 외부 발행 | [[swagger-springdoc]] §10 |
| API 버전 정책 (v1/v2, 매트릭스) | api-versioning (예정) |
| 테스트 기반 문서 (REST Docs) | rest-docs (예정) |

---

## 6. 관련

- [[../api-design/api-design|↗ API 레시피]]
- [[swagger-springdoc]] — springdoc 본편
- [[../java-spring|↑ Java Spring]]
