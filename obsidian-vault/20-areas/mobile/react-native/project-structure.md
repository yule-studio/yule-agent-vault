---
title: "Project Structure — job-answer-app-rn 기준 src 구조"
kind: knowledge
project: mobile
agent: engineering-agent/tech-lead
status: current
tags: [mobile, react-native, structure, beginner]
---

# Project Structure

**[[react-native|↑ RN Hub]]**

> `job-answer-app-rn` 기준 + Bare RN 의 표준 구조.

## 1. 최상위 layout

```
job-answer-app-rn/
├── android/                # Android native (Gradle / Kotlin / Java)
├── ios/                    # iOS native (Xcode / Swift / Objective-C)
├── src/                    # 우리가 작성하는 JS/TS 코드
├── __tests__/              # Jest 테스트
├── App.tsx                 # 루트 컴포넌트
├── index.js                # native 진입점 (App 을 등록)
├── package.json
├── yarn.lock
├── tsconfig.json
├── babel.config.js         # Babel 설정 (module-resolver)
├── metro.config.js         # Metro bundler 설정
├── react-native.config.js  # RN CLI 설정
├── app.json                # 앱 이름 / displayName
├── jest.config.js
├── Gemfile / Gemfile.lock  # Ruby (CocoaPods 의 dep)
├── declaration.d.ts        # 글로벌 type 선언
└── README.md
```

## 2. native 폴더는 무엇?

### `android/`
- Android 의 Gradle 프로젝트.
- `app/build.gradle` — 빌드 설정 (버전, SDK, 의존성).
- `app/src/main/AndroidManifest.xml` — 권한, 메인 액티비티.
- `app/src/main/res/` — 리소스 (아이콘, 색).
- `gradle.properties`, `settings.gradle`.

### `ios/`
- Xcode 프로젝트.
- `Podfile` — CocoaPods 의존성.
- `MyApp/Info.plist` — 권한 메시지, deep link scheme.
- `MyApp/AppDelegate.swift` (또는 `.mm`) — iOS 앱 lifecycle.
- `.xcworkspace` — Xcode 로 열기.

→ 직접 수정하는 경우:
- 권한 (마이크 / 카메라 / 위치) 추가.
- native lib 가 link 안 될 때.
- 앱 아이콘 / splash screen 변경.
- 버전 / build number 변경.

## 3. job-answer-app-rn 의 `src/` 구조

```
src/
├── assets/                 # 이미지, 폰트, lottie JSON
├── components/             # 재사용 컴포넌트
│   ├── BottomSheet/
│   ├── Chat/
│   ├── CommonButton/
│   ├── ConfirmDialog/
│   ├── ConfirmInputDialog/
│   ├── CustomTextInput/
│   ├── CustomToggle/
│   ├── DevelopPage/
│   ├── ErrorMessage/
│   ├── Header/
│   ├── MarkdownView/
│   ├── MarkerText/
│   ├── ProgressHeader/
│   ├── RecordingButton/
│   ├── SimplifiedSvgIcon/
│   ├── SingleButtonDialog/
│   ├── SocialButton/
│   ├── SocialLoginSection/
│   ├── TermsAgreementBottomSheet/
│   └── VoiceInputBar/
├── constants/              # 상수 (color, font, key)
├── hooks/                  # custom hook (예: useChatSocket.ts)
├── i18n/                   # 다국어 (ko, en JSON + setup)
├── meta/                   # 메타 (link.ts — RootStackParamList)
├── navigate/               # navigator (MainTabNavigator.tsx)
├── pages/                  # 페이지 (route 의 leaf)
│   ├── basic-question/
│   ├── main/
│   ├── pre-question/
│   ├── reset-password/
│   ├── setting/
│   ├── signin/
│   ├── signup/
│   ├── start/
│   ├── tabs/
│   └── tutorial/
├── router.tsx              # 루트 navigator (Stack)
├── styles/                 # 글로벌 스타일 (color, spacing)
├── types/                  # 글로벌 TS type
└── utils/                  # helper (date, format, validate)
```

→ frontend 의 `answer-fe` / `job-answer-fe` 와 비슷한 결.

## 4. 페이지 폴더의 nested 구조

```
pages/signup/
├── index.tsx               # 메인 화면
├── _pages/                 # signup flow 의 sub-page
│   ├── finish/
│   │   └── index.tsx
│   ├── step-1/
│   ├── step-2/
│   └── ...
└── _components/            # signup 전용 컴포넌트 (다른 page 와 안 공유)
```

→ `_` prefix = "private to this folder" 컨벤션.

## 5. App.tsx — 루트 컴포넌트

```tsx
// App.tsx (실제 job-answer-app-rn 패턴)
import { NavigationContainer } from '@react-navigation/native';
import { SafeAreaProvider } from 'react-native-safe-area-context';
import { GestureHandlerRootView } from 'react-native-gesture-handler';
import { RootNavigator } from './src/router';

export default function App() {
  return (
    <GestureHandlerRootView style={{ flex: 1 }}>
      <SafeAreaProvider>
        <NavigationContainer>
          <RootNavigator />
        </NavigationContainer>
      </SafeAreaProvider>
    </GestureHandlerRootView>
  );
}
```

→ Provider 3 계층:
1. `GestureHandlerRootView` — 제스처 (drag, swipe).
2. `SafeAreaProvider` — 노치 / 상태바.
3. `NavigationContainer` — react-navigation.

추가 가능:
- `QueryClientProvider` — react-query 사용 시.
- `ThemeProvider` — styled-components / Theme 사용 시.

## 6. index.js — native 진입

```js
import { AppRegistry } from 'react-native';
import App from './App';
import { name as appName } from './app.json';

AppRegistry.registerComponent(appName, () => App);
```

→ 보통 수정 안 함.

## 7. router.tsx — 루트 stack

```tsx
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import { RootStackParamList } from './meta/link';

const Stack = createNativeStackNavigator<RootStackParamList>();

export const RootNavigator = () => (
  <Stack.Navigator
    initialRouteName="Start"
    screenOptions={{
      headerShown: false,
      gestureEnabled: false,
    }}
  >
    <Stack.Screen name="Start" component={StartPage} />
    <Stack.Screen name="MainTab" component={MainTabNavigator} />
    <Stack.Screen name="Signin" component={SigninPage} />
    <Stack.Screen name="Signup" component={SignupPage} />
    ...
  </Stack.Navigator>
);
```

자세히 [[navigation/stack-navigator]].

## 8. 명명 컨벤션

- **컴포넌트 폴더 / 파일**: PascalCase (`CommonButton/`, `CommonButton.tsx`).
- **페이지 폴더**: kebab-case (`basic-question/`, `reset-password/`).
- **utility / hook 파일**: camelCase (`useChatSocket.ts`, `formatDate.ts`).
- **상수**: SCREAMING_CASE (`KAKAO_KEY`, `API_URL`).

→ `job-answer-app-rn` 패턴.

## 9. import 경로 — babel module-resolver

```js
// babel.config.js
module.exports = {
  presets: ['module:@react-native/babel-preset'],
  plugins: [
    [
      'module-resolver',
      {
        root: ['./src'],
        alias: {
          '@components': './src/components',
          '@pages': './src/pages',
          '@hooks': './src/hooks',
          '@utils': './src/utils',
        },
      },
    ],
  ],
};
```

```tsx
import { CommonButton } from '@components/CommonButton';
import { useChatSocket } from '@hooks/useChatSocket';
```

→ `tsconfig.json` 의 `paths` 도 동기화 필요.

## 10. assets 폴더 — 이미지 / 폰트 / lottie

```
src/assets/
├── images/
│   ├── logo.png
│   ├── logo@2x.png            # iOS retina
│   ├── logo@3x.png
├── fonts/
│   ├── Pretendard-Bold.ttf
├── lottie/
│   ├── loading.json
└── svg/
    └── icon.svg
```

```tsx
// 이미지
import logo from '@/assets/images/logo.png';
<Image source={logo} />

// 폰트 — react-native.config.js + link 필요
yarn add react-native-vector-icons
npx react-native link
```

자세히 [[ui/svg-and-gradient]], [[ui/lottie]].

## 11. 환경 변수

```bash
yarn add react-native-dotenv
# 또는 react-native-config
```

```env
# .env
API_URL=https://api.example.com
KAKAO_NATIVE_KEY=xxx
```

```tsx
import { API_URL } from '@env';
```

→ Vite 의 `import.meta.env.VITE_X` 와 같은 역할. 자세히 [[configuration]].

## 12. masterway-dev / 다른 -fe 와 차이

| | web -fe | RN -app-rn |
| --- | --- | --- |
| 라우팅 | react-router-dom | @react-navigation |
| 저장 | localStorage | AsyncStorage |
| 입력 | input | TextInput |
| 스타일 | styled-components / Tailwind | StyleSheet.create |
| HTTP | axios + interceptor | axios + interceptor (거의 동일) |
| OAuth | redirect / SDK | native lib (apple-auth, kakao-login, google-signin) |
| Push | web push (서비스 워커) | FCM native |

→ web 기반 지식이 70~80% 그대로 적용.

## 13. 함정

1. **이미지 retina 누락** — `@2x`, `@3x` 안 만들면 흐림.
2. **android/ios 폴더 안 들여다보지 않음** — 권한 / 빌드 에러는 native 폴더 수정 필요.
3. **`metro.config.js` 의 watchFolders** — yarn workspace 같은 monorepo 면 추가 필요.
4. **alias 동기화 누락** — babel 만 설정하고 tsconfig 안 하면 TS 가 import 못 찾음.
5. **`.gitignore` 누락** — `ios/Pods/`, `android/build/`, `*.keystore` 등 커밋 금지.
6. **AppDelegate.mm 의 native 코드** — 일부 SDK 가 직접 수정 요구. README 정독.

## 14. 다음 단계

- [[configuration]] — metro / babel / tsconfig / Info.plist / AndroidManifest
- [[core-concepts/core-concepts]] — RN 컴포넌트

## 15. 관련

- [[react-native]]
- [[../../frontend/react/project-structure|web 의 src 구조]]
