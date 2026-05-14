---
title: "Bottom Tab Navigator — 하단 탭 바"
kind: knowledge
project: mobile
agent: engineering-agent/tech-lead
status: current
tags: [mobile, react-native, react-navigation, bottom-tabs]
---

# Bottom Tab Navigator

**[[navigation|↑ Navigation Hub]]**

> `@react-navigation/bottom-tabs`. 앱 하단의 탭 바.

## 1. 설치

```bash
yarn add @react-navigation/bottom-tabs
```

## 2. 기본

```tsx
import { createBottomTabNavigator } from '@react-navigation/bottom-tabs';

const Tab = createBottomTabNavigator();

function MainTab() {
  return (
    <Tab.Navigator>
      <Tab.Screen name="Home" component={HomeScreen} />
      <Tab.Screen name="Search" component={SearchScreen} />
      <Tab.Screen name="Profile" component={ProfileScreen} />
    </Tab.Navigator>
  );
}
```

## 3. icon — vector-icons + tabBarIcon

```bash
yarn add react-native-vector-icons
```

```tsx
import Icon from 'react-native-vector-icons/Ionicons';

<Tab.Navigator
  screenOptions={({ route }) => ({
    tabBarIcon: ({ focused, color, size }) => {
      const map: Record<string, string> = {
        Home: focused ? 'home' : 'home-outline',
        Search: focused ? 'search' : 'search-outline',
        Profile: focused ? 'person' : 'person-outline',
      };
      return <Icon name={map[route.name]} size={size} color={color} />;
    },
    tabBarActiveTintColor: '#0064FF',
    tabBarInactiveTintColor: 'gray',
  })}
>
  ...
</Tab.Navigator>
```

자세히 [[../ui/vector-icons]].

## 4. tabBarStyle — 디자인

```tsx
screenOptions={{
  tabBarStyle: {
    backgroundColor: 'white',
    borderTopColor: '#eee',
    borderTopWidth: 1,
    height: 60,
    paddingBottom: 8,
  },
  tabBarLabelStyle: { fontSize: 12 },
  tabBarShowLabel: true,
}}
```

## 5. badge — 알림 개수

```tsx
<Tab.Screen
  name="Message"
  component={MessageScreen}
  options={{ tabBarBadge: unreadCount }}
/>
```

→ 동적이면:
```tsx
useEffect(() => {
  navigation.setOptions({ tabBarBadge: count > 0 ? count : undefined });
}, [count]);
```

## 6. tab 숨기기 — 특정 페이지에서

```tsx
// 어떤 화면에서 tab 숨기기
useEffect(() => {
  navigation.getParent()?.setOptions({ tabBarStyle: { display: 'none' } });
  return () => navigation.getParent()?.setOptions({ tabBarStyle: defaultStyle });
}, [navigation]);
```

→ 채팅 상세 같은 immersive 화면.

## 7. Stack 안 Tab — 흔한 구조

```tsx
// RootStack 안 MainTab 안 각 탭의 Stack
const RootStack = createNativeStackNavigator();
const MainTab = createBottomTabNavigator();
const HomeStack = createNativeStackNavigator();

function HomeStackNavigator() {
  return (
    <HomeStack.Navigator>
      <HomeStack.Screen name="HomeMain" component={Home} />
      <HomeStack.Screen name="HomeDetail" component={Detail} />
    </HomeStack.Navigator>
  );
}

function MainTabNavigator() {
  return (
    <MainTab.Navigator>
      <MainTab.Screen name="HomeTab" component={HomeStackNavigator} />
      <MainTab.Screen name="ProfileTab" component={ProfileStackNavigator} />
    </MainTab.Navigator>
  );
}

function App() {
  return (
    <NavigationContainer>
      <RootStack.Navigator>
        <RootStack.Screen name="Auth" component={AuthStack} />
        <RootStack.Screen name="Main" component={MainTabNavigator} />
        <RootStack.Screen name="Detail" component={Detail} />  {/* 탭 위에 modal 처럼 */}
      </RootStack.Navigator>
    </NavigationContainer>
  );
}
```

## 8. 탭 클릭 동작 — 같은 탭 누름

기본 동작:
- 다른 탭 누름 → 그 탭으로 이동.
- 같은 탭 누름 → 그 탭의 stack 의 첫 화면으로 (popToTop).

커스텀:
```tsx
<Tab.Screen
  name="Home"
  component={Home}
  listeners={({ navigation }) => ({
    tabPress: (e) => {
      // e.preventDefault();          // 이동 자체 막기
      // scrollToTop();                // 같은 탭 누름 시 추가 동작
    },
  })}
/>
```

## 9. 안 보이지만 등록된 screen

```tsx
<Tab.Screen
  name="Hidden"
  component={HiddenScreen}
  options={{ tabBarButton: () => null }}    // 탭 아이콘 안 보임
/>
```

→ tab navigator 안 라우팅만 필요할 때.

## 10. center tab (중앙 큰 button)

```tsx
<Tab.Navigator>
  <Tab.Screen name="Home" component={Home} />
  <Tab.Screen
    name="Create"
    component={CreateScreen}
    options={{
      tabBarButton: (props) => (
        <Pressable
          {...props}
          style={{
            top: -20,
            backgroundColor: '#0064FF',
            width: 60, height: 60, borderRadius: 30,
            justifyContent: 'center', alignItems: 'center',
          }}
        >
          <Icon name="add" size={32} color="white" />
        </Pressable>
      ),
    }}
  />
  <Tab.Screen name="Profile" component={Profile} />
</Tab.Navigator>
```

→ Instagram / Twitter 류 중앙 + 버튼.

## 11. 키보드 시 tab 숨김

```tsx
screenOptions={{
  tabBarHideOnKeyboard: true,    // Android 권장. iOS 는 자체 처리
}}
```

## 12. job-answer-app-rn 의 MainTabNavigator

```tsx
// navigate/MainTabNavigator.tsx (실제 구조 추정)
const Tab = createBottomTabNavigator();

export const MainTabNavigator = () => (
  <Tab.Navigator
    screenOptions={({ route }) => ({
      headerShown: false,
      tabBarIcon: ({ focused, color, size }) => {
        const icons = {
          Home: 'home',
          Progress: 'stats-chart',
          Settings: 'settings',
        };
        return <Icon name={icons[route.name] ?? 'help'} size={size} color={color} />;
      },
      tabBarActiveTintColor: '#0064FF',
      tabBarInactiveTintColor: '#999',
    })}
  >
    <Tab.Screen name="Home" component={HomeTab} />
    <Tab.Screen name="Progress" component={ProgressTab} />
    <Tab.Screen name="Settings" component={SettingsTab} />
  </Tab.Navigator>
);
```

## 13. 함정

1. **icon prop 의 동적 매핑** — string typo 시 빈 아이콘.
2. **tabBarStyle 동적 변경** — `getParent()?.setOptions` 필요.
3. **screen 안 push 시 tab 안 사라짐** — nested 구조 의도면 root stack 에 detail 두기.
4. **headerShown: true 의 중복 헤더** — Tab + Stack 둘 다 헤더 → 두 줄. 한쪽 끄기.
5. **Android 의 키보드 push up** — `tabBarHideOnKeyboard: true`.
6. **iPad / 가로 모드** — 탭 자동 위치 다름. 테스트.

## 14. 다음 단계

- [[deep-linking]] — 외부 URL 로 진입
- [[../ui/vector-icons]] — 탭 아이콘

## 15. 외부 자료

- [Bottom Tabs 공식](https://reactnavigation.org/docs/bottom-tab-navigator)

## 16. 관련

- [[navigation]]
- [[stack-navigator]]
- [[../ui/vector-icons]]
