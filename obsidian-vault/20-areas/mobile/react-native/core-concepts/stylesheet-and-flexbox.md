---
title: "StyleSheet + Flexbox — RN 의 스타일 시스템"
kind: knowledge
project: mobile
agent: engineering-agent/tech-lead
status: current
tags: [mobile, react-native, stylesheet, flexbox, beginner]
---

# StyleSheet + Flexbox

**[[core-concepts|↑ Core Concepts]]**

> RN 의 모든 스타일. CSS 와 비슷하지만 **객체 + 숫자 단위 + 다른 default**.

## 1. StyleSheet.create — 기본

```tsx
import { StyleSheet, View, Text } from 'react-native';

export default function Card() {
  return (
    <View style={styles.container}>
      <Text style={styles.title}>제목</Text>
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    padding: 16,
    backgroundColor: 'white',
    borderRadius: 8,
  },
  title: {
    fontSize: 18,
    fontWeight: 'bold',
    color: '#1A1A1A',
  },
});
```

### `StyleSheet.create` vs 인라인 객체

```tsx
// ✅ create (권장)
<View style={styles.container} />

// 인라인 (간단할 때)
<View style={{ padding: 16, backgroundColor: 'white' }} />
```

→ create 가 더 성능 + IDE 자동완성. 동적 값은 인라인 또는 함수.

## 2. CSS 와 RN 의 단위 차이

| | CSS | RN |
| --- | --- | --- |
| 단위 | `16px`, `1rem`, `50%` | 숫자만 (= dp), 또는 `'50%'` |
| color | `red`, `#ff0000` | 동일 |
| margin / padding | shorthand `10px 20px` | 객체 prop 만 (`marginHorizontal`, `paddingVertical`) |
| border shorthand | `1px solid red` | 분리 (`borderWidth`, `borderColor`) |

```tsx
{ padding: 16 }                    // 16dp
{ paddingHorizontal: 16 }          // 좌우 16
{ paddingVertical: 8 }             // 위아래 8
{ width: '50%' }                   // 부모의 50%
{ borderWidth: 1, borderColor: '#ddd' }   // 분리
```

→ dp = density-independent pixel. 디바이스 dpi 무관 같은 크기.

## 3. Flexbox — RN 의 핵심

### web 과의 default 차이

| | CSS web | RN |
| --- | --- | --- |
| `display` | 기본 block | 모든 View 가 자동 flex |
| `flex-direction` | `row` 기본 | **`column` 기본** ← 큰 차이 |
| `flex` | 비율 | 동일 (flex: 1 이 100%) |

### 가장 자주 쓰는 layout

```tsx
// 세로 중앙
{ flex: 1, alignItems: 'center', justifyContent: 'center' }

// 가로로 늘어놓기
{ flexDirection: 'row' }

// 양쪽 끝
{ flexDirection: 'row', justifyContent: 'space-between', alignItems: 'center' }

// 줄바꿈
{ flexDirection: 'row', flexWrap: 'wrap' }

// 간격
{ flexDirection: 'row', gap: 8 }    // RN 0.71+
```

### Flexbox prop 정리

| prop | 값 |
| --- | --- |
| `flexDirection` | `column` (기본) / `row` / `column-reverse` / `row-reverse` |
| `justifyContent` | `flex-start` / `center` / `flex-end` / `space-between` / `space-around` / `space-evenly` |
| `alignItems` | `flex-start` / `center` / `flex-end` / `stretch` (기본) |
| `alignSelf` | 자식이 자기 정렬 override |
| `flex` | 비율 (1 = 모든 공간 차지) |
| `flexWrap` | `nowrap` (기본) / `wrap` |
| `gap` | item 간 간격 (RN 0.71+) |

### `flex: 1` 의 의미
```tsx
<View style={{ flex: 1 }}>
  // 부모의 가용 공간 100% 차지
</View>

<View style={{ flex: 1 }}>
  <View style={{ flex: 1 }}>       // 부모 (= 전체) 의 절반
  <View style={{ flex: 2 }}>       // 절반의 2배 = 전체의 2/3
</View>
```

## 4. dimensions / 화면 크기

```tsx
import { Dimensions } from 'react-native';

const { width, height } = Dimensions.get('window');
// width: iPhone 14 = 390, iPad = 1024 등
```

### 반응형
```tsx
const isTablet = width >= 768;
<View style={{ padding: isTablet ? 32 : 16 }} />
```

### 이벤트 — 회전 / 윈도우 크기 변경
```tsx
useEffect(() => {
  const sub = Dimensions.addEventListener('change', ({ window }) => {
    console.log(window.width);
  });
  return () => sub.remove();
}, []);
```

→ 또는 `useWindowDimensions` hook:
```tsx
import { useWindowDimensions } from 'react-native';
const { width } = useWindowDimensions();
```

## 5. 폰트

```tsx
{ fontSize: 16 }
{ fontWeight: 'bold' }        // '100' ~ '900' | 'normal' | 'bold'
{ fontStyle: 'italic' }
{ fontFamily: 'Pretendard-Bold' }     // 등록된 폰트
{ lineHeight: 24 }
{ letterSpacing: 0.5 }
{ textAlign: 'center' }       // 'left' | 'right' | 'justify' (Android)
{ textDecorationLine: 'underline' }
```

### 폰트 등록 (Bare RN)
1. `src/assets/fonts/` 에 `.ttf` / `.otf` 추가.
2. `react-native.config.js`:
```js
assets: ['./src/assets/fonts'],
```
3. `npx react-native-asset` (또는 직접 link).
4. `cd ios && pod install`.
5. 사용:
```tsx
<Text style={{ fontFamily: 'Pretendard-Bold' }}>안녕</Text>
```

→ iOS 는 .ttf 의 PostScript name. Android 는 파일명. 둘 다 동일 권장.

## 6. shadow / elevation

```tsx
// iOS
{
  shadowColor: '#000',
  shadowOffset: { width: 0, height: 2 },
  shadowOpacity: 0.1,
  shadowRadius: 4,
}

// Android
{ elevation: 4 }

// 둘 다
import { Platform } from 'react-native';
...Platform.select({
  ios: { shadowColor: '#000', shadowOpacity: 0.1, shadowOffset: { width: 0, height: 2 }, shadowRadius: 4 },
  android: { elevation: 4 },
}),
```

→ shadow 는 iOS 만 자동, Android 는 elevation 별도.

## 7. border / border-radius

```tsx
{
  borderWidth: 1,
  borderColor: '#ddd',
  borderRadius: 8,
}

// 각 모서리
{
  borderTopLeftRadius: 8,
  borderTopRightRadius: 8,
}

// 각 변 (드물게 사용)
{
  borderTopWidth: 1,
  borderBottomWidth: 0,
}
```

## 8. background / overflow

```tsx
{ backgroundColor: '#0064FF' }
{ backgroundColor: 'rgba(0,0,0,0.5)' }
{ backgroundColor: 'transparent' }

{ overflow: 'hidden' }        // 자식이 부모 밖으로 안 나감
```

## 9. position

```tsx
// 기본 (Flexbox)
{ position: 'relative' }

// 절대 위치 (부모 안)
{ position: 'absolute', top: 0, right: 0 }

// 화면 전체 absolute
import { StyleSheet } from 'react-native';
<View style={[StyleSheet.absoluteFillObject, { backgroundColor: 'red' }]} />
// = { position: 'absolute', top: 0, left: 0, right: 0, bottom: 0 }
```

## 10. transform — 회전 / 이동 / 확대

```tsx
{
  transform: [
    { translateX: 10 },
    { translateY: -5 },
    { rotate: '45deg' },
    { scale: 1.2 },
  ]
}
```

→ animation 과 결합 (Animated / Reanimated).

## 11. 스타일 배열 / 조건부

```tsx
// 배열 — 뒤가 앞을 덮음
<View style={[styles.button, styles.primary, { opacity: 0.8 }]} />

// 조건부
<View style={[styles.button, isActive && styles.active]} />

// false 값 무시됨 — null / undefined / false OK
```

## 12. theme — 디자인 토큰

```ts
// theme.ts
export const theme = {
  color: {
    primary: '#0064FF',
    text: '#1A1A1A',
    bg: '#FFFFFF',
    border: '#E5E5E5',
  },
  space: { xs: 4, sm: 8, md: 16, lg: 24, xl: 32 },
  font: { sm: 12, md: 14, lg: 16, xl: 20 },
  radius: { sm: 4, md: 8, lg: 16 },
};

// 사용
import { theme } from '@/constants/theme';

const styles = StyleSheet.create({
  card: {
    padding: theme.space.md,
    backgroundColor: theme.color.bg,
    borderRadius: theme.radius.md,
  },
});
```

→ `job-answer-app-rn/src/constants/` 에 보통 정의.

## 13. styled-components / Tailwind 대응?

```bash
yarn add styled-components
# 또는
yarn add nativewind             # Tailwind 의 RN 버전
```

```tsx
// styled-components/native
import styled from 'styled-components/native';

const Button = styled.Pressable`
  padding: 12px 24px;
  background: ${p => p.primary ? '#0064FF' : '#fff'};
  border-radius: 8px;
`;
```

```tsx
// NativeWind (Tailwind for RN)
<View className="p-4 bg-white rounded-lg">
  <Text className="text-lg font-bold">Hello</Text>
</View>
```

→ job-answer-app-rn 은 **StyleSheet 만** 사용 (외부 lib 없음).

## 14. RN 의 CSS 미지원 — 흔한 함정

| CSS | RN 대응 |
| --- | --- |
| `display: grid` | ❌ — 직접 layout |
| `transition` / `animation` (CSS) | Animated API 또는 Reanimated |
| `pseudo-class :hover` | onPressIn / onPressOut |
| `@media` | Dimensions / useWindowDimensions |
| `box-shadow` | shadow* (iOS) + elevation (Android) |
| `cursor` | RN-Web 만. native 없음 |
| `z-index` | 지원 (단 같은 stack 안만) |

## 15. 디버깅 — Layout Inspector

```
Dev Menu (Cmd + D 또는 Cmd + M) → Show Inspector
```

→ 화면 element 클릭 시 위치 / 크기 / 스타일 확인.

또는 React DevTools.

## 16. 함정

1. **px 단위** — 작동 안 함. 숫자만.
2. **`flex-direction: row` 가 기본인 줄 알고** — column 이 기본. row 명시 필요.
3. **margin 의 음수** — RN 일부 환경에서 제약. padding 권장.
4. **shadow 가 Android 에서 안 보임** — `elevation` 필요.
5. **`%` 의 부모 의존** — 부모 height 가 정해지지 않으면 `'50%'` 가 0.
6. **gap 의 RN 버전 미지원** — 0.71 미만은 `margin*` 으로 spacing.
7. **StyleSheet.create 의 동적 값** — 정적만. 동적은 인라인 또는 함수.

## 17. 다음 단계

- [[platform-differences]] — iOS/Android 분기
- [[safe-area-and-status-bar]]

## 18. 관련

- [[core-concepts]]
- [[rn-components]]
- [[../../../frontend/react/styling/styling|web 의 styling]]
