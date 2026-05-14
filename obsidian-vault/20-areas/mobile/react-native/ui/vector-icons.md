---
title: "Vector Icons — react-native-vector-icons"
kind: knowledge
project: mobile
agent: engineering-agent/tech-lead
status: current
tags: [mobile, react-native, ui, icons, vector-icons]
---

# Vector Icons

**[[ui|↑ UI Hub]]**

> 표준 아이콘 라이브러리. **Ionicons / MaterialIcons / FontAwesome** 등 17 패밀리.

## 1. 설치

```bash
yarn add react-native-vector-icons
yarn add @types/react-native-vector-icons -D
```

### iOS — font link
RN 0.69+ 는 자동. 그 미만이거나 안 되면:
```bash
cd ios && pod install
```

`ios/<AppName>/Info.plist`:
```xml
<key>UIAppFonts</key>
<array>
  <string>Ionicons.ttf</string>
  <string>MaterialIcons.ttf</string>
  <string>FontAwesome.ttf</string>
  <!-- 사용할 패밀리만 -->
</array>
```

### Android — font link
RN 0.69+ 자동. 그 미만:
```gradle
// android/app/build.gradle
apply from: "../../node_modules/react-native-vector-icons/fonts.gradle"

// 또는 특정 패밀리만
project.ext.vectoricons = [
    iconFontNames: ['Ionicons.ttf', 'MaterialIcons.ttf']
]
```

## 2. 사용

```tsx
import Icon from 'react-native-vector-icons/Ionicons';

<Icon name="home" size={24} color="#0064FF" />
<Icon name="heart-outline" size={20} color="gray" />
<Icon name="search" size={28} />
```

### 다른 family
```tsx
import MaterialIcon from 'react-native-vector-icons/MaterialIcons';
import FontAwesome from 'react-native-vector-icons/FontAwesome';
import Feather from 'react-native-vector-icons/Feather';
import AntDesign from 'react-native-vector-icons/AntDesign';
import Entypo from 'react-native-vector-icons/Entypo';
import Octicons from 'react-native-vector-icons/Octicons';
import MaterialCommunityIcons from 'react-native-vector-icons/MaterialCommunityIcons';
```

## 3. 아이콘 검색

- [Ionicons](https://ionic.io/ionicons)
- [Material Icons](https://fonts.google.com/icons)
- [FontAwesome](https://fontawesome.com/icons)
- [Feather](https://feathericons.com/)

→ Vector Icons 의 directory: https://oblador.github.io/react-native-vector-icons/

## 4. Pressable + Icon = IconButton

```tsx
<Pressable
  onPress={onPress}
  hitSlop={10}                      // 탭 영역 확장
  style={({ pressed }) => [{ padding: 8, opacity: pressed ? 0.6 : 1 }]}
>
  <Icon name="close" size={24} color="black" />
</Pressable>
```

→ 작은 아이콘은 `hitSlop` 으로 터치 영역 확장.

## 5. Tab navigator 의 icon

```tsx
import { createBottomTabNavigator } from '@react-navigation/bottom-tabs';
import Icon from 'react-native-vector-icons/Ionicons';

<Tab.Navigator
  screenOptions={({ route }) => ({
    tabBarIcon: ({ focused, color, size }) => {
      const map: Record<string, [string, string]> = {
        Home: ['home', 'home-outline'],
        Search: ['search', 'search-outline'],
        Profile: ['person', 'person-outline'],
      };
      const [filled, outline] = map[route.name];
      return <Icon name={focused ? filled : outline} size={size} color={color} />;
    },
  })}
/>
```

## 6. Button + Icon — 함께

```tsx
<Pressable
  onPress={...}
  style={{
    flexDirection: 'row',
    alignItems: 'center',
    backgroundColor: '#0064FF',
    paddingHorizontal: 16,
    paddingVertical: 12,
    borderRadius: 8,
    gap: 8,
  }}
>
  <Icon name="add" size={20} color="white" />
  <Text style={{ color: 'white', fontWeight: '600' }}>추가</Text>
</Pressable>
```

## 7. Icon 의 a11y

```tsx
<Icon
  name="close"
  size={24}
  color="black"
  accessibilityLabel="닫기"
  accessible
/>
```

→ 스크린 리더 사용자.

## 8. 커스텀 아이콘 — SVG

vector-icons 에 없는 아이콘 = SVG.

```bash
yarn add react-native-svg
yarn add -D react-native-svg-transformer
```

```tsx
import MyIcon from '@assets/icons/my-icon.svg';
<MyIcon width={24} height={24} fill="black" />
```

자세히 [[svg-and-gradient]].

## 9. icon 생성 + bundle 최적화

```bash
# 사용 안 하는 family 제외 → bundle 절약
# android/app/build.gradle
project.ext.vectoricons = [
    iconFontNames: ['Ionicons.ttf', 'MaterialIcons.ttf']    // 이 두 개만
]
```

→ 전체 (~3-4MB) → 일부만.

## 10. 폰트 size + line height

```tsx
<View style={{ flexDirection: 'row', alignItems: 'center', gap: 4 }}>
  <Icon name="time-outline" size={14} color="gray" />
  <Text style={{ fontSize: 12, color: 'gray', lineHeight: 16 }}>3분 전</Text>
</View>
```

→ icon size 와 text fontSize 맞추기. line-height 로 vertical align.

## 11. job-answer-app-rn 의 패턴

```tsx
// components/SimplifiedSvgIcon (커스텀 SVG wrap)
import { SvgXml } from 'react-native-svg';

export const SimplifiedSvgIcon = ({ name, size, color }) => {
  const svgString = iconMap[name];   // SVG 문자열 매핑
  return <SvgXml xml={svgString} width={size} height={size} stroke={color} />;
};

// 사용
<SimplifiedSvgIcon name="mic" size={24} color="#0064FF" />
```

→ vector-icons 와 자체 SVG 의 혼용. 디자인 일관성.

## 12. 함정

1. **font link 안 됨** — 아이콘 자리에 ❑ (네모) 표시. iOS Info.plist / Android gradle 확인.
2. **size prop 누락** — 기본 12 또는 패밀리 default. 명시 권장.
3. **vector-icons 의 v10+** — RN 0.72+ 자동 link. 명시 link 불필요.
4. **TypeScript 의 name 타입** — 자동완성 안 됨. 모든 icon 이름이 string. `keyof typeof iconMap` 같은 narrowing 권장.
5. **너무 많은 family import** — bundle 폭증. 사용 family 만.
6. **iOS sim 의 font 캐시** — 변경 후 안 보이면 `xcrun simctl shutdown all` 후 재실행.

## 13. 다음 단계

- [[svg-and-gradient]]
- [[lottie]]

## 14. 외부 자료

- [react-native-vector-icons](https://github.com/oblador/react-native-vector-icons)
- [Vector Icons Directory](https://oblador.github.io/react-native-vector-icons/)
- [Ionicons](https://ionic.io/ionicons)

## 15. 관련

- [[ui]]
- [[../navigation/tab-navigator]]
- [[svg-and-gradient]]
