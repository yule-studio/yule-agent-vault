---
title: "Getting Started — RN 첫 프로젝트 + iOS/Android 환경"
kind: knowledge
project: mobile
agent: engineering-agent/tech-lead
status: current
tags: [mobile, react-native, getting-started, ios, android, setup, beginner]
---

# Getting Started

**[[react-native|↑ RN Hub]]**

> 0 → RN 앱이 시뮬레이터에서 뜨기까지.

## 1. 사전 조건

| | macOS (iOS 빌드) | Windows / Linux |
| --- | --- | --- |
| Node | 20+ (LTS) | 동일 |
| Yarn 또는 npm | 1+ | 동일 |
| Watchman | `brew install watchman` | (선택) |
| **Android Studio** | 필요 | 필요 |
| **Xcode + Command Line Tools** | 필요 (iOS) | 불가 (Windows/Linux 에서는 iOS 빌드 X) |
| **CocoaPods** | `sudo gem install cocoapods` (iOS) | - |
| **JDK 17+** | Android | Android |

→ **iOS 빌드는 Mac 만 가능**. Windows 에서는 Android 만.

## 2. 환경 설치 (macOS 기준)

```bash
# 1. Node + Yarn
brew install node
brew install yarn

# 2. Watchman (파일 watcher)
brew install watchman

# 3. Xcode (App Store) + 설정
#    Xcode → Settings → Locations → Command Line Tools 선택

# 4. CocoaPods (iOS native dep manager)
sudo gem install cocoapods
# 또는
brew install cocoapods

# 5. Java (JDK 17)
brew install --cask zulu@17
# 환경 변수 (~/.zshrc)
export JAVA_HOME=/Library/Java/JavaVirtualMachines/zulu-17.jdk/Contents/Home

# 6. Android Studio
#    https://developer.android.com/studio 에서 다운로드
#    설치 후:
#    - SDK Manager: Android 14 (API 34) 설치
#    - AVD Manager: 가상 디바이스 생성 (Pixel + API 34)
```

## 3. 환경 변수 — `~/.zshrc` / `~/.bashrc`

```bash
# JDK
export JAVA_HOME=/Library/Java/JavaVirtualMachines/zulu-17.jdk/Contents/Home

# Android SDK
export ANDROID_HOME=$HOME/Library/Android/sdk
export PATH=$PATH:$ANDROID_HOME/emulator
export PATH=$PATH:$ANDROID_HOME/platform-tools
```

```bash
source ~/.zshrc
adb --version            # Android Debug Bridge
xcrun --version          # Xcode CLI
node --version
```

## 4. 첫 프로젝트 — `npx @react-native-community/cli`

```bash
npx @react-native-community/cli@latest init MyFirstApp
cd MyFirstApp
```

→ "Bare RN" 프로젝트. iOS + Android native 폴더 포함.

### Expo 로 시작하기 (대안, 권장)
```bash
npx create-expo-app@latest MyExpoApp
cd MyExpoApp
npx expo start
```

→ native 환경 없이도 일단 실행 가능. **신규 학습은 Expo 추천**.

## 5. 실행 — iOS / Android

### iOS (Mac)
```bash
# iOS native dep 설치
cd ios && pod install && cd ..

# 시뮬레이터에서 실행
yarn ios
# 또는 특정 시뮬레이터
yarn ios --simulator="iPhone 15"
```

### Android (Mac / Windows / Linux)
```bash
# Android Studio 의 AVD 켜기 (또는 USB 디바이스 연결)
# 별 터미널에서
yarn start              # Metro bundler

# 다른 터미널에서
yarn android
```

→ 첫 빌드는 1~5분 소요 (gradle 다운로드).

## 6. 프로젝트 구조 (Bare RN)

```
MyFirstApp/
├── android/                  # Android native 코드
│   ├── app/
│   │   ├── build.gradle
│   │   └── src/main/AndroidManifest.xml
├── ios/                      # iOS native 코드
│   ├── MyFirstApp/
│   │   ├── Info.plist
│   ├── Podfile
├── src/                      # 우리가 작성 (직접 만들기)
├── App.tsx                   # 진입점
├── index.js                  # RN 등록
├── package.json
├── metro.config.js
├── babel.config.js
├── tsconfig.json
└── app.json
```

자세히 [[project-structure]].

## 7. App.tsx 의 첫 모습

```tsx
import React, { useState } from 'react';
import { View, Text, Button, StyleSheet } from 'react-native';

export default function App() {
  const [count, setCount] = useState(0);

  return (
    <View style={styles.container}>
      <Text style={styles.title}>안녕, RN!</Text>
      <Text>카운트: {count}</Text>
      <Button title="+1" onPress={() => setCount(c => c + 1)} />
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    backgroundColor: '#fff',
  },
  title: {
    fontSize: 24,
    fontWeight: 'bold',
    marginBottom: 16,
  },
});
```

→ React 와 거의 동일. 차이:
- `<div>` 대신 `<View>`.
- `<span>` 대신 `<Text>`.
- `onClick` 대신 `onPress`.
- CSS 대신 `StyleSheet.create`.

## 8. 디바이스에 빌드

### iOS 실기기
1. Apple Developer 계정 (무료 OK).
2. Xcode 열어 `ios/MyFirstApp.xcworkspace`.
3. Signing & Capabilities — Team 선택.
4. 디바이스 USB 연결.
5. ▶︎ 버튼.

### Android 실기기
1. 디바이스: 설정 → 빌드번호 7번 탭 → 개발자 모드.
2. 개발자 옵션 → USB 디버깅 ON.
3. USB 연결.
4. `adb devices` 로 인식 확인.
5. `yarn android`.

## 9. 첫 에러 트러블슈팅

### iOS 의 `Pod install failed`
```bash
cd ios && pod install --repo-update
```
→ `Podfile.lock` 의 spec 못 찾을 때.

### Metro 의 cache 문제
```bash
yarn start --reset-cache
# 또는
watchman watch-del-all
rm -rf node_modules && yarn
```

### Android 의 `Could not find tools.jar`
→ JDK 8 깔려 있고 17 안 잡힘. `JAVA_HOME` 확인.

### "Unable to load script. Make sure you're either running a Metro server"
→ `yarn start` 가 안 떠 있음. 별 터미널에서 실행.

### iOS 의 `Multiple commands produce`
```bash
cd ios && pod deintegrate && pod install && cd ..
```

## 10. job-answer-app-rn 실행 시 추가 단계

```bash
cd ~/masterway-dev/job-answer-app-rn
yarn install              # node_modules
cd ios && pod install     # iOS native
cd ..

# 환경 변수 (.env) 가 있다면 채우기 (Firebase, Kakao key 등)

# 실행
yarn start                # 별 터미널 1
yarn ios                  # 또는 yarn android
```

→ 첫 실행 전 Firebase / Kakao / Apple developer 콘솔 등록 + 인증 파일 (`GoogleService-Info.plist`, `google-services.json`) 배치 필요.

## 11. Reload / Debug

| 단축키 (iOS sim) | 동작 |
| --- | --- |
| `Cmd + R` | Reload |
| `Cmd + D` | Dev Menu (Debug, Performance) |
| `Cmd + Ctrl + Z` | Shake (= Dev Menu) |

| (Android emulator) | |
| --- | --- |
| `R, R` (두 번) | Reload |
| `Cmd + M` (Mac) / `Ctrl + M` | Dev Menu |

### React DevTools
```bash
yarn global add react-devtools
react-devtools
```

→ 별도 창에서 React tree 확인.

### Flipper (deprecated 중) → 새 RN debugger
- RN 0.73+ 부터 Flipper 비추.
- Hermes inspector + Chrome DevTools 사용.
- `yarn start` 후 j 키 누르면 Hermes debugger.

## 12. 다음 단계

- [[project-structure]] — 실제 프로젝트 src 구조
- [[configuration]] — metro / babel / iOS plist / Android manifest

## 13. 외부 자료

- [reactnative.dev — Setting up environment](https://reactnative.dev/docs/set-up-your-environment)
- [Expo 공식 — Get Started](https://docs.expo.dev/get-started/installation/)
- [Android Studio](https://developer.android.com/studio)
