---
title: "UI Libraries in RN — vector-icons / svg / lottie / splash / spotlight"
kind: knowledge
project: mobile
agent: engineering-agent/tech-lead
status: current
tags: [mobile, react-native, ui, hub]
---

# UI Libraries in RN

**[[../react-native|↑ RN Hub]]**

> RN 의 UI 라이브러리. job-answer-app-rn 기준 + 일반 옵션.

## 1. 카테고리

| | 라이브러리 |
| --- | --- |
| **아이콘** | `react-native-vector-icons`, `@expo/vector-icons` |
| **SVG** | `react-native-svg`, `react-native-svg-transformer` |
| **그라데이션** | `react-native-linear-gradient` |
| **블러** | `@react-native-community/blur` |
| **로티 애니메이션** | `lottie-react-native` |
| **스플래시** | `react-native-splash-screen`, `react-native-bootsplash` |
| **온보딩 / 스포트라이트** | `react-native-spotlight-tour`, `react-native-app-intro-slider` |
| **모달 / Bottom Sheet** | `@gorhom/bottom-sheet` |
| **UI 키트** | `react-native-paper` (Material), `react-native-elements`, `tamagui`, `nativewind` (Tailwind) |
| **차트** | `react-native-svg-charts`, `victory-native`, `react-native-gifted-charts` |
| **이미지 캐싱** | `react-native-fast-image` |

## 2. job-answer-app-rn 의 UI lib

```json
"react-native-vector-icons": "^10.3.0",
"react-native-svg": "^15.12.1",
"react-native-linear-gradient": "^2.8.3",
"@react-native-community/blur": "^4.4.1",
"lottie-react-native": "^7.3.4",
"react-native-splash-screen": "^3.3.0",
"react-native-spotlight-tour": "^4.0.0",
"react-native-loader-kit": "^3.0.0"
```

→ **UI 키트 없이 직접 컴포넌트** + 작은 lib 조합.

## 3. UI 키트 — 도입 여부

| | 장점 | 단점 |
| --- | --- | --- |
| **사용 X (직접)** | 자유 + 가볍다 | 작성량 많음 |
| **react-native-paper** | Material 완성도 | 디자인 변경 비용 |
| **nativewind** | Tailwind 의 RN | 동적 클래스 제약 |
| **tamagui** | 다중 플랫폼 + 빠름 | 학습 곡선 |
| **gluestack-ui** | 헤드리스 + 유연 | 신생 |

→ job-answer-app-rn = **직접 + 작은 lib**. 신규 프로젝트 검토 권장.

## 4. 학습 우선순위

1. **[[vector-icons]]** — 아이콘 (탭, 버튼).
2. **[[svg-and-gradient]]** — 커스텀 그래픽.
3. **[[lottie]]** — 애니메이션 (splash, success).
4. **splash-screen 설정** — 자세히 [[../configuration|configuration]].
5. **bottom sheet / 모달** — `@gorhom/bottom-sheet`.

## 5. job-answer-app-rn 의 흔한 컴포넌트 패턴

```
src/components/
├── CommonButton/      ← Pressable + 디자인 + size/variant
├── CustomTextInput/   ← TextInput + label + error + clear button
├── BottomSheet/       ← @gorhom/bottom-sheet wrap
├── ConfirmDialog/     ← Modal + 확인/취소
├── Header/            ← 페이지 상단 (뒤로가기, 제목)
├── ProgressHeader/    ← 스텝 진행 표시
├── MarkdownView/      ← react-native-markdown-display 등
├── SocialButton/      ← 카카오/구글/애플 버튼
├── SimplifiedSvgIcon/ ← SVG 아이콘 wrap
├── RecordingButton/   ← 음성 녹음 버튼 (lottie)
└── VoiceInputBar/     ← 음성 입력 UI
```

→ **재사용 컴포넌트** 가 디자인 시스템 역할.

## 6. 디자인 토큰 + StyleSheet

```ts
// constants/theme.ts
export const color = {
  primary: '#0064FF',
  secondary: '#FF6B00',
  text: { main: '#1A1A1A', sub: '#666' },
  bg: { main: '#FFFFFF', card: '#F5F5F5' },
  border: '#E5E5E5',
};

export const space = { xs: 4, sm: 8, md: 16, lg: 24, xl: 32 };
export const font = { sm: 12, md: 14, lg: 16, xl: 20, xxl: 24 };
export const radius = { sm: 4, md: 8, lg: 16, full: 9999 };
```

→ Tailwind 없이도 일관성 유지.

## 7. font 등록

```
src/assets/fonts/Pretendard-Regular.ttf
src/assets/fonts/Pretendard-Bold.ttf
```

```js
// react-native.config.js
module.exports = {
  assets: ['./src/assets/fonts'],
};
```

```bash
npx react-native-asset
# 또는 RN 0.69+
yarn ios && yarn android  # 자동 link
```

```tsx
<Text style={{ fontFamily: 'Pretendard-Bold' }}>안녕</Text>
```

## 8. dark mode

```bash
yarn add @react-native-community/hooks
# 또는 RN 내장
```

```tsx
import { useColorScheme } from 'react-native';

const scheme = useColorScheme();   // 'light' | 'dark' | null

<View style={{ backgroundColor: scheme === 'dark' ? '#000' : '#fff' }}>
```

→ 시스템 dark mode 감지. 또는 자체 toggle.

## 9. animation — 기본 / Reanimated

```tsx
// 기본 (Animated)
import { Animated } from 'react-native';

const opacity = useRef(new Animated.Value(0)).current;

useEffect(() => {
  Animated.timing(opacity, { toValue: 1, duration: 500, useNativeDriver: true }).start();
}, []);

<Animated.View style={{ opacity }}>
  ...
</Animated.View>
```

→ 더 강력 = `react-native-reanimated` (60fps, native thread). 자세히 [[../performance/reanimated-gesture]].

## 10. 함정

1. **이미지의 width/height 없음** — uri 이미지 안 보임. style 필수.
2. **font 안 들어감** — react-native-asset 안 돌림. iOS Info.plist 의 UIAppFonts 자동.
3. **vector-icons 의 font 누락** — link 안 됨. iOS 의 Info.plist 의 `UIAppFonts` 에 `Ionicons.ttf` 등 추가.
4. **SVG 의 metro 설정** — `react-native-svg-transformer` 필요.
5. **gradient 의 native module** — pod install 누락 시 안 보임.
6. **lottie 의 native module** — 동일.
7. **모달 안 모달** — 동시 안 됨. 닫고 새로 열기.

## 11. 다음 단계

- [[vector-icons]]
- [[svg-and-gradient]]
- [[lottie]]

## 12. 외부 자료

- [reactnative.directory](https://reactnative.directory/) — RN lib 검색
- [@gorhom/bottom-sheet](https://gorhom.dev/react-native-bottom-sheet/)
- [react-native-paper](https://callstack.github.io/react-native-paper/)

## 13. 관련

- [[../react-native]]
- [[../core-concepts/stylesheet-and-flexbox]]
- [[../performance/reanimated-gesture]]
