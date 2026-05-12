---
title: "성능 / 프로파일링"
kind: knowledge
project: agent-references
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-12T23:45:00+09:00
tags:
  - reference
  - links
  - android
  - android-performance
---

# 성능 / 프로파일링

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 15 링크 |

**[[android|↑ Android]]**

> Profiler / Macrobenchmark / Startup / Frame 측정.

## Reference 링크

- [Performance overview](https://developer.android.com/topic/performance) — 권위 docs
- [Android Studio Profiler](https://developer.android.com/studio/profile) — CPU / Memory / Network
- [Macrobenchmark](https://developer.android.com/topic/performance/benchmarking/macrobenchmark-overview) — 실기 성능 측정
- [Microbenchmark](https://developer.android.com/topic/performance/benchmarking/microbenchmark-overview) — 메서드 단위
- [Baseline Profiles](https://developer.android.com/topic/performance/baselineprofiles/overview) — 시작 성능 개선
- [App Startup Library](https://developer.android.com/topic/libraries/app-startup) — 초기화 순서
- [Compose Performance](https://developer.android.com/jetpack/compose/performance) — recomposition 최적화
- [Jank detection (vitals)](https://developer.android.com/topic/performance/vitals/render) — frame drop 측정
- [Perfetto (시스템 트레이스)](https://perfetto.dev/docs/) — Linux + Android 트레이스
- [Android Vitals (Play Console)](https://support.google.com/googleplay/android-developer/answer/9844486) — Play 의 vitals
- [Memory leak — LeakCanary](https://square.github.io/leakcanary/) — 메모리 누수 자동 감지
- [Strict Mode](https://developer.android.com/reference/android/os/StrictMode) — 디버그 안티 패턴 감지
- [Awesome Android Performance](https://github.com/aritraroy/awesome-android-tips) — 자료 인덱스
- [Android Performance Patterns (YouTube)](https://www.youtube.com/playlist?list=PLWz5rJ2EKKc9CBxr3BVjPTPoDPLdPIFCE) — 공식 영상 시리즈
- [App size optimization](https://developer.android.com/topic/performance/reduce-apk-size) — APK 크기