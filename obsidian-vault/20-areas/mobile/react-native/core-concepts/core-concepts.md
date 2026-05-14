---
title: "Core Concepts — RN 의 컴포넌트와 스타일"
kind: knowledge
project: mobile
agent: engineering-agent/tech-lead
status: current
tags: [mobile, react-native, core-concepts, hub, beginner]
---

# Core Concepts

**[[../react-native|↑ RN Hub]]**

> RN 의 기본 컴포넌트, 스타일, 플랫폼 차이, SafeArea.

## 1. 학습 순서

1. **[[rn-components]]** — View / Text / Image / ScrollView / FlatList / TextInput / Pressable
2. **[[stylesheet-and-flexbox]]** — StyleSheet.create + Flexbox (web 과 다른 default)
3. **[[platform-differences]]** — Platform.OS / `.ios.tsx` vs `.android.tsx`
4. **[[safe-area-and-status-bar]]** — Notch / Dynamic Island / 상태바
5. **(다음)** [[../navigation/navigation|Navigation]]

## 2. React 개념은 그대로

`[[../../../frontend/react/core-concepts/core-concepts|frontend Core Concepts]]` 에서 다룬 다음 개념은 **RN 도 동일**:
- 컴포넌트 = 함수.
- JSX.
- Props / State.
- Hooks (useState, useEffect, useRef, useMemo, useCallback, useReducer, useContext, custom).
- 조건부 / List 렌더링.
- Event handler (단 `onClick` 대신 `onPress`).
- Forms (controlled input — `TextInput` 의 `value` + `onChangeText`).

→ web React 익숙하면 RN 의 90% 가 친숙.

## 3. RN 만의 차이점 — 6 가지

| | web React | RN |
| --- | --- | --- |
| 1. HTML 태그 X | `<div>`, `<p>`, `<img>` | `<View>`, `<Text>`, `<Image>` |
| 2. CSS X | className / styled-components | `StyleSheet.create` |
| 3. 텍스트는 `<Text>` 안에만 | string 어디든 | `<View>` 안 직접 string ❌ |
| 4. Flexbox default | `flex-direction: row` 기본 | `flex-direction: column` 기본 |
| 5. 단위 | px / rem / % | 숫자 (= density-independent pixel) |
| 6. 이벤트 | onClick / onChange | onPress / onChangeText |

## 4. 가장 간단한 RN 컴포넌트

```tsx
import React from 'react';
import { View, Text, StyleSheet, Pressable } from 'react-native';

export default function Greeting() {
  return (
    <View style={styles.container}>
      <Text style={styles.title}>안녕!</Text>
      <Pressable onPress={() => alert('눌렸음')} style={styles.button}>
        <Text style={styles.buttonText}>버튼</Text>
      </Pressable>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, alignItems: 'center', justifyContent: 'center' },
  title: { fontSize: 24, fontWeight: 'bold' },
  button: { marginTop: 16, backgroundColor: '#0064FF', padding: 12, borderRadius: 8 },
  buttonText: { color: 'white', fontSize: 16 },
});
```

→ View / Text / Pressable + StyleSheet. RN 의 거의 모든 컴포넌트가 이 패턴.

## 5. 가장 자주 헷갈리는 함정

### (1) `<View>` 안 string 직접

```tsx
// ❌ "Text strings must be rendered within a <Text> component"
<View>안녕</View>

// ✅
<View><Text>안녕</Text></View>
```

### (2) `onClick` 사용

```tsx
// ❌ web 의 onClick 은 RN 에서 동작 X
<Pressable onClick={...}>...</Pressable>

// ✅
<Pressable onPress={...}>...</Pressable>
```

### (3) `<input>` 사용

```tsx
// ❌ HTML input
<input value={...} onChange={...} />

// ✅
<TextInput value={...} onChangeText={setText} />
```

### (4) CSS 단위

```tsx
// ❌
<View style={{ padding: '16px' }} />     // string 단위

// ✅
<View style={{ padding: 16 }} />          // number (dp)
```

### (5) 디바이스 크기 가정

```tsx
// ❌ 고정 크기
<View style={{ width: 375 }} />

// ✅ Dimensions / 비율
import { Dimensions } from 'react-native';
const { width } = Dimensions.get('window');
<View style={{ width: width * 0.9 }} />
```

자세히 [[stylesheet-and-flexbox]].

## 6. 모든 RN core 컴포넌트 (자주 쓰는)

| 카테고리 | 컴포넌트 |
| --- | --- |
| **레이아웃** | View, ScrollView, SafeAreaView, KeyboardAvoidingView |
| **텍스트** | Text |
| **이미지** | Image, ImageBackground |
| **입력** | TextInput, Switch, Pressable, TouchableOpacity, TouchableHighlight |
| **리스트** | FlatList, SectionList, VirtualizedList |
| **상태** | ActivityIndicator (로딩), RefreshControl |
| **모달** | Modal |
| **알림** | Alert, ToastAndroid (Android 전용) |
| **상태바** | StatusBar |
| **플랫폼** | Platform, Dimensions, PixelRatio |
| **링크** | Linking (외부 URL) |

자세히 [[rn-components]].

## 7. native 컴포넌트로 변환

```tsx
<View>...</View>
       ↓
iOS:     UIView
Android: ViewGroup
```

→ RN 은 우리가 작성한 JSX 를 native 컴포넌트로 매핑. 결과: 진짜 native UI.

## 8. 학습 단계 — 처음 1 주

1. **Day 1** — View / Text / Image 로 정적 화면 1개.
2. **Day 2** — useState + Pressable 로 동적 화면.
3. **Day 3** — TextInput + 로그인 form.
4. **Day 4** — ScrollView / FlatList 로 list 화면.
5. **Day 5** — StyleSheet + Flexbox 로 디자인 흉내.
6. **Day 6** — Platform.OS / .ios.tsx 로 분기.
7. **Day 7** — SafeArea / StatusBar 로 마무리.

## 9. 다음 단계

- [[rn-components]] — 컴포넌트 정밀
- [[../navigation/navigation]] — 페이지 이동

## 10. 관련

- [[../../../frontend/react/core-concepts/core-concepts|web React core]]
- [[../react-native]]
