---
title: "Deployment in RN — Play Store + App Store hub"
kind: knowledge
project: mobile
agent: engineering-agent/tech-lead
status: current
tags: [mobile, react-native, deployment, play-store, app-store, hub]
---

# Deployment in RN

**[[../react-native|↑ RN Hub]]**

> 첫 배포의 모든 단계. **Play Store + App Store**.

## 1. 큰 그림

```
1. 앱 아이콘 + 스플래시 + 이름 + 버전 설정
2. release keystore 만들기 (Android)
3. Apple Developer / Google Play Console 등록
4. release 빌드 (APK / AAB 또는 .ipa)
5. 업로드 + 심사 제출
6. 심사 통과 → 출시
```

## 2. 영역

| | 노트 |
| --- | --- |
| **Android 빌드 / Play Store** | [[build-android]] |
| **iOS 빌드 / App Store** | [[build-ios]] |
| **버전 관리 / OTA** | (아래 §6) |

## 3. 사전 준비

### Apple
- Apple Developer Program 등록 (연 $99).
- App Store Connect 의 앱 생성.
- App ID 등록.

### Google
- Google Play Console 계정 ($25 일회성).
- 앱 등록.

→ 한국 외 발급 가능 + 한국 계정 가능.

## 4. 첫 배포 체크리스트

- [ ] 앱 이름 / Bundle ID / package name 확정.
- [ ] 앱 아이콘 (모든 크기).
- [ ] splash screen.
- [ ] 권한 메시지 (iOS Info.plist / Android Manifest).
- [ ] 환경 변수 → production 값.
- [ ] release keystore (Android).
- [ ] Apple Developer Certificate / Provisioning Profile.
- [ ] 빌드 번호 / 버전 (`versionCode`, `versionName`).
- [ ] crash report (Sentry / Crashlytics).
- [ ] analytics.
- [ ] privacy policy URL.
- [ ] 스크린샷 + 스토어 description.

## 5. 버전 관리

### `package.json`
```json
{ "version": "1.0.0" }
```

→ JS 의 버전. 자체 사용.

### iOS — `Info.plist` 또는 `project.pbxproj`
```
CFBundleShortVersionString = 1.0.0     (사용자 표시)
CFBundleVersion = 1                     (build number — store 별 unique)
```

### Android — `app/build.gradle`
```gradle
versionName "1.0.0"       // 사용자 표시
versionCode 1              // 정수 — 매 빌드 increment
```

### 통일 — react-native-version
```bash
yarn add -D react-native-version

# package.json 의 version 변경 후
npx react-native-version
```

→ 자동 iOS / Android 동기화.

## 6. OTA (Over-The-Air) 업데이트

JS 코드만 변경 시 store 심사 없이 업데이트:
- **CodePush** (Microsoft) — discontinued (2025).
- **Expo EAS Update** — Expo 의 OTA.
- **react-native-ota-hot-update** — 신생.
- **자체 구축** — bundle 다운로드 + 적용.

```bash
# Expo
yarn add expo-updates
eas update --branch production
```

→ native 코드 변경은 OTA 불가. JS 만.

## 7. CI/CD

### Bitrise / GitHub Actions / Fastlane

```yaml
# .github/workflows/ios.yml
name: iOS build
on: [push]
jobs:
  build:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v3
      - run: yarn install
      - run: cd ios && pod install
      - run: xcodebuild ... archive
      - run: xcodebuild -exportArchive ...
      - run: # upload to TestFlight
```

### Fastlane (가장 인기)

```ruby
# ios/fastlane/Fastfile
lane :beta do
  build_app(scheme: "MyApp")
  upload_to_testflight
end
```

```bash
cd ios && fastlane beta
```

## 8. crash / error 추적

```bash
yarn add @sentry/react-native
```

```ts
import * as Sentry from '@sentry/react-native';

Sentry.init({
  dsn: 'https://...',
  environment: __DEV__ ? 'dev' : 'production',
});

// 자동 — 크래시 / unhandled exception
// 수동
Sentry.captureException(error);
Sentry.captureMessage('something happened');
```

→ Crashlytics / Bugsnag / Sentry 중 선택. Sentry 가 가장 활발.

## 9. analytics

```bash
yarn add @react-native-firebase/analytics
```

```ts
import analytics from '@react-native-firebase/analytics';

await analytics().logEvent('purchase', { item_id: 1, value: 5000 });
await analytics().logScreenView({ screen_name: 'Home' });
```

→ 또는 amplitude / mixpanel / posthog.

## 10. 심사

### iOS
- Apple 의 App Review (1~3일).
- 메타데이터 + 빌드.
- 거절 시 reason + 수정.

### Android
- Google Play Review (수시간 ~ 며칠).
- 빠름 + 보통 통과.

## 11. 흔한 심사 거절 이유

- 권한 메시지 부정확 / 누락.
- 첫 실행 시 강제 종료.
- guidelines 위반 (도박 / 성인 / scraping).
- 사용 안 되는 권한 요청.
- privacy policy 누락.
- ATT (App Tracking Transparency, iOS 14.5+) 미구현.

## 12. job-answer-app-rn 의 배포 추정 단계

1. Apple Developer / Google Play 의 앱 등록.
2. Firebase 의 production 환경 설정 (`GoogleService-Info.plist`).
3. release keystore 생성 / Apple Certificate.
4. 환경 변수 production (`.env.master`).
5. `yarn build:ios` / `yarn build:android` (Fastlane 등).
6. TestFlight / Internal Testing 으로 QA.
7. 심사 제출.

## 13. 학습 우선순위

1. **[[build-android]]** — Play Store 가 더 빠른 출시.
2. **[[build-ios]]** — App Store + TestFlight.
3. **Sentry / Crashlytics**.
4. **CI/CD** (Fastlane).
5. **OTA** (Expo EAS Update 또는 자체).

## 14. 함정

1. **dev 환경 의 빌드** — `.env.dev` 사용한 빌드를 store 에 올림. environment 확인.
2. **release keystore 분실** — Android. 새 keystore 면 업데이트 불가. **백업 필수**.
3. **빌드 번호 같음** — store 가 거절. 매 빌드 increment.
4. **proguard 의 native crash** — 일부 lib 가 proguard 와 충돌. proguard-rules.pro 의 keep rule.
5. **ATS (App Transport Security)** — release 에서 http 통신 fail. HTTPS only.
6. **권한 메시지 영어 only** — 한국어 사용자 confusion. 다국어 권장.
7. **`__DEV__` 의 코드** — production 에 dev 코드 누락 확인.

## 15. 다음 단계

- [[build-android]]
- [[build-ios]]

## 16. 외부 자료

- [reactnative.dev — Publishing to Google Play](https://reactnative.dev/docs/signed-apk-android)
- [reactnative.dev — Publishing to App Store](https://reactnative.dev/docs/publishing-to-app-store)
- [Fastlane](https://fastlane.tools/)
- [Sentry RN](https://docs.sentry.io/platforms/react-native/)

## 17. 관련

- [[../react-native]]
- [[../mobile-publishing/mobile-publishing|상위 mobile-publishing]]
- [[../configuration]]
