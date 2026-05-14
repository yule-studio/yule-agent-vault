---
title: "Android Build — APK / AAB / Play Store"
kind: knowledge
project: mobile
agent: engineering-agent/tech-lead
status: current
tags: [mobile, react-native, android, build, play-store, deployment]
---

# Android Build

**[[deployment|↑ Deployment Hub]]**

> APK (테스트) / AAB (Play Store) 빌드 + 배포.

## 1. APK vs AAB

| | APK | AAB (Android App Bundle) |
| --- | --- | --- |
| 형식 | 직접 설치 가능 | Play Store 만 |
| Play Store 업로드 | **2021년 8월부터 불가** | 필수 |
| 사이즈 | 모든 디바이스용 | Play Store 가 디바이스별 split |
| 직접 배포 (사내) | OK | apks 의 별 도구 |

→ Play Store = AAB. 직접 배포 / 테스트 = APK.

## 2. 1. Release Keystore 생성

```bash
cd android/app
keytool -genkeypair -v \
  -storetype PKCS12 \
  -keystore my-release-key.keystore \
  -alias my-key-alias \
  -keyalg RSA -keysize 2048 -validity 10000
```

- 비밀번호 입력 — **잊지 말기**.
- `my-release-key.keystore` 가 생성.

→ **이 파일 + 비밀번호 잃으면 앱 업데이트 영영 불가**. 안전한 곳 백업.

## 3. 2. gradle 에 등록

`android/gradle.properties`:
```properties
MYAPP_UPLOAD_STORE_FILE=my-release-key.keystore
MYAPP_UPLOAD_KEY_ALIAS=my-key-alias
MYAPP_UPLOAD_STORE_PASSWORD=*****
MYAPP_UPLOAD_KEY_PASSWORD=*****
```

→ **이 파일 .gitignore 처리** 또는 환경 변수.

`android/app/build.gradle`:
```gradle
android {
  ...
  signingConfigs {
    release {
      if (project.hasProperty('MYAPP_UPLOAD_STORE_FILE')) {
        storeFile file(MYAPP_UPLOAD_STORE_FILE)
        storePassword MYAPP_UPLOAD_STORE_PASSWORD
        keyAlias MYAPP_UPLOAD_KEY_ALIAS
        keyPassword MYAPP_UPLOAD_KEY_PASSWORD
      }
    }
  }
  buildTypes {
    release {
      signingConfig signingConfigs.release
      minifyEnabled true
      shrinkResources true
      proguardFiles ...
    }
  }
}
```

## 4. 3. 빌드

```bash
cd android
./gradlew bundleRelease       # AAB (Play Store 용)
./gradlew assembleRelease     # APK
```

결과:
- AAB: `android/app/build/outputs/bundle/release/app-release.aab`
- APK: `android/app/build/outputs/apk/release/app-release.apk`

## 5. 4. 로컬에서 release 테스트

```bash
# release APK 를 디바이스에 설치
adb install android/app/build/outputs/apk/release/app-release.apk
```

→ debug 와 다른 동작 확인 (ProGuard, env 변수).

## 6. Play Console 업로드

1. **play.google.com/console** → 앱 등록.
2. **Store presence** — 앱 이름 / 설명 / 스크린샷.
3. **Production / Internal Testing / Closed Testing** 트랙 선택.
4. **AAB 업로드**.
5. **출시 검토** → 제출.

### 첫 출시
- Closed Testing → Open Testing → Production 순서 권장.
- 모든 단계마다 review.

## 7. 앱 아이콘

`android/app/src/main/res/`:
```
mipmap-mdpi/ic_launcher.png       (48×48)
mipmap-hdpi/ic_launcher.png       (72×72)
mipmap-xhdpi/ic_launcher.png      (96×96)
mipmap-xxhdpi/ic_launcher.png     (144×144)
mipmap-xxxhdpi/ic_launcher.png    (192×192)
```

### 자동 생성
```bash
yarn add -g app-icon
app-icon generate -i icon.png --platforms=android
```

### Adaptive Icon (Android 8+)
- `mipmap-anydpi-v26/ic_launcher.xml` + foreground + background layer.
- Android Studio 의 Image Asset Studio 사용.

## 8. 앱 이름

`android/app/src/main/res/values/strings.xml`:
```xml
<resources>
  <string name="app_name">취업답변</string>
</resources>
```

→ 한국어 OK.

## 9. SplashScreen

- `react-native-splash-screen` 또는 `react-native-bootsplash`.
- 또는 `android/app/src/main/res/drawable/launch_screen.xml` 직접.

자세히 [[../configuration|configuration]].

## 10. ProGuard / R8

```gradle
buildTypes {
  release {
    minifyEnabled true                       // 코드 축소
    shrinkResources true                      // 리소스 축소
    proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
  }
}
```

→ APK / AAB 크기 줄임. **native lib 와 충돌 가능** → `proguard-rules.pro` 의 keep rule.

```
# proguard-rules.pro
-keep class com.facebook.react.** { *; }
-keep class com.kakao.** { *; }
-keep class com.example.app.** { *; }
```

## 11. 빌드 변형 — flavors

```gradle
android {
  flavorDimensions "env"
  productFlavors {
    dev {
      dimension "env"
      applicationIdSuffix ".dev"            // 다른 package
      versionNameSuffix "-dev"
    }
    prod {
      dimension "env"
    }
  }
}
```

```bash
./gradlew assembleDevRelease
./gradlew assembleProdRelease
```

→ dev / stage / prod 의 다른 빌드.

## 12. 환경 변수 — production

```bash
# .env.master
API_URL=https://api.example.com
KAKAO_NATIVE_KEY=...

# react-native-config 사용 시
ENVFILE=.env.master ./gradlew bundleRelease
```

→ release 빌드 시 dev env 가 들어가면 큰 문제.

## 13. job-answer-app-rn 빌드 단계 추정

```bash
# 1. env 변경
ENVFILE=.env.master

# 2. version 올리기
# android/app/build.gradle 의 versionCode++, versionName 변경

# 3. clean
cd android && ./gradlew clean

# 4. 빌드
./gradlew bundleRelease

# 5. Play Console 업로드
# (Fastlane / 수동)
```

## 14. Fastlane — 자동화

```bash
cd android
bundle init
echo 'gem "fastlane"' >> Gemfile
bundle install

fastlane init
```

`android/fastlane/Fastfile`:
```ruby
default_platform(:android)

platform :android do
  desc "Submit a new build to Play Internal track"
  lane :internal do
    gradle(task: "clean")
    gradle(task: "bundle", build_type: "Release")
    upload_to_play_store(
      track: "internal",
      aab: "app/build/outputs/bundle/release/app-release.aab"
    )
  end
end
```

```bash
fastlane internal
```

→ 1 명령어로 빌드 + 업로드.

## 15. 흔한 빌드 에러

### `Could not find tools.jar`
- JDK 17 안 잡힘. `JAVA_HOME`.

### `SDK location not found`
- `android/local.properties` 의 `sdk.dir`.

### `Duplicate class`
- 의존성 충돌. `./gradlew app:dependencies` 로 분석.

### ProGuard 후 native crash
- `proguard-rules.pro` 의 `-keep` 추가.

### `Failed to install on emulator`
- 디바이스 storage 부족 또는 옛 버전 설치돼 있음.

## 16. 함정

1. **keystore 분실** — 새 앱 등록 + 사용자 재설치 필요.
2. **versionCode 안 올림** — Play Store 거절.
3. **debug env 의 release** — production 의 dev 값 노출.
4. **ProGuard 의 native crash** — keep rule 누락.
5. **AAB 의 size 큼** — 사용 안 하는 SDK / asset 제거.
6. **64-bit 안 빌드** — Play Store 64-bit 필수.

## 17. 외부 자료

- [Android — Signing your app](https://developer.android.com/studio/publish/app-signing)
- [Android — App Bundle](https://developer.android.com/guide/app-bundle)
- [Play Console](https://play.google.com/console)
- [Fastlane Android](https://docs.fastlane.tools/getting-started/android/setup/)

## 18. 관련

- [[deployment]]
- [[build-ios]]
- [[../configuration]]
