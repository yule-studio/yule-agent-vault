---
title: "RN 함정 모음 — Metro / Platform / 빌드 등"
kind: knowledge
project: mobile
agent: engineering-agent/tech-lead
status: current
tags: [mobile, react-native, pitfalls, gotchas, hub]
---

# RN 함정 모음

**[[../react-native|↑ RN Hub]]**

> RN 특유의 자주 만나는 함정.

## 1. 카테고리

| | 노트 |
| --- | --- |
| Metro / 캐시 / 빌드 | [[metro-cache]] |
| iOS vs Android 동작 차이 | [[platform-bugs]] |
| React 공통 함정 | [[../../../frontend/react/pitfalls/pitfalls|web pitfalls]] |

## 2. TOP 10 RN-specific 함정

### (1) `<View>` 안 raw string
```tsx
<View>안녕</View>          // ❌ Text strings must be rendered within a <Text>
<View><Text>안녕</Text></View>   // ✅
```

### (2) `onPress` 대신 `onClick`
```tsx
<Pressable onClick={...} />    // ❌ 동작 안 함
<Pressable onPress={...} />    // ✅
```

### (3) Metro cache
```bash
yarn start --reset-cache       # JS 코드 변경 안 반영 시
```
→ 자세히 [[metro-cache]].

### (4) iOS pod install 누락
```bash
cd ios && pod install
```
→ native lib 추가 후 필수. 안 하면 Linker 에러.

### (5) iOS 빌드 — Xcode 만 동작
- `yarn ios` 가 안 되면 → Xcode 열어 직접 빌드 → 상세 에러 확인.

### (6) Android `Could not find tools.jar`
- JDK 17 안 설정. `JAVA_HOME` 확인.

### (7) localhost (Android emulator)
- `http://localhost` ❌ → `http://10.0.2.2`.

### (8) Info.plist / Manifest 의 권한 메시지 누락
- iOS 카메라 / 마이크 first request 시 강제 종료.

### (9) Hermes vs JSC
- Intl / Date / Regex 일부 차이.
- Hermes 가 기본. 확인.

### (10) Strict Mode 의 useEffect 2 회
- Dev 모드. cleanup 작성 강제 + production 은 1 회.

## 3. 자주 만나는 에러 메시지

### `Unable to load script. Make sure you're either running a Metro server`
→ `yarn start` 안 켜짐. 별 터미널.

### `Multiple commands produce '/path/...'` (iOS)
```bash
cd ios && pod deintegrate && pod install
```

### `Could not find a declaration file for module 'X'`
→ TS 의 `@types/X` 누락. 또는 `declaration.d.ts` 에 `declare module 'X';`.

### `Invariant Violation: Text strings must be rendered within a <Text>`
→ `<View>안녕</View>` — `<Text>` 안으로.

### `Maximum update depth exceeded`
→ useEffect 무한 loop. [[../../../frontend/react/pitfalls/re-render-loops|re-render-loops]].

### `RNGestureHandlerModule could not be found`
→ `react-native-gesture-handler` 미설치 또는 `GestureHandlerRootView` 누락.

### `requireNativeComponent: 'RNX' was not found in the UIManager`
→ native module 미설치 또는 pod install 누락 또는 autolink 실패.

### `Element type is invalid: expected a string ... but got: undefined`
→ import 의 default vs named 잘못. 또는 lib 자체 안 import.

## 4. RN 0.72+ 의 새 아키텍처 (Fabric / TurboModules)

- 점진 도입. 기본 자동 활성화 (0.74+).
- 일부 옛 lib 호환 X.
- 문제 시 일시적으로 끄기 (`newArchEnabled=false`).

## 5. 일반적인 디버깅 절차

1. **에러 메시지 정확히 읽기** — 보통 직접적 힌트.
2. **Metro cache reset** — `yarn start --reset-cache`.
3. **`pod install` 다시** — iOS 의 90%.
4. **Android `gradlew clean`** — Android 의 90%.
5. **node_modules 삭제 + 재설치**.
6. **Xcode / Android Studio 로 직접 빌드** — JS 빌드와 native 빌드 분리.

```bash
# 안전한 reset 시퀀스
watchman watch-del-all
rm -rf node_modules
rm -rf ios/Pods ios/Podfile.lock
yarn install
cd ios && pod install && cd ..
cd android && ./gradlew clean && cd ..
yarn start --reset-cache
```

## 6. dev 모드의 함정

- Strict Mode → useEffect 2 회.
- Reload (`Cmd+R`) — state 손실.
- HMR (Fast Refresh) — 가끔 cache 오류.
- 시뮬레이터 의 시계 / 시간대.

## 7. 시뮬레이터 vs 실기기

| | 시뮬레이터 | 실기기 |
| --- | --- | --- |
| 카메라 | 시뮬레이션 | 실제 |
| 음성인식 | 작동 | 작동 |
| 푸시 | iOS 16+ 의 `.apns` drop / Android 가능 | 정상 |
| OAuth 일부 | 동작 | 동작 |
| Apple Sign In | 동작 | 동작 |
| Kakao 앱 callback | 카톡 미설치 | 카톡 가능 |
| 성능 측정 | 부정확 | 정확 |

→ **실기기 정기 테스트 필수**.

## 8. 학습 우선순위

1. **[[metro-cache]]** — 가장 흔한 빌드 문제.
2. **[[platform-bugs]]** — iOS vs Android.
3. **[[../../../frontend/react/pitfalls/re-render-loops|re-render-loops]]** — React 공통.
4. **[[../../../frontend/react/pitfalls/stale-closures|stale-closures]]** — React 공통.

## 9. 관련

- [[metro-cache]]
- [[platform-bugs]]
- [[../../../frontend/react/pitfalls/pitfalls|web pitfalls]]
- [[../core-concepts/platform-differences]]
