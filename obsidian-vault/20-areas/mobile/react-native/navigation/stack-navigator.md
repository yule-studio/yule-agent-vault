---
title: "Native Stack Navigator — 페이지 push/pop"
kind: knowledge
project: mobile
agent: engineering-agent/tech-lead
status: current
tags: [mobile, react-native, react-navigation, native-stack]
---

# Native Stack Navigator

**[[navigation|↑ Navigation Hub]]**

> `@react-navigation/native-stack`. 네이티브 애니메이션 + 60fps.

## 1. Native Stack vs Stack

| | Stack (JS) | Native Stack |
| --- | --- | --- |
| 애니메이션 | JS thread | native thread (UIKit / Fragment) |
| 성능 | 보통 | 빠름 |
| 커스텀 | 자유 | 제한 |
| 헤더 | JS | native |

→ **Native Stack 이 표준**. 디자인이 매우 특수하면 Stack.

## 2. 설치

```bash
yarn add @react-navigation/native-stack
yarn add react-native-screens react-native-safe-area-context
cd ios && pod install
```

## 3. 기본 사용

```tsx
import { createNativeStackNavigator } from '@react-navigation/native-stack';

const Stack = createNativeStackNavigator();

<NavigationContainer>
  <Stack.Navigator>
    <Stack.Screen name="Home" component={HomeScreen} />
    <Stack.Screen name="Detail" component={DetailScreen} />
  </Stack.Navigator>
</NavigationContainer>
```

## 4. params 전달 + 받기

### 전달
```tsx
navigation.navigate('Detail', { id: 5, source: 'list' });
```

### 받기
```tsx
function DetailScreen({ route }) {
  const { id, source } = route.params;
  return <Text>ID: {id}</Text>;
}
```

### TypeScript
```ts
type RootStackParamList = {
  Home: undefined;
  Detail: { id: number; source?: string };
};

type DetailProps = NativeStackScreenProps<RootStackParamList, 'Detail'>;

function DetailScreen({ route, navigation }: DetailProps) {
  const { id, source } = route.params;
  navigation.setParams({ source: 'edited' });    // 부분 업데이트
}
```

## 5. screenOptions vs options

```tsx
// 모든 screen 의 기본
<Stack.Navigator
  screenOptions={{
    headerStyle: { backgroundColor: '#0064FF' },
    headerTintColor: 'white',
    headerTitleStyle: { fontWeight: 'bold' },
    contentStyle: { backgroundColor: 'white' },
  }}
>
  {/* 개별 screen 의 override */}
  <Stack.Screen
    name="Home"
    component={Home}
    options={{ title: '홈', headerShown: false }}
  />
  <Stack.Screen
    name="Detail"
    component={Detail}
    options={({ route }) => ({ title: `Post ${route.params.id}` })}
  />
</Stack.Navigator>
```

## 6. headerRight / headerLeft — 커스텀 버튼

```tsx
<Stack.Screen
  name="Edit"
  component={EditScreen}
  options={({ navigation }) => ({
    title: '편집',
    headerRight: () => (
      <Pressable onPress={() => save()}>
        <Text style={{ color: 'blue' }}>저장</Text>
      </Pressable>
    ),
    headerLeft: () => (
      <Pressable onPress={() => navigation.goBack()}>
        <Text>취소</Text>
      </Pressable>
    ),
  })}
/>
```

### 화면 내에서 동적
```tsx
function EditScreen({ navigation }) {
  const [isDirty, setIsDirty] = useState(false);

  useEffect(() => {
    navigation.setOptions({
      headerRight: () => (
        <Pressable onPress={save} disabled={!isDirty}>
          <Text style={{ color: isDirty ? 'blue' : 'gray' }}>저장</Text>
        </Pressable>
      ),
    });
  }, [navigation, isDirty]);
}
```

## 7. animation 옵션

```tsx
options={{
  animation: 'slide_from_right',    // iOS default
  // 'slide_from_left' | 'slide_from_bottom' (modal) | 'fade' | 'none'
}}
```

### 모달 스타일
```tsx
options={{
  presentation: 'modal',            // 아래에서 슬라이드 업
  // 또는 'transparentModal'
}}
```

## 8. 헤더 숨기기 / 보이기

```tsx
options={{
  headerShown: false,           // 헤더 안 그림
  statusBarStyle: 'dark',
  statusBarColor: 'white',
}}
```

→ 헤더 없으면 직접 SafeAreaView + 커스텀 header 컴포넌트.

## 9. gestureEnabled — swipe back

```tsx
options={{
  gestureEnabled: false,        // 사용자 스와이프로 뒤로가기 막기 (form 진행 중)
  fullScreenGestureEnabled: true,  // 전체 화면 swipe
}}
```

## 10. headerBackButton — 커스텀 back

```tsx
options={{
  headerBackTitle: '뒤로',         // iOS
  headerBackButtonDisplayMode: 'minimal',  // 아이콘만
  headerBackTitleStyle: { fontSize: 14 },
}}
```

→ 또는 `headerLeft` 로 완전 커스텀.

## 11. push / navigate / replace / pop 의 차이

```tsx
navigation.navigate('Detail', { id: 5 });
// stack 의 같은 이름 페이지가 있으면 새 페이지 안 만듦. params 업데이트.

navigation.push('Detail', { id: 5 });
// 항상 새 페이지 push. 같은 이름 두 개 stack.

navigation.replace('Login');
// 현재 페이지를 Login 으로 교체. 뒤로가기 X.

navigation.goBack();
navigation.pop();
navigation.pop(2);              // 2번 뒤로
navigation.popToTop();          // 첫 페이지로

navigation.reset({              // stack 전체 교체
  index: 0,
  routes: [{ name: 'Home' }],
});
```

### 흔한 패턴 — 로그인 후
```tsx
const onLoginSuccess = () => {
  navigation.reset({
    index: 0,
    routes: [{ name: 'MainTab' }],
  });
};
```

→ 로그인 stack 의 history 다 사라짐.

## 12. nested navigator — Tab 안 Stack 안 Tab...

```tsx
<RootStack.Navigator>
  <RootStack.Screen name="Auth" component={AuthStack} />
  <RootStack.Screen name="Main" component={MainTab} />
</RootStack.Navigator>

// 깊은 곳에서 root 의 다른 stack 으로
navigation.navigate('Auth', { screen: 'Login' });
navigation.navigate('Main', { screen: 'Home', params: { tab: 'feed' } });
```

→ nested 면 `navigate(parent, { screen, params })`.

## 13. transition 콜백

```tsx
useEffect(() => {
  const unsub = navigation.addListener('transitionStart', (e) => {
    console.log('애니메이션 시작', e.data.closing);
  });
  return unsub;
}, [navigation]);
```

이벤트:
- `focus` / `blur` — 화면 in / out.
- `beforeRemove` — pop 직전. 예: "저장 안 했는데 뒤로?".
- `transitionStart` / `transitionEnd` — 애니메이션.
- `state` — navigator state 변경.

### beforeRemove — 확인 dialog

```tsx
useEffect(() => {
  const unsub = navigation.addListener('beforeRemove', (e) => {
    if (!isDirty) return;
    e.preventDefault();
    Alert.alert('변경 사항이 있습니다', '나가시겠습니까?', [
      { text: '취소', style: 'cancel' },
      { text: '나가기', style: 'destructive', onPress: () => navigation.dispatch(e.data.action) },
    ]);
  });
  return unsub;
}, [navigation, isDirty]);
```

## 14. job-answer-app-rn 의 패턴

```tsx
<Stack.Navigator
  initialRouteName="Start"
  screenOptions={{ headerShown: false, gestureEnabled: false }}
>
  <Stack.Screen name="Start" component={StartPage} />
  <Stack.Screen name="MainTab" component={MainTabNavigator} />
  <Stack.Screen name="Signin" component={SigninPage} />
  <Stack.Screen name="Signup" component={SignupPage} />
  <Stack.Screen name="SignupFinish" component={SignupFinishPage} />
</Stack.Navigator>
```

→ `headerShown: false` 통일 + 페이지마다 자체 ProgressHeader / Header 컴포넌트. `gestureEnabled: false` 로 swipe back 차단 (form flow 보호).

## 15. 함정

1. **`react-native-screens` 미설치** — Native Stack 동작 X.
2. **`SafeAreaProvider` 누락** — `react-native-screens` 가 SafeArea 필요.
3. **navigate vs push 헷갈림** — 같은 이름 페이지 재방문 시 동작 다름.
4. **headerShown: false 후 SafeArea 누락** — 노치 침범.
5. **params 변경 시 navigation.setParams** — 부분 업데이트.
6. **TypeScript 의 RootStackParamList 의 typo** — 한 글자 차이로 자동완성 안 됨.
7. **deep nested navigator 의 navigate** — `{ screen, params }` 객체 필요.

## 16. 다음 단계

- [[tab-navigator]] — 하단 탭
- [[deep-linking]] — URL 스킴

## 17. 외부 자료

- [Native Stack 공식](https://reactnavigation.org/docs/native-stack-navigator)
- [Common mistakes](https://reactnavigation.org/docs/troubleshooting)

## 18. 관련

- [[navigation]]
- [[tab-navigator]]
- [[../auth/auth]] — 로그인 후 reset
