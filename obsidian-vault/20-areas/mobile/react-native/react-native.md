---
title: "React Native — 학습 hub"
kind: knowledge
project: mobile
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T04:00:00+09:00
tags: [area, mobile, react-native, hub, learning-path]
---

# React Native — 학습 hub

| 문서 버전 | 작성일 | 주요 변경 사항 |
| --- | --- | --- |
| v.1.0.0 | 2026-05-13 | 최초 placeholder |
| v.2.0.0 | 2026-05-14 | job-answer-app-rn 기반 학습 가이드 전체 구성 |

**[[../mobile|↑ mobile]]**

> **React 의 사상을 native 앱에**. 같은 JS/JSX 로 iOS + Android 두 앱 동시 빌드.
> 실제 프로젝트 기준 = `~/masterway-dev/job-answer-app-rn` (RN 0.81 + React 19 + react-navigation v7).

## 1. Quick Start 로드맵

### Week 1 — 기초 + 환경
1. **[[getting-started]]** — RN 첫 프로젝트, iOS/Android 개발 환경.
2. **[[project-structure]]** — job-answer-app-rn 의 `src/` 구조.
3. **[[configuration]]** — `metro.config.js`, `babel.config.js`, `tsconfig`, `app.json`, iOS Info.plist, Android AndroidManifest.
4. **[[core-concepts/core-concepts]]** — RN 의 기본 컴포넌트 (View/Text/Image 등).
5. **[[core-concepts/rn-components]]** — View / Text / Image / ScrollView / FlatList.
6. **[[core-concepts/stylesheet-and-flexbox]]** — RN 의 StyleSheet + Flexbox (web 과 차이).
7. **[[core-concepts/platform-differences]]** — Platform.OS / `.ios.tsx` / `.android.tsx` 분기.
8. **[[core-concepts/safe-area-and-status-bar]]** — Notch / Dynamic Island / Android status bar.

### Week 2 — Navigation + 상태 + 통신
9. **[[navigation/navigation]]** — react-navigation v7 hub.
10. **[[navigation/stack-navigator]]** — 페이지 간 이동 (native-stack).
11. **[[navigation/tab-navigator]]** — 하단 탭.
12. **[[navigation/deep-linking]]** — URL 스킴 / Universal Link.
13. **[[state/state]]** — RN 상태 관리 (대부분 frontend 패턴 + async-storage).
14. **[[state/async-storage]]** — RN 의 localStorage 대체.
15. **[[networking/networking]]** — fetch / axios / WebSocket hub.
16. **[[networking/axios-fetch]]** — HTTP 통신 (frontend 와 거의 동일).
17. **[[networking/websocket-stomp]]** — STOMP / Socket.IO (job-answer-app-rn 의 채팅 채널).

### Week 3 — Auth + UI
18. **[[auth/auth]]** — 소셜 로그인 hub.
19. **[[auth/apple-signin]]** — `@invertase/react-native-apple-authentication`.
20. **[[auth/google-signin]]** — `@react-native-google-signin/google-signin`.
21. **[[auth/kakao-login]]** — `@react-native-seoul/kakao-login`.
22. **[[ui/ui]]** — UI 라이브러리 hub.
23. **[[ui/vector-icons]]** — `react-native-vector-icons`.
24. **[[ui/svg-and-gradient]]** — `react-native-svg` + `linear-gradient`.
25. **[[ui/lottie]]** — `lottie-react-native` 애니메이션.

### Week 4 — Native 기능 + 최적화 + 배포
26. **[[native-features/native-features]]** — 네이티브 기능 hub.
27. **[[native-features/permissions]]** — `react-native-permissions`.
28. **[[native-features/push-fcm]]** — Firebase Cloud Messaging.
29. **[[native-features/image-picker]]** — 카메라 / 갤러리.
30. **[[native-features/voice-stt]]** — `@react-native-voice/voice` 음성 인식.
31. **[[native-features/inappbrowser]]** — 인앱 브라우저.
32. **[[native-features/localize-i18n]]** — 다국어 / 시스템 로캘 감지.
33. **[[performance/performance]]** — RN 성능 hub.
34. **[[performance/flatlist]]** — 가상화 list (web 의 react-window 대응).
35. **[[performance/reanimated-gesture]]** — 60fps 애니메이션 + 제스처.
36. **[[pitfalls/pitfalls]]** — RN 함정 모음 hub.
37. **[[pitfalls/metro-cache]]** — Metro / cache / yarn link 류 문제.
38. **[[pitfalls/platform-bugs]]** — iOS vs Android 동작 차이.
39. **[[deployment/deployment]]** — 배포 hub.
40. **[[deployment/build-android]]** — APK / AAB / Play Console.
41. **[[deployment/build-ios]]** — Xcode Archive / TestFlight / App Store.

## 2. RN 의 핵심 사상

- **JS 로 작성, native 컴포넌트로 렌더** — `<View>` = iOS `UIView` / Android `ViewGroup`.
- **단일 JS thread + native bridge** — JS 가 native 와 비동기 통신.
- **새로운 아키텍처 (Fabric + TurboModules)** — RN 0.68+ 에서 점진 도입. 성능 개선.
- **`react-native run-ios` / `run-android`** — 디바이스 / 시뮬레이터에서 실행.

## 3. RN vs React (web) — 핵심 차이

| | React (web) | React Native |
| --- | --- | --- |
| 컴포넌트 | `<div>`, `<span>` 등 HTML | `<View>`, `<Text>` 등 native |
| 스타일 | CSS / styled-components / Tailwind | `StyleSheet.create` + JS 객체 |
| navigation | react-router-dom | @react-navigation |
| 로컬 저장소 | localStorage | AsyncStorage |
| 이미지 | `<img>` | `<Image>` |
| 입력 | `<input>` | `<TextInput>` |
| list | `.map(...)` | `<FlatList>` 권장 |
| 빌드 | Vite / Webpack → JS bundle | Metro → JS bundle + iOS/Android native |
| 배포 | 정적 호스팅 | Play Store / App Store |
| 디버깅 | Chrome DevTools | Flipper / React DevTools (RN) |

→ React 의 hook / state / props / context 는 그대로.

## 4. job-answer-app-rn 의 스택

확인:
```bash
cat ~/masterway-dev/job-answer-app-rn/package.json | head -60
```

핵심 라이브러리:
- **React 19 + RN 0.81** — 최신.
- **@react-navigation v7** — native-stack + bottom-tabs.
- **OAuth native lib** — Apple / Google / Kakao 모두 native module.
- **Firebase messaging** — push notification.
- **Realtime** — `@stomp/stompjs`, `@stomp/rx-stomp`, `socket.io-client`.
- **Voice STT** — `@react-native-voice/voice` (음성 인식).
- **Native UI** — `vector-icons`, `svg`, `linear-gradient`, `lottie`, `blur`, `splash-screen`, `spotlight-tour`.
- **Media** — `image-picker`, `nitro-sound`.
- **System** — `permissions`, `gesture-handler`, `screens`, `safe-area-context`, `device-info`, `localize`, `clipboard`, `inappbrowser`.
- **CRM** — Channel Talk SDK.
- **i18n** — `i18n-js`.

## 5. 외부 학습 자료

- [react.dev — Learn React](https://react.dev/learn) (RN 도 React)
- [reactnative.dev — Get Started](https://reactnative.dev/docs/getting-started) — 공식
- [reactnative.dev — Components and APIs](https://reactnative.dev/docs/components-and-apis)
- [Reanimated 공식](https://docs.swmansion.com/react-native-reanimated/)
- [react-navigation 공식](https://reactnavigation.org/)
- [callstack/react-native-paper](https://callstack.github.io/react-native-paper/) (MUI 풍 UI lib)
- [Expo 공식](https://docs.expo.dev/) — RN 의 친화적 layer

## 6. Bare RN vs Expo

| | Bare RN | Expo |
| --- | --- | --- |
| native 코드 수정 | 자유 | EAS Build / Prebuild 필요 |
| 학습 곡선 | 가파름 (iOS/Android 모두 알아야) | 완만 |
| 빌드 환경 | Mac (iOS) + Android Studio + Xcode | EAS cloud 또는 local |
| native module | 직접 link | Expo SDK 또는 config plugin |
| 사용 사례 | OAuth native lib, blur 등 native UI 필요 | 빠른 시작, 표준 SDK 충분 |

→ `job-answer-app-rn` = **Bare RN** (`react-native init`, Expo 미사용).
신규 학습이면 **Expo 부터** 권장. 그 후 native 기능 필요 시 bare 전환.

## 7. 학습 우선순위 (이미 web React 친숙)

이미 [[../../frontend/react/react|frontend/React]] 학습했으면:
1. **차이점만 빠르게** — [[core-concepts/rn-components]], [[core-concepts/stylesheet-and-flexbox]], [[core-concepts/platform-differences]].
2. **navigation** — react-router 대신 react-navigation.
3. **AsyncStorage** — localStorage 대신.
4. **native lib 1 개씩** — auth, push, image-picker.
5. **빌드 / 배포** — 가장 새로운 영역.

## 8. 관련

- [[../mobile|↑ mobile]] · [[../android-kotlin/android-kotlin|Android Kotlin]] · [[../ios-swift/ios-swift|iOS Swift]] · [[../flutter-dart/flutter-dart|Flutter]]
- [[../../frontend/react/react|frontend React]] — 공유 사상
- [[../mobile-publishing/mobile-publishing|배포 일반]]
