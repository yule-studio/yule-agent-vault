---
title: "Metro Cache / 빌드 문제 — 트러블슈팅"
kind: knowledge
project: mobile
agent: engineering-agent/tech-lead
status: current
tags: [mobile, react-native, metro, cache, troubleshooting, build]
---

# Metro Cache / 빌드 문제

**[[pitfalls|↑ Pitfalls Hub]]**

> RN 의 80% 빌드 문제 = 캐시. 안 풀리면 단계적 reset.

## 1. Metro cache reset

```bash
yarn start --reset-cache
# 또는
npx react-native start --reset-cache
```

→ JS 변경 안 반영 시 첫 시도.

## 2. Watchman 캐시

```bash
watchman watch-del-all
```

→ 파일 watcher 의 stale state.

## 3. node_modules 재설치

```bash
rm -rf node_modules
yarn install         # 또는 npm i
```

→ peer dependency 충돌 시.

## 4. iOS Pods reinstall

```bash
cd ios
rm -rf Pods Podfile.lock
pod install
cd ..
```

→ native lib 추가 / 업그레이드 후.

### `pod deintegrate`
```bash
cd ios
pod deintegrate
pod install
```

→ Multiple commands produce 류 에러.

## 5. Android gradle clean

```bash
cd android
./gradlew clean
cd ..
```

또는 build 디렉토리 직접:
```bash
rm -rf android/app/build
rm -rf android/build
rm -rf android/.gradle
```

## 6. Xcode DerivedData

```
~/Library/Developer/Xcode/DerivedData/
```

→ Xcode 의 cache. 가끔 stale.

```bash
rm -rf ~/Library/Developer/Xcode/DerivedData
```

## 7. "원자폭탄" — 전체 reset

```bash
# 1. 모든 cache 삭제
watchman watch-del-all
rm -rf node_modules
rm -rf $TMPDIR/metro-*
rm -rf $TMPDIR/haste-map-*
rm -rf $TMPDIR/react-*
rm -rf ios/Pods ios/Podfile.lock ios/build
rm -rf ~/Library/Developer/Xcode/DerivedData
rm -rf android/app/build android/build android/.gradle

# 2. 재설치
yarn install
cd ios && pod install && cd ..

# 3. 빌드
yarn start --reset-cache
# 별 터미널
yarn ios          # 또는 yarn android
```

→ 거의 모든 문제 해결.

## 8. 흔한 에러 → 해결

### `Unable to resolve module`
```bash
yarn start --reset-cache
```

원인:
- 새 dep 추가 후 Metro 가 못 봄.
- alias 가 babel.config / tsconfig 불일치.
- 파일 이름 case 차이 (Mac case-insensitive, Linux CI case-sensitive).

### `Multiple commands produce`
```bash
cd ios && pod deintegrate && pod install
```

→ 같은 파일 두 곳에서 build. 보통 Pods 의 충돌.

### `Could not connect to development server`
```bash
yarn start
# 또는 metro 가 다른 port 면
yarn start --port 8082
```

iOS:
```bash
# Mac 의 firewall 또는 port 충돌
lsof -i :8081
```

### `tools.jar not found` (Android)
```bash
echo $JAVA_HOME
# JDK 17 의 경로 확인
export JAVA_HOME=/Library/Java/JavaVirtualMachines/zulu-17.jdk/Contents/Home
```

### `SDK location not found` (Android)
`android/local.properties` 생성:
```
sdk.dir=/Users/yourname/Library/Android/sdk
```

### `Hermes is not enabled`
```ruby
# ios/Podfile
use_react_native!(
  :hermes_enabled => true,    # 또는 false
)
```

```gradle
// android/app/build.gradle
hermesEnabled = true
```

### TypeError: Cannot read property of undefined (NativeModule)
- pod install 누락 또는 autolink 실패.
- iOS — `xcodebuild` 로 직접 빌드 + 에러 확인.
- Android — `./gradlew app:assembleDebug --info`.

### `Stuck on splash screen`
- JS bundle 못 load. Metro 안 돔.
- `yarn start` 켜져 있는지 확인.
- 디바이스 / emulator 가 Metro 의 IP 접근 가능한지.

## 9. dev mode 의 reload

| | 단축키 |
| --- | --- |
| iOS sim | `Cmd + R` (reload), `Cmd + D` (dev menu) |
| Android emulator | `R, R` (두 번), `Cmd + M` |
| 실기기 | shake to dev menu |

### Fast Refresh
- 자동 — 코드 변경 시 즉시 반영.
- 끄기: Dev Menu → Disable Fast Refresh.

### Reload 가 안 될 때
```bash
# Metro 끄고 다시
yarn start --reset-cache

# 또는 앱 강제 종료 후 재실행
```

## 10. 환경 변수 안 들어옴

```bash
# .env 변경 후
yarn start --reset-cache
# react-native-dotenv 의 캐시 reset
```

→ `metro-config` 의 babel cache 도 고려.

## 11. Hermes 디버깅

```bash
# Hermes JS 엔진 사용 중이면 — Chrome DevTools 의 JS debugger 일부 다름
# yarn start 후 j 키 → Hermes inspector 열림 (Chrome 자동)
```

## 12. CI 의 문제

CI 에서 캐시 / 환경 다름:
- `node_modules` 캐시 — version mismatch.
- `pod` 캐시 — CocoaPods spec repo.
- `gradle` 캐시.
- Mac runner 필요 (iOS).

→ CI 에서는 항상 fresh install + 모든 cache 재생성.

## 13. node version

```bash
# 권장
nvm use 20

# .nvmrc 추가
echo "20" > .nvmrc
```

→ 팀원 / CI 가 같은 node 사용.

## 14. 함정

1. **`--reset-cache` 만으로 해결 안 됨** — Pods / gradle 도 reset 필요.
2. **`rm -rf node_modules` 잊고 yarn upgrade** — 불완전 dep tree.
3. **Xcode DerivedData 의 끔찍한 캐시** — 가끔 모든 게 깨끗해도 Xcode 만 stale.
4. **Watchman 의 stale watch** — Mac 의 가장 음흉한 cache.
5. **Metro port conflict** — 다른 process 가 8081 점유. `lsof -i :8081`.
6. **시뮬레이터 시간** — 시뮬레이터 clock 이 PC 와 다름. JWT 만료 등.

## 15. 진단 명령

```bash
# 환경 일괄 확인
npx react-native doctor

# RN version 확인
cat node_modules/react-native/package.json | grep version

# pod 버전 확인
cd ios && pod outdated
```

## 16. 외부 자료

- [reactnative.dev — Troubleshooting](https://reactnative.dev/docs/troubleshooting)
- [Metro 공식](https://metrobundler.dev/)

## 17. 관련

- [[pitfalls]]
- [[platform-bugs]]
- [[../configuration]]
