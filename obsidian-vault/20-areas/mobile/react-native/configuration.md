---
title: "Configuration — metro / babel / tsconfig / iOS plist / Android manifest"
kind: knowledge
project: mobile
agent: engineering-agent/tech-lead
status: current
tags: [mobile, react-native, configuration, metro, babel, plist, manifest]
---

# Configuration

**[[react-native|↑ RN Hub]]**

> RN 프로젝트의 모든 설정 파일.

## 1. `package.json` 의 scripts

```json
{
  "scripts": {
    "android": "react-native run-android",
    "ios": "react-native run-ios",
    "lint": "eslint .",
    "start": "react-native start",
    "test": "jest"
  }
}
```

→ web 의 `dev / build / preview` 대신 `android / ios / start`.

## 2. `metro.config.js` — bundler 설정

```js
const { getDefaultConfig, mergeConfig } = require('@react-native/metro-config');

const config = {
  // svg transformer (svg 를 컴포넌트처럼 import)
  transformer: {
    babelTransformerPath: require.resolve('react-native-svg-transformer'),
  },
  resolver: {
    assetExts: defaultConfig.resolver.assetExts.filter(ext => ext !== 'svg'),
    sourceExts: [...defaultConfig.resolver.sourceExts, 'svg'],
  },
};

module.exports = mergeConfig(getDefaultConfig(__dirname), config);
```

→ Metro = RN 의 Webpack/Vite 격. JS 번들링 + 라이브 리로딩.

### 커스텀 — assetExts / extraNodeModules / monorepo

```js
resolver: {
  assetExts: [...defaultConfig.resolver.assetExts, 'lottie'],   // .lottie 도 asset 으로
  extraNodeModules: {
    '@components': path.resolve(__dirname, 'src/components'),
  },
},

// monorepo (workspaces)
watchFolders: [
  path.resolve(__dirname, '../../node_modules'),
  path.resolve(__dirname, '../shared'),
],
```

## 3. `babel.config.js`

```js
module.exports = {
  presets: ['module:@react-native/babel-preset'],
  plugins: [
    [
      'module-resolver',
      {
        root: ['./src'],
        extensions: ['.ios.js', '.android.js', '.js', '.ts', '.tsx', '.json'],
        alias: {
          '@components': './src/components',
          '@pages': './src/pages',
          '@hooks': './src/hooks',
          '@utils': './src/utils',
          '@assets': './src/assets',
        },
      },
    ],
    'react-native-reanimated/plugin',     // reanimated 사용 시 마지막에
  ],
};
```

→ alias + 플랫폼별 파일 (`*.ios.tsx`, `*.android.tsx`) 인식.

## 4. `tsconfig.json`

```json
{
  "extends": "@react-native/typescript-config/tsconfig.json",
  "compilerOptions": {
    "baseUrl": "./",
    "paths": {
      "@components/*": ["src/components/*"],
      "@pages/*": ["src/pages/*"],
      "@hooks/*": ["src/hooks/*"],
      "@utils/*": ["src/utils/*"],
      "@assets/*": ["src/assets/*"]
    }
  }
}
```

→ babel alias 와 동기화. **둘 다 수정해야 함**.

## 5. `app.json`

```json
{
  "name": "JobAnswerFe",
  "displayName": "JobAnswerFe"
}
```

→ 앱 이름. iOS / Android 의 native 설정에 영향. 변경 시 native 빌드 필요.

## 6. `react-native.config.js`

```js
module.exports = {
  project: {
    ios: {},
    android: {},
  },
  assets: ['./src/assets/fonts'],     // 폰트 자동 link
  dependencies: {
    // 특정 lib 자동 link 끄기
    '@some/lib': { platforms: { ios: null } },
  },
};
```

→ `react-native link` (RN CLI) 의 설정. 폰트 install 시 자주 사용.

## 7. iOS — `Info.plist` (권한 / deep link)

`ios/<AppName>/Info.plist`:

```xml
<!-- 권한 메시지 (필수) -->
<key>NSMicrophoneUsageDescription</key>
<string>음성 입력을 위해 마이크 접근이 필요합니다.</string>

<key>NSCameraUsageDescription</key>
<string>프로필 사진 촬영을 위해 카메라 접근이 필요합니다.</string>

<key>NSPhotoLibraryUsageDescription</key>
<string>사진 첨부를 위해 사진 접근이 필요합니다.</string>

<!-- 알림 -->
<key>UIBackgroundModes</key>
<array>
  <string>remote-notification</string>
</array>

<!-- Deep Link / Universal Link -->
<key>CFBundleURLTypes</key>
<array>
  <dict>
    <key>CFBundleURLSchemes</key>
    <array><string>myapp</string></array>
  </dict>
</array>

<!-- App Transport Security (HTTPS 강제 완화) -->
<key>NSAppTransportSecurity</key>
<dict>
  <key>NSAllowsArbitraryLoads</key>
  <true/>
</dict>
```

→ **권한 메시지 누락 시 앱 store 리젝**. 정직한 한국어 / 영어.

## 8. Android — `AndroidManifest.xml`

`android/app/src/main/AndroidManifest.xml`:

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

  <!-- 권한 -->
  <uses-permission android:name="android.permission.INTERNET" />
  <uses-permission android:name="android.permission.CAMERA" />
  <uses-permission android:name="android.permission.RECORD_AUDIO" />
  <uses-permission android:name="android.permission.READ_MEDIA_IMAGES" />
  <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />

  <application
    android:name=".MainApplication"
    android:label="@string/app_name"
    android:icon="@mipmap/ic_launcher"
    android:usesCleartextTraffic="true">
    
    <activity
      android:name=".MainActivity"
      android:launchMode="singleTask"
      android:exported="true">
      <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
      </intent-filter>
      
      <!-- Deep Link -->
      <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="myapp" />
      </intent-filter>
    </activity>
  </application>
</manifest>
```

## 9. Android — `app/build.gradle`

```gradle
android {
    compileSdk 34
    defaultConfig {
        applicationId "com.example.myapp"      // package 이름
        minSdk 24                              // 최소 Android 7.0
        targetSdk 34
        versionCode 1                          // build number
        versionName "1.0.0"                    // 화면 표시 버전
    }

    signingConfigs {
        release {
            storeFile file("my-release-key.keystore")
            storePassword "..."
            keyAlias "..."
            keyPassword "..."
        }
    }

    buildTypes {
        release {
            signingConfig signingConfigs.release
            minifyEnabled true                  // ProGuard
            proguardFiles ...
        }
    }
}
```

## 10. iOS — `Podfile`

```ruby
require_relative '../node_modules/react-native/scripts/react_native_pods'

platform :ios, '14.0'
prepare_react_native_project!

target 'MyApp' do
  config = use_native_modules!

  use_react_native!(
    :path => config[:reactNativePath],
    :hermes_enabled => true,
  )

  # 추가 native pod (직접 추가하는 경우 드뭄. 보통 yarn add 면 자동 link)
end
```

```bash
cd ios && pod install      # 의존성 설치 / 업데이트
```

→ `Podfile.lock` 은 commit. `Pods/` 폴더는 `.gitignore`.

## 11. 환경 변수 — react-native-config 또는 react-native-dotenv

### react-native-dotenv (간단)
```bash
yarn add -D react-native-dotenv
```

```js
// babel.config.js
plugins: [
  ['module:react-native-dotenv', { moduleName: '@env', path: '.env' }],
]
```

```env
# .env
API_URL=https://api.example.com
KAKAO_NATIVE_KEY=xxx
```

```tsx
import { API_URL } from '@env';
```

### react-native-config (more robust)
```bash
yarn add react-native-config
```

→ native 코드에서도 접근 가능 (`Info.plist` 의 `$(API_URL)` 등).

## 12. 환경별 — dev / stage / prod

```bash
# .env.master      (= production)
API_URL=https://api.example.com

# .env.stage
API_URL=https://api-stage.example.com

# .env.test
API_URL=https://api-test.example.com
```

```bash
ENVFILE=.env.stage yarn android
```

→ `react-native-config` 의 표준. 빌드 시 어떤 .env 쓸지 명시.

## 13. ESLint / Prettier

```js
// .eslintrc.js (또는 eslint.config.js)
module.exports = {
  root: true,
  extends: '@react-native',
};
```

```json
// .prettierrc.json
{
  "arrowParens": "avoid",
  "bracketSameLine": true,
  "bracketSpacing": false,
  "singleQuote": true,
  "trailingComma": "all"
}
```

## 14. iOS / Android 의 SplashScreen 변경

### iOS
- `ios/<AppName>/LaunchScreen.storyboard` 를 Xcode 로 열어 편집.

### Android
- `android/app/src/main/res/drawable/launch_screen.xml` 편집.

### `react-native-splash-screen` 라이브러리
```bash
yarn add react-native-splash-screen
cd ios && pod install
```

```tsx
import SplashScreen from 'react-native-splash-screen';
useEffect(() => { SplashScreen.hide(); }, []);
```

→ JS 준비되면 splash 숨기기. 자세히 [[ui/ui]].

## 15. 앱 아이콘 / 이름 변경

### 도구: `app-icon` or `react-native-rename`
```bash
npx react-native-rename "New Name"
yarn add -g app-icon
app-icon generate -i icon.png
```

→ 모든 사이즈 자동 생성 + iOS/Android 양쪽 적용.

## 16. job-answer-app-rn 의 설정 (실제 확인)

```bash
cat ~/masterway-dev/job-answer-app-rn/metro.config.js
cat ~/masterway-dev/job-answer-app-rn/babel.config.js
cat ~/masterway-dev/job-answer-app-rn/app.json
```

→ 사용된 설정 학습.

## 17. 함정

1. **alias 한쪽만 수정** — babel 만 / tsconfig 만 → import 깨짐.
2. **iOS `pod install` 누락** — native lib 추가 후 pod install 안 하면 빌드 실패.
3. **Android `gradle clean`** — Android 빌드 캐시 문제는 `cd android && ./gradlew clean`.
4. **Metro cache** — `yarn start --reset-cache` 자주.
5. **권한 메시지 누락** — iOS store 리젝의 빈번 이유.
6. **release / debug 분기** — debug 만 동작하는 경우 (HTTPS, ProGuard).
7. **`.env` 의 `git add`** — 커밋 X. `.env.example` 만 commit.

## 18. 다음 단계

- [[core-concepts/core-concepts]] — RN 컴포넌트
- [[deployment/deployment]] — 배포 (앱 아이콘 / 키스토어 / TestFlight)

## 19. 관련

- [[react-native]]
- [[project-structure]]
- [[native-features/permissions|권한 처리]]
