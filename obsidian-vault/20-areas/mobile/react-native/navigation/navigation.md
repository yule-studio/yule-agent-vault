---
title: "Navigation in RN — react-navigation v7 hub"
kind: knowledge
project: mobile
agent: engineering-agent/tech-lead
status: current
tags: [mobile, react-native, navigation, react-navigation, hub, beginner]
---

# Navigation in RN

**[[../react-native|↑ RN Hub]]**

> RN 의 페이지 이동 = **@react-navigation v7**. job-answer-app-rn 가 사용.

## 1. 큰 그림

```
RootNavigator (Stack)
├── Start
├── Signin / Signup
├── MainTab (Tab Navigator)
│   ├── Home
│   ├── Search
│   ├── Profile
│   └── ...
├── Setting
└── Detail
```

→ **Stack** (push/pop) + **Tab** (하단 탭) 중첩.

## 2. 라이브러리 — 어떤 navigator?

| | 용도 |
| --- | --- |
| **Native Stack** (`@react-navigation/native-stack`) | 페이지 → 페이지 push. 네이티브 애니메이션. **표준**. |
| **Stack** (`@react-navigation/stack`) | JS 기반 stack. 커스텀 애니메이션 자유. |
| **Bottom Tabs** | 하단 탭 바. |
| **Material Top Tabs** | 상단 탭 (swipe). |
| **Drawer** | 사이드 메뉴 (햄버거). |

→ **Native Stack + Bottom Tabs** 가 80% 케이스.

## 3. 설치

```bash
yarn add @react-navigation/native
yarn add @react-navigation/native-stack
yarn add @react-navigation/bottom-tabs

# 의존성 (필수)
yarn add react-native-screens react-native-safe-area-context
cd ios && pod install
```

## 4. 최소 구성

```tsx
// App.tsx
import { NavigationContainer } from '@react-navigation/native';
import { createNativeStackNavigator } from '@react-navigation/native-stack';

const Stack = createNativeStackNavigator();

function App() {
  return (
    <NavigationContainer>
      <Stack.Navigator>
        <Stack.Screen name="Home" component={HomeScreen} />
        <Stack.Screen name="Detail" component={DetailScreen} />
      </Stack.Navigator>
    </NavigationContainer>
  );
}
```

```tsx
// 페이지 안
function HomeScreen({ navigation }) {
  return (
    <Pressable onPress={() => navigation.navigate('Detail')}>
      <Text>상세로</Text>
    </Pressable>
  );
}
```

→ `Stack.Navigator` 안 `Stack.Screen` 들. `navigation.navigate(name)` 으로 이동.

## 5. 핵심 hook 들

```tsx
import { useNavigation, useRoute } from '@react-navigation/native';

const navigation = useNavigation();
const route = useRoute();

navigation.navigate('Detail', { id: 5 });      // 이동
navigation.goBack();                            // 뒤로
navigation.push('Detail', { ... });             // stack 에 push
navigation.replace('Login');                    // 현재 페이지 교체
navigation.popToTop();                          // 첫 페이지로
navigation.reset({                              // stack 전체 교체
  index: 0,
  routes: [{ name: 'Login' }],
});

route.params.id                                 // params 읽기
route.name                                      // 'Detail'
```

## 6. TypeScript — RootStackParamList

```ts
// meta/link.ts (job-answer-app-rn 패턴)
export type RootStackParamList = {
  Start: undefined;
  Signin: undefined;
  Signup: undefined;
  MainTab: undefined;
  Detail: { id: number };
  Settings: { tab?: 'account' | 'language' };
};
```

```tsx
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import { RootStackParamList } from './meta/link';

const Stack = createNativeStackNavigator<RootStackParamList>();
```

```tsx
// 페이지의 props 타입
import { NativeStackScreenProps } from '@react-navigation/native-stack';

type DetailProps = NativeStackScreenProps<RootStackParamList, 'Detail'>;

function DetailScreen({ route, navigation }: DetailProps) {
  const { id } = route.params;     // type-safe
  return ...;
}
```

```tsx
// useNavigation 의 타입
import { useNavigation } from '@react-navigation/native';
import { NativeStackNavigationProp } from '@react-navigation/native-stack';

const navigation = useNavigation<NativeStackNavigationProp<RootStackParamList>>();
navigation.navigate('Detail', { id: 5 });
```

## 7. 학습 우선순위

1. **[[stack-navigator]]** — Stack / Native Stack 사용법.
2. **[[tab-navigator]]** — 하단 탭.
3. **[[deep-linking]]** — URL 스킴 / Universal Link 로 외부에서 진입.
4. **헤더 / 옵션** — title, headerRight, gestureEnabled.
5. **인증 흐름** — 조건부 navigator (로그인 / 비로그인).

## 8. 인증 흐름 패턴

```tsx
function App() {
  const { user, isLoading } = useAuth();

  if (isLoading) return <Splash />;

  return (
    <NavigationContainer>
      {user ? <MainStack /> : <AuthStack />}
    </NavigationContainer>
  );
}
```

→ 로그인 / 비로그인 stack 자체를 swap. 로그아웃 시 자동 AuthStack 으로.

또는 한 stack 안 조건부 screen:
```tsx
<Stack.Navigator>
  {user ? (
    <>
      <Stack.Screen name="Home" component={HomeScreen} />
      <Stack.Screen name="Detail" component={DetailScreen} />
    </>
  ) : (
    <>
      <Stack.Screen name="Signin" component={SigninScreen} />
      <Stack.Screen name="Signup" component={SignupScreen} />
    </>
  )}
</Stack.Navigator>
```

## 9. 헤더 옵션

```tsx
<Stack.Screen
  name="Detail"
  component={DetailScreen}
  options={{
    title: '상세',
    headerShown: true,
    headerBackTitle: '뒤로',       // iOS 만
    headerStyle: { backgroundColor: '#0064FF' },
    headerTintColor: 'white',
    headerTitleStyle: { fontWeight: 'bold' },
    headerRight: () => <Pressable><Text>저장</Text></Pressable>,
    gestureEnabled: true,
    animation: 'slide_from_right',
  }}
/>
```

→ 화면 내에서 동적으로:
```tsx
useEffect(() => {
  navigation.setOptions({ title: post.title });
}, [post]);
```

## 10. 화면 진입 / 이탈 이벤트

```tsx
useFocusEffect(
  useCallback(() => {
    console.log('진입');
    return () => console.log('이탈');
  }, [])
);

// 또는
useEffect(() => {
  const unsub = navigation.addListener('focus', () => {
    console.log('focus');
  });
  return unsub;
}, [navigation]);
```

→ 페이지 진입 시 데이터 refetch 등.

## 11. job-answer-app-rn 의 구조

```tsx
// router.tsx
const Stack = createNativeStackNavigator<RootStackParamList>();

export const RootNavigator = () => (
  <Stack.Navigator
    initialRouteName="Start"
    screenOptions={{ headerShown: false, gestureEnabled: false }}
  >
    <Stack.Screen name="Start" component={StartPage} />
    <Stack.Screen name="MainTab" component={MainTabNavigator} />
    <Stack.Screen name="Setting" component={SettingScreen} />
    <Stack.Screen name="Signup" component={SignupPage} />
    <Stack.Screen name="Signin" component={SigninPage} />
    ...
  </Stack.Navigator>
);

// navigate/MainTabNavigator.tsx
const Tab = createBottomTabNavigator();

export const MainTabNavigator = () => (
  <Tab.Navigator screenOptions={{ headerShown: false }}>
    <Tab.Screen name="Home" component={HomeTab} />
    <Tab.Screen name="Search" component={SearchTab} />
    <Tab.Screen name="Profile" component={ProfileTab} />
  </Tab.Navigator>
);
```

## 12. 다음 단계

- [[stack-navigator]] — Native Stack 깊이
- [[tab-navigator]] — Bottom Tabs
- [[deep-linking]] — 외부에서 앱 진입

## 13. 외부 자료

- [react-navigation 공식](https://reactnavigation.org/)
- [v7 변경 사항](https://reactnavigation.org/docs/upgrading-from-6.x/)
- [TypeScript 가이드](https://reactnavigation.org/docs/typescript)

## 14. 관련

- [[../react-native]]
- [[../auth/auth]] — 인증 흐름
- [[../../../frontend/react/routing/react-router-v6|web 의 react-router-v6]]
