---
title: "Lottie — lottie-react-native 애니메이션"
kind: knowledge
project: mobile
agent: engineering-agent/tech-lead
status: current
tags: [mobile, react-native, ui, lottie, animation]
---

# Lottie

**[[ui|↑ UI Hub]]**

> After Effects 의 애니메이션을 JSON 으로 export → RN 에서 native 60fps 렌더.

## 1. 설치

```bash
yarn add lottie-react-native
cd ios && pod install
```

## 2. 기본 사용

```tsx
import LottieView from 'lottie-react-native';

<LottieView
  source={require('@assets/lottie/loading.json')}
  autoPlay
  loop
  style={{ width: 200, height: 200 }}
/>
```

### URL 에서 로딩
```tsx
<LottieView
  source={{ uri: 'https://example.com/loading.json' }}
  autoPlay
  loop
/>
```

## 3. 제어 — ref

```tsx
const ref = useRef<LottieView>(null);

useEffect(() => {
  ref.current?.play();           // 재생
  ref.current?.pause();
  ref.current?.reset();
  ref.current?.play(30, 90);     // frame 30~90 만
}, []);

<LottieView ref={ref} source={...} />
```

## 4. autoPlay vs ref

```tsx
// 자동 재생 + 무한 루프 (로딩 스피너)
<LottieView source={...} autoPlay loop />

// 한 번만 재생 (성공 체크)
<LottieView source={...} autoPlay loop={false} onAnimationFinish={() => navigate('Next')} />

// 수동 제어
const ref = useRef<LottieView>(null);
<LottieView ref={ref} source={...} loop={false} />
// 어떤 이벤트에 대해 ref.current?.play()
```

## 5. progress — 부분 재생 / scrub

```tsx
const [progress, setProgress] = useState(0);
<LottieView source={...} progress={progress} />

// progress 를 Animated value 로 연동 — drag 등
```

## 6. speed

```tsx
<LottieView source={...} autoPlay loop speed={1.5} />   // 1.5배
```

## 7. 색 변경 — colorFilters

```tsx
<LottieView
  source={...}
  colorFilters={[
    { keypath: 'Layer 1', color: '#FF0000' },
    { keypath: 'Background', color: '#0064FF' },
  ]}
/>
```

→ `keypath` 는 After Effects 의 layer 이름. 디자이너에게 layer 이름 확인.

## 8. 어디서 얻을까?

- **LottieFiles** (https://lottiefiles.com/) — 무료 + 유료 라이브러리.
- **IconScout** (https://iconscout.com/) — Lottie 아이콘.
- **자체 제작** — After Effects → Bodymovin plugin → JSON.

→ 다운로드 시 size 확인. 너무 크면 부담 (수MB).

## 9. 흔한 사용 케이스

### Loading 스피너
```tsx
<LottieView source={require('@assets/lottie/spinner.json')} autoPlay loop />
```

### Splash screen
```tsx
function Splash() {
  const ref = useRef<LottieView>(null);

  return (
    <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
      <LottieView
        ref={ref}
        source={require('@assets/lottie/splash.json')}
        autoPlay
        loop={false}
        onAnimationFinish={() => navigation.replace('MainTab')}
      />
    </View>
  );
}
```

### Empty state
```tsx
<View>
  <LottieView source={require('@assets/lottie/empty.json')} autoPlay loop style={{ width: 200, height: 200 }} />
  <Text>아직 내용이 없어요</Text>
</View>
```

### Success / Error 애니메이션
```tsx
<LottieView source={require('@assets/lottie/check.json')} autoPlay loop={false} />
```

## 10. job-answer-app-rn 패턴

```tsx
// components/RecordingButton — 녹음 중 lottie 펄스
<LottieView
  source={require('@assets/lottie/recording.json')}
  autoPlay={isRecording}
  loop
  style={{ width: 80, height: 80 }}
/>

// 또는 LoaderKit 의 lottie 기반 loader
import LoaderKit from 'react-native-loader-kit';
<LoaderKit name="BallPulse" size={40} color="#0064FF" />
```

## 11. dotLottie 포맷

```bash
# .lottie 파일 (압축된 lottie + 메타)
yarn add @lottiefiles/dotlottie-react-native
```

→ JSON 보다 30%+ 작음. 신규 추천.

## 12. 함정

1. **JSON 파일 크기 큼** — 수MB 의 lottie 가 bundle 폭증. 최적화 (https://lottiefiles.com/lottie-optimizer).
2. **autoPlay + loop={false}** — 한 번만 재생. `onAnimationFinish`.
3. **iOS 의 `pod install`** 누락.
4. **Android 의 hardware acceleration** — 가끔 깜빡임. `renderMode="HARDWARE"` 또는 `SOFTWARE`.
5. **dynamic source** — `source={{ uri }}` 의 변경. ref reset.
6. **memory leak** — 큰 lottie 의 useEffect cleanup. pause + reset.

## 13. 외부 자료

- [lottie-react-native](https://github.com/lottie-react-native/lottie-react-native)
- [LottieFiles](https://lottiefiles.com/)
- [Bodymovin plugin](https://aescripts.com/bodymovin/)

## 14. 관련

- [[ui]]
- [[svg-and-gradient]]
- [[../performance/reanimated-gesture]]
