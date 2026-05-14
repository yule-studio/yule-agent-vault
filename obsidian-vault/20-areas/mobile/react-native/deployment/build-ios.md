---
title: "iOS Build — Xcode Archive / TestFlight / App Store"
kind: knowledge
project: mobile
agent: engineering-agent/tech-lead
status: current
tags: [mobile, react-native, ios, build, app-store, testflight, deployment]
---

# iOS Build

**[[deployment|↑ Deployment Hub]]**

> Xcode 로 archive → TestFlight → App Store. **Mac 필수**.

## 1. 사전 조건

- **Apple Developer Program** 가입 (연 $99).
- **Mac** (필수, Xcode 만 빌드 가능).
- **Xcode 15+** 최신.

## 2. App Store Connect 의 앱 등록

1. **appstoreconnect.apple.com** → 내 앱 → `+`.
2. **Bundle ID** — 이미 Apple Developer 의 App IDs 에 등록된 것.
3. **앱 이름** (검색 가능).
4. **첫 build 업로드 전엔 store 정보만 채울 수 있음**.

## 3. App ID + Certificate + Profile

Apple Developer 콘솔:

### App ID
- com.example.myapp (Bundle ID).
- Capabilities 활성화 (Push Notifications, Sign in with Apple, Associated Domains 등).

### Certificate (개발자 인증서)
- 자동 (Xcode 가 manage) — 권장.
- 수동 (.cer / .p12) — Fastlane 등 자동화 시.

### Provisioning Profile
- Development — 개발 + dev 디바이스.
- App Store Distribution — App Store 업로드.
- Ad Hoc — 사내 배포.

→ Xcode 의 "Automatically manage signing" 가 가장 쉬움.

## 4. Xcode 설정

`open ios/<AppName>.xcworkspace` (`.xcodeproj` 아님).

### General
- Display Name (사용자 표시 이름).
- Bundle Identifier.
- Version (사용자 표시): `1.0.0`.
- Build (store 별 unique): `1`, `2`, `3`, ...
- Deployment Target: iOS 14+ 권장.

### Signing & Capabilities
- Team 선택.
- Automatically manage signing 체크.
- Capability 추가 (Push, Sign in with Apple 등).

## 5. 권한 메시지 — Info.plist

```xml
<key>NSCameraUsageDescription</key>
<string>프로필 사진 촬영을 위해 카메라가 필요합니다.</string>
<!-- 사용 안 하는 권한 — 제거 필수 (안 그러면 거절) -->
```

## 6. 앱 아이콘

- `Assets.xcassets/AppIcon.appiconset/` 에 모든 크기 (1024 + 다양한 device).
- 자동 도구 — `app-icon` (위 [[build-android]] 참조).

## 7. Splash Screen

`ios/<AppName>/LaunchScreen.storyboard` 편집 (Xcode 에서) 또는 `react-native-bootsplash`.

## 8. 빌드 — Archive

### Xcode 에서
1. 좌상단 device 선택 → **Any iOS Device** (또는 실기기).
2. **Product → Archive**.
3. 빌드 (5~10분).
4. Organizer 창에서 archive 확인.

### CLI 에서
```bash
cd ios

# Release 빌드
xcodebuild -workspace MyApp.xcworkspace \
  -scheme MyApp \
  -configuration Release \
  -archivePath build/MyApp.xcarchive \
  archive

# IPA export
xcodebuild -exportArchive \
  -archivePath build/MyApp.xcarchive \
  -exportPath build \
  -exportOptionsPlist ExportOptions.plist
```

## 9. TestFlight 업로드

### Xcode 에서
1. Organizer → archive 선택 → **Distribute App**.
2. **App Store Connect** → Upload.
3. 자동 업로드 + 처리 (10~30분).
4. App Store Connect 에서 build 확인.

### CLI / Transporter / Fastlane
```bash
xcrun altool --upload-app --type ios \
  -f build/MyApp.ipa \
  -u "your@apple.id" \
  -p "@keychain:AC_PASSWORD"
```

## 10. TestFlight 테스트

- Apple 의 처리 후 → TestFlight 의 build 에 표시.
- Internal Testing — 팀 멤버 (최대 100명).
- External Testing — 외부 테스터 (최대 10,000명, Apple 의 review 필요).
- 90일 만료.

→ 정식 출시 전 충분히 TestFlight 로 QA.

## 11. App Store 제출

1. App Store Connect → 앱 → 새 버전 만들기.
2. **버전 정보** 채우기:
   - 스크린샷 (모든 디바이스).
   - 앱 설명.
   - 키워드.
   - 카테고리.
   - 가격 / 무료.
   - Age Rating.
   - privacy policy URL.
   - support URL.
3. **빌드 선택** (TestFlight 에서 검증된 것).
4. **심사 제출**.

### 심사 기간
- 평균 24시간 ~ 7일.
- 거절 시 reason 안내 → 수정 후 재제출.

## 12. 자주 거절 이유

- 사용 안 되는 권한 메시지.
- privacy policy 누락.
- 첫 실행 시 강제 종료 (시뮬레이터 만 테스트하면 못 잡음).
- guidelines 4.8 (Sign in with Apple 누락 — 다른 OAuth 만 있을 때).
- guidelines 5.1.1 (사용자 정보 명확 동의).
- Apple ID 사용 못 함 (회원 탈퇴 시 Apple revoke 안 함).
- IAP (앱 내 결제) 우회 — Apple 의 결제 시스템 강제.

## 13. Fastlane — 자동화

```bash
cd ios
bundle init
echo 'gem "fastlane"' >> Gemfile
bundle install
fastlane init
```

`ios/fastlane/Fastfile`:
```ruby
default_platform(:ios)

platform :ios do
  desc "Build and upload to TestFlight"
  lane :beta do
    increment_build_number(xcodeproj: "MyApp.xcodeproj")
    build_app(workspace: "MyApp.xcworkspace", scheme: "MyApp")
    upload_to_testflight
  end

  desc "Build and submit to App Store"
  lane :release do
    increment_build_number(xcodeproj: "MyApp.xcodeproj")
    build_app(workspace: "MyApp.xcworkspace", scheme: "MyApp")
    upload_to_app_store(skip_metadata: true, skip_screenshots: true)
  end
end
```

```bash
cd ios
fastlane beta            # TestFlight
fastlane release         # App Store
```

→ Apple ID + App Specific Password 필요. 환경 변수 또는 `fastlane match` (인증서 git 동기화).

## 14. 환경 변수 — production

```bash
# react-native-config 의 .env 선택
ENVFILE=.env.master xcodebuild -workspace ... build
```

→ release 빌드 전 production env 확인.

## 15. job-answer-app-rn iOS 빌드 단계 추정

```bash
# 1. env
cp .env.master .env

# 2. version 올리기
cd ios
agvtool new-marketing-version 1.0.1     # CFBundleShortVersionString
agvtool new-version -all $(($(agvtool what-version -terse) + 1))   # build

# 3. Pods 재설치 (필요 시)
pod install

# 4. archive (Xcode 또는 fastlane)
fastlane beta
```

## 16. 흔한 빌드 에러

### `Code Signing Error`
- Team 선택 안 됨.
- Provisioning Profile 만료.
- Bundle ID mismatch.

해결:
- Xcode → Signing & Capabilities → Team 재선택.
- 자동 manage signing 켜기.

### `Module 'X' not found`
- Pod install 누락.
- `pod deintegrate && pod install`.

### `Hermes engine not found`
- Pods 의 Hermes 못 받음.
- `pod repo update && pod install`.

### `Library not loaded: @rpath/XX.framework`
- Embed 설정 누락.
- Build Phases → Embed Frameworks 확인.

## 17. 함정

1. **`.xcodeproj` 열기** — `.xcworkspace` 가 정답.
2. **build number 안 올림** — App Store 가 거절.
3. **dev env 의 release** — production 값 누락.
4. **TestFlight 통과 = App Store 통과** ❌ — App Store 가 더 엄격.
5. **iOS 의 IAP 우회** — 결제 logic 은 Apple In-App Purchase 만.
6. **사용 안 되는 권한 메시지** — Info.plist 정리.
7. **Provisioning Profile 만료** — 1년 마다. 갱신.

## 18. 외부 자료

- [reactnative.dev — Publishing to App Store](https://reactnative.dev/docs/publishing-to-app-store)
- [Apple — App Store Review Guidelines](https://developer.apple.com/app-store/review/guidelines/)
- [Fastlane iOS](https://docs.fastlane.tools/getting-started/ios/setup/)
- [App Store Connect Help](https://developer.apple.com/help/app-store-connect/)

## 19. 관련

- [[deployment]]
- [[build-android]]
- [[../configuration]]
- [[../auth/apple-signin]]
