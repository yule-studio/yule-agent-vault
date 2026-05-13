---
title: "Singleton — Hub"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
tags: [design-patterns, gof, creational, singleton]
---

# Singleton — Hub

**[[../design-patterns|↑ 디자인 패턴]]** · GoF Creational

## 1. 한 줄 정의

**한 클래스가 인스턴스 하나만 가지도록 보장하고 그 인스턴스에 글로벌 접근점을 제공한다.**

## 2. 의도 (Intent)

- 단 1개의 인스턴스만 존재한다.
- 어디서든 같은 인스턴스에 접근할 수 있다.
- 첫 호출 시 lazy 생성 (보통).

## 3. 구조

```
   ┌─────────────┐
   │  Singleton  │
   │─────────────│
   │-instance:   │  ← static, 자기 자신 1 개 보유
   │  Singleton  │
   │─────────────│
   │+getInstance │  ← static
   │-Singleton() │  ← private constructor
   │+ operation()│
   └─────────────┘
```

## 4. 언제 쓰는가

- **로깅 / 설정 / DB 연결 풀** — 시스템 전체 1 개만.
- **하드웨어 자원 추상화** — 프린터 스풀러 / 파일 manager.
- **캐시 / 레지스트리** — 글로벌 공유.

## 5. 언제 안 쓰는가 (안티)

- **테스트 어려움** — global state 가 mocking 막음.
- **숨겨진 의존성** — 모든 코드가 Singleton 에 결합.
- **동시성** — multi-thread 의 race / double-checked locking 복잡.
- **확장** — 나중에 "2 개 필요" 면 대규모 리팩.

→ **현대 권장: Dependency Injection** (DI 컨테이너가 "single instance 보장" 을 대신).

## 6. 언어 별 구현

- [[singleton-java|Java / Spring]]
- [[singleton-python|Python / Django / FastAPI]]
- [[singleton-typescript|TypeScript / Node / React / NestJS]]
- [[singleton-go|Go]]

## 7. 비교

| | Singleton | Module-level constant |
| --- | --- | --- |
| 인스턴스 통제 | ✓ | 언어가 보장 |
| Lazy 초기화 | ✓ | 부분적 |
| 테스트 격리 | ✗ | ✗ |
| 명시적 의도 | ✓ | 약함 |

| | Singleton | Dependency Injection (DI) |
| --- | --- | --- |
| 인스턴스 수 | 1 강제 | 컨테이너 정책 |
| 테스트 | 어려움 | 쉬움 (mock 주입) |
| 결합도 | 강함 | 약함 |
| 코드 양 | 적음 | 약간 많음 |

## 8. 실 프레임워크 예

- **Spring**: `@Component` / `@Service` 의 기본 scope = singleton.
- **Java Logger**: `Logger.getLogger("name")`.
- **Python**: module 자체가 singleton (한 번 import).
- **Go**: package-level 변수 + `sync.Once`.
- **React**: Context Provider + Redux store.

## 9. 함정

1. **Multi-thread 에서 double-checked locking** — 잘못 구현 시 race.
2. **Reflection / Serialization 우회** — Java 의 `Class.forName + newInstance` 등.
3. **Cloning** — `clone()` override 안 하면 인스턴스 2 개.
4. **Multi-class loader** — Java EE 환경에서 ClassLoader 별 별도 인스턴스.
5. **테스트 시 reset 불가** — static state 가 다음 test 로 leak.
6. **글로벌 상태 폭주** — 모든 모듈이 직접 참조 → 결합도 폭증.

## 10. 관련

- [[../factory-method/factory-method]] — Singleton 의 인스턴스 생성을 Factory 가 책임지기도.
- [[../prototype/prototype]] — Singleton 의 반대 (다수 인스턴스 복제).
- [[../../software-engineering/software-engineering]] — DI / IoC.
