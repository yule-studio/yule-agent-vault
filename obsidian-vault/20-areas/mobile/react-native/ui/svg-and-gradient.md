---
title: "SVG + Gradient — react-native-svg + linear-gradient"
kind: knowledge
project: mobile
agent: engineering-agent/tech-lead
status: current
tags: [mobile, react-native, ui, svg, gradient, blur]
---

# SVG + Gradient

**[[ui|↑ UI Hub]]**

> 커스텀 그래픽 / 그라데이션 / 블러.

## 1. react-native-svg — SVG 표시

```bash
yarn add react-native-svg
cd ios && pod install
```

### 1.1 SVG 컴포넌트 직접 작성

```tsx
import Svg, { Circle, Rect, Path, G } from 'react-native-svg';

<Svg width="100" height="100" viewBox="0 0 100 100">
  <Circle cx="50" cy="50" r="40" stroke="blue" strokeWidth="2.5" fill="green" />
  <Rect x="10" y="10" width="20" height="20" fill="red" />
  <Path d="M 10 10 L 90 90" stroke="black" />
</Svg>
```

### 1.2 SVG 파일을 컴포넌트처럼 import

```bash
yarn add -D react-native-svg-transformer
```

`metro.config.js`:
```js
const { getDefaultConfig, mergeConfig } = require('@react-native/metro-config');

const defaultConfig = getDefaultConfig(__dirname);
const { assetExts, sourceExts } = defaultConfig.resolver;

module.exports = mergeConfig(defaultConfig, {
  transformer: {
    babelTransformerPath: require.resolve('react-native-svg-transformer'),
  },
  resolver: {
    assetExts: assetExts.filter(ext => ext !== 'svg'),
    sourceExts: [...sourceExts, 'svg'],
  },
});
```

`declaration.d.ts`:
```ts
declare module '*.svg' {
  import { SvgProps } from 'react-native-svg';
  const content: React.FC<SvgProps>;
  export default content;
}
```

```tsx
import Logo from '@assets/svg/logo.svg';

<Logo width={120} height={40} fill="black" />
```

→ 디자이너의 SVG 그대로 사용. fill / stroke prop 으로 색 변경.

### 1.3 SVG 문자열 — SvgXml

```tsx
import { SvgXml } from 'react-native-svg';

const xml = `<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24"><path d="..." /></svg>`;

<SvgXml xml={xml} width={24} height={24} fill="#0064FF" />
```

→ 동적으로 SVG 문자열 받아 표시.

### 1.4 SVG 변환 도구

- **SVGR Playground** (https://react-svgr.com/playground/) — 미리 변환 시도.
- **SVG fill="currentColor"** — JSX 변환 시 `fill={props.fill}` 로 자동 매핑되는 경우 있음. 디자이너에게 currentColor 사용 요청.

## 2. linear-gradient

```bash
yarn add react-native-linear-gradient
cd ios && pod install
```

```tsx
import LinearGradient from 'react-native-linear-gradient';

<LinearGradient
  colors={['#FF6B00', '#FFB800']}        // 색 배열
  start={{ x: 0, y: 0 }}                  // 시작점 (0,0 = 좌상)
  end={{ x: 1, y: 1 }}                    // 끝점 (1,1 = 우하)
  style={{
    padding: 16,
    borderRadius: 12,
  }}
>
  <Text style={{ color: 'white', fontWeight: 'bold' }}>그라데이션</Text>
</LinearGradient>
```

### 자주 쓰는 방향

| | start | end |
| --- | --- | --- |
| 좌 → 우 | `{0,0}` | `{1,0}` |
| 상 → 하 | `{0,0}` | `{0,1}` |
| 대각 | `{0,0}` | `{1,1}` |

### locations — 색 위치 비율

```tsx
<LinearGradient
  colors={['red', 'yellow', 'blue']}
  locations={[0, 0.3, 1]}            // 30% 위치에 yellow
  style={...}
/>
```

### 버튼 + gradient

```tsx
<Pressable onPress={...}>
  <LinearGradient
    colors={['#0064FF', '#0084FF']}
    start={{ x: 0, y: 0 }}
    end={{ x: 1, y: 0 }}
    style={{ paddingVertical: 14, borderRadius: 8, alignItems: 'center' }}
  >
    <Text style={{ color: 'white', fontWeight: '600' }}>로그인</Text>
  </LinearGradient>
</Pressable>
```

## 3. radial / sweep gradient

→ `react-native-linear-gradient` 는 linear 만.
radial / sweep 은:
```bash
yarn add react-native-svg
```

```tsx
import Svg, { Defs, RadialGradient, Stop, Circle } from 'react-native-svg';

<Svg width="200" height="200">
  <Defs>
    <RadialGradient id="grad" cx="50%" cy="50%" r="50%">
      <Stop offset="0%" stopColor="yellow" />
      <Stop offset="100%" stopColor="red" />
    </RadialGradient>
  </Defs>
  <Circle cx="100" cy="100" r="100" fill="url(#grad)" />
</Svg>
```

## 4. blur — @react-native-community/blur

```bash
yarn add @react-native-community/blur
cd ios && pod install
```

```tsx
import { BlurView } from '@react-native-community/blur';

<View style={{ flex: 1 }}>
  <Image source={bg} style={StyleSheet.absoluteFillObject} />
  <BlurView
    style={StyleSheet.absoluteFillObject}
    blurType="light"            // 'xlight' | 'light' | 'dark'
    blurAmount={10}
    reducedTransparencyFallbackColor="white"
  />
  <Text>블러 위에 텍스트</Text>
</View>
```

### iOS Glassmorphism 효과
```tsx
<BlurView blurType="dark" blurAmount={20}>
  <Text style={{ color: 'white' }}>...</Text>
</BlurView>
```

→ Android 는 일부 device 에서 blur 성능 떨어짐. fallback 색.

## 5. shadow / elevation (참고)

→ `react-native` 내장. SVG / gradient 아님. 자세히 [[../core-concepts/stylesheet-and-flexbox]].

## 6. job-answer-app-rn 의 패턴

```tsx
// components/SimplifiedSvgIcon
import { SvgXml } from 'react-native-svg';
import { iconMap } from '@/constants/icons';

export const SimplifiedSvgIcon = ({ name, size = 24, color }) => (
  <SvgXml
    xml={iconMap[name]}
    width={size}
    height={size}
    fill={color}
  />
);
```

```tsx
// Splash / Background — gradient
<LinearGradient
  colors={['#0064FF', '#003ECC']}
  style={StyleSheet.absoluteFillObject}
>
  ...
</LinearGradient>
```

```tsx
// Modal backdrop — blur (iOS only)
{Platform.OS === 'ios' ? (
  <BlurView blurType="dark" blurAmount={10} style={StyleSheet.absoluteFillObject} />
) : (
  <View style={[StyleSheet.absoluteFillObject, { backgroundColor: 'rgba(0,0,0,0.5)' }]} />
)}
```

## 7. 함정

1. **svg-transformer 설정 누락** — `.svg` import 실패. metro.config.js 확인.
2. **TypeScript 의 .svg declaration 누락** — `Cannot find module '...svg'`. declaration.d.ts.
3. **gradient 의 native module** — pod install 누락.
4. **BlurView 의 Android 성능** — 큰 영역에서 fps 낮음. fallback.
5. **SVG 의 fill / stroke 의 currentColor** — 디자이너 SVG 가 hardcode 색 → prop 으로 변경 X. SVG 편집 필요.
6. **재귀 SVG (defs / use)** — react-native-svg 가 일부 미지원. 평탄화.

## 8. 다음 단계

- [[lottie]]
- [[vector-icons]]

## 9. 외부 자료

- [react-native-svg](https://github.com/software-mansion/react-native-svg)
- [react-native-svg-transformer](https://github.com/kristerkari/react-native-svg-transformer)
- [react-native-linear-gradient](https://github.com/react-native-linear-gradient/react-native-linear-gradient)
- [@react-native-community/blur](https://github.com/Kureev/react-native-blur)

## 10. 관련

- [[ui]]
- [[vector-icons]]
- [[lottie]]
