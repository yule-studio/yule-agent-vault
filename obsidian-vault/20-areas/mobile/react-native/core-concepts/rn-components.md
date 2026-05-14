---
title: "RN Components — View / Text / Image / ScrollView / FlatList / TextInput / Pressable"
kind: knowledge
project: mobile
agent: engineering-agent/tech-lead
status: current
tags: [mobile, react-native, components, view, text, image, flatlist, beginner]
---

# RN Components

**[[core-concepts|↑ Core Concepts]]**

> RN 의 기본 native 컴포넌트 7가지. 거의 모든 화면은 이 조합.

## 1. View — `<div>` 의 RN 대응

```tsx
<View style={{ padding: 16, backgroundColor: 'white' }}>
  ...
</View>
```

- 레이아웃 / 컨테이너.
- Flexbox 자동 (`flex-direction: column` 기본).
- 클릭 이벤트 직접 X — Pressable / TouchableOpacity 로 감싸기.

## 2. Text — 텍스트 표시

```tsx
<Text style={{ fontSize: 16, color: '#333' }}>안녕하세요</Text>
```

- **모든 텍스트는 반드시 Text 안**. `<View>안녕</View>` ❌.
- 중첩 가능:
```tsx
<Text>
  큰 글씨{' '}
  <Text style={{ fontWeight: 'bold', color: 'red' }}>강조</Text>
  {' '}계속
</Text>
```

### selectable / numberOfLines
```tsx
<Text selectable>복사 가능 텍스트</Text>
<Text numberOfLines={2} ellipsizeMode="tail">긴 텍스트...</Text>
```

## 3. Image — `<img>` 대응

```tsx
// 로컬
import logo from '@assets/images/logo.png';
<Image source={logo} style={{ width: 100, height: 100 }} />

// URL
<Image
  source={{ uri: 'https://example.com/photo.jpg' }}
  style={{ width: 200, height: 200 }}
/>

// 권장: resizeMode
<Image source={...} style={...} resizeMode="cover" />
```

### resizeMode
- `cover` — 비율 유지 + 영역 채움 + 잘림 가능 (object-fit cover).
- `contain` — 비율 유지 + 안 잘림 + 여백.
- `stretch` — 비율 무시 + 영역 채움.
- `center` — 원본 크기 중앙.

### Image vs ImageBackground
```tsx
<ImageBackground source={bg} style={{ flex: 1 }}>
  <Text>위에 표시</Text>
</ImageBackground>
```

→ 이미지 위에 children 올릴 때.

### 캐싱 / 빠른 이미지 — `react-native-fast-image`
```bash
yarn add react-native-fast-image
```

```tsx
import FastImage from 'react-native-fast-image';
<FastImage source={{ uri }} style={...} />
```

→ 큰 이미지 list 의 표준.

## 4. ScrollView — 스크롤 가능 영역

```tsx
<ScrollView style={{ flex: 1 }}>
  <Text>긴 컨텐츠 1</Text>
  <Text>긴 컨텐츠 2</Text>
  ...
</ScrollView>
```

- 작은 list / 정적 컨텐츠.
- **큰 list 는 FlatList** (가상화).
- 가로 스크롤: `horizontal`.

### 새로고침 (Pull to refresh)
```tsx
import { RefreshControl } from 'react-native';

const [refreshing, setRefreshing] = useState(false);

<ScrollView
  refreshControl={
    <RefreshControl
      refreshing={refreshing}
      onRefresh={async () => {
        setRefreshing(true);
        await reload();
        setRefreshing(false);
      }}
    />
  }
>
  ...
</ScrollView>
```

### Keyboard 회피
```tsx
import { KeyboardAvoidingView } from 'react-native';

<KeyboardAvoidingView behavior={Platform.OS === 'ios' ? 'padding' : 'height'}>
  <ScrollView>
    <TextInput ... />
  </ScrollView>
</KeyboardAvoidingView>
```

→ 키보드가 input 가리지 않게.

## 5. FlatList — 가상화 list (필수 도구)

```tsx
<FlatList
  data={items}
  keyExtractor={(item) => String(item.id)}
  renderItem={({ item }) => (
    <Pressable onPress={() => onSelect(item.id)}>
      <Text>{item.name}</Text>
    </Pressable>
  )}
/>
```

→ 100+ row 면 ScrollView 대신 **FlatList 필수**. 자동 가상화.

### 자주 쓰는 prop
```tsx
<FlatList
  data={items}
  renderItem={renderItem}
  keyExtractor={...}
  
  // 헤더 / 푸터
  ListHeaderComponent={<Header />}
  ListFooterComponent={isLoading ? <Spinner /> : null}
  ListEmptyComponent={<EmptyState />}
  
  // 구분선
  ItemSeparatorComponent={() => <View style={{ height: 1, backgroundColor: '#eee' }} />}
  
  // 무한 스크롤
  onEndReached={loadMore}
  onEndReachedThreshold={0.5}
  
  // pull to refresh
  refreshing={refreshing}
  onRefresh={onRefresh}
  
  // 성능
  initialNumToRender={10}
  maxToRenderPerBatch={10}
  windowSize={5}
  removeClippedSubviews
/>
```

자세히 [[../performance/flatlist]].

## 6. SectionList — 섹션 구분 list

```tsx
<SectionList
  sections={[
    { title: 'A', data: ['Apple', 'Avocado'] },
    { title: 'B', data: ['Banana', 'Blueberry'] },
  ]}
  keyExtractor={(item, i) => item + i}
  renderItem={({ item }) => <Text>{item}</Text>}
  renderSectionHeader={({ section }) => <Text style={{ fontWeight: 'bold' }}>{section.title}</Text>}
/>
```

→ 알파벳 / 카테고리 구분.

## 7. TextInput — input 대응

```tsx
const [text, setText] = useState('');

<TextInput
  value={text}
  onChangeText={setText}
  placeholder="이메일"
  keyboardType="email-address"
  autoCapitalize="none"
  autoCorrect={false}
  style={{
    borderWidth: 1,
    borderColor: '#ccc',
    borderRadius: 8,
    padding: 12,
  }}
/>
```

### keyboardType
- `default` — 일반.
- `email-address` — 이메일 (`@` 키).
- `numeric` — 숫자 패드.
- `phone-pad` — 전화번호.
- `decimal-pad` — 소수점.
- `url`.

### 비밀번호
```tsx
<TextInput value={pw} onChangeText={setPw} secureTextEntry />
```

### returnKeyType (Enter 키 라벨)
```tsx
<TextInput returnKeyType="next" onSubmitEditing={...} />
// "next" | "done" | "send" | "search" | "go"
```

### Focus 이동
```tsx
const inputRef = useRef<TextInput>(null);
<TextInput onSubmitEditing={() => inputRef.current?.focus()} />
<TextInput ref={inputRef} />
```

## 8. Pressable — 클릭 (최신 권장)

```tsx
<Pressable
  onPress={() => console.log('clicked')}
  onLongPress={() => console.log('long')}
  style={({ pressed }) => [
    styles.button,
    pressed && { opacity: 0.7 },
  ]}
>
  <Text style={styles.buttonText}>Click</Text>
</Pressable>
```

### Pressable vs TouchableOpacity

| | 옛 (TouchableOpacity) | 새 (Pressable) |
| --- | --- | --- |
| 권장 | 보존 사용 | 최신 권장 |
| 압력 상태 | `activeOpacity` prop | `({ pressed })` 콜백 |
| API | 단순 | 더 풍부 (longPress, hoverIn 등) |

→ 신규 코드는 Pressable.

### Button (간단 + 한정적)
```tsx
<Button title="제출" onPress={...} color="#0064FF" />
```

→ 디자인 커스텀 제한. Pressable 권장.

## 9. Switch — toggle

```tsx
const [on, setOn] = useState(false);
<Switch value={on} onValueChange={setOn} />
```

## 10. ActivityIndicator — 로딩 스피너

```tsx
<ActivityIndicator size="large" color="#0064FF" />
```

## 11. Modal — 전체 화면 모달

```tsx
import { Modal } from 'react-native';

const [visible, setVisible] = useState(false);

<Modal
  visible={visible}
  transparent
  animationType="fade"
  onRequestClose={() => setVisible(false)}
>
  <View style={{ flex: 1, backgroundColor: 'rgba(0,0,0,0.5)', justifyContent: 'center' }}>
    <View style={{ backgroundColor: 'white', padding: 24, margin: 24, borderRadius: 8 }}>
      <Text>모달 컨텐츠</Text>
      <Pressable onPress={() => setVisible(false)}>
        <Text>닫기</Text>
      </Pressable>
    </View>
  </View>
</Modal>
```

→ `transparent` + 자체 backdrop 으로 dialog 만들기.
더 복잡한 경우 **BottomSheet** (`@gorhom/bottom-sheet`) 라이브러리.

## 12. Alert — 시스템 alert

```tsx
import { Alert } from 'react-native';

Alert.alert(
  '삭제하시겠습니까?',
  '되돌릴 수 없습니다.',
  [
    { text: '취소', style: 'cancel' },
    { text: '삭제', style: 'destructive', onPress: () => doDelete() },
  ]
);
```

→ 네이티브 alert. 디자인 커스텀 X.

## 13. SafeAreaView — Notch 영역 회피

```tsx
import { SafeAreaView } from 'react-native-safe-area-context';

<SafeAreaView style={{ flex: 1 }}>
  <Text>iPhone 노치 / 다이내믹 아일랜드 침범 안 함</Text>
</SafeAreaView>
```

자세히 [[safe-area-and-status-bar]].

## 14. StatusBar — 상태바 (시계 영역)

```tsx
import { StatusBar } from 'react-native';

<StatusBar barStyle="dark-content" backgroundColor="white" />
// barStyle: 'default' | 'light-content' (흰 텍스트) | 'dark-content'
```

## 15. Linking — 외부 URL / deep link

```tsx
import { Linking } from 'react-native';

// 외부
Linking.openURL('https://google.com');
Linking.openURL('tel:01012345678');
Linking.openURL('mailto:hello@example.com');

// 우리 앱의 deep link 수신
useEffect(() => {
  const handler = (url) => console.log(url);
  Linking.addEventListener('url', handler);
  return () => /* removeEventListener */;
}, []);
```

자세히 [[../navigation/deep-linking]].

## 16. job-answer-app-rn 의 컴포넌트 예

```bash
ls ~/masterway-dev/job-answer-app-rn/src/components/
```

확인된 자체 컴포넌트:
- `CommonButton/` — Pressable wrap + 디자인.
- `CustomTextInput/` — TextInput wrap + label / error.
- `BottomSheet/` — 모달의 상승 / 하강.
- `ConfirmDialog/` — 확인 다이얼로그.
- `Header/`, `ProgressHeader/` — 페이지 상단.
- `MarkdownView/` — 마크다운 표시 (외부 lib + wrap).
- `SocialButton/` — 소셜 로그인 버튼.
- `VoiceInputBar/`, `RecordingButton/` — 음성 입력 UI.

## 17. 함정

1. **`<View>` 안 raw string** — "Text strings must be rendered..."
2. **`onClick`** — `onPress` 가 정답.
3. **`<input>` / `<button>`** — 각각 TextInput / Pressable.
4. **고정 dp** — 디바이스 크기 다양. Dimensions / 비율.
5. **ScrollView 안 큰 list** — 메모리 폭주. FlatList.
6. **`alert(...)`** — web 의 alert 동작 X. `Alert.alert(...)`.
7. **이미지 size 누락** — 안 보임. `style={{ width: ..., height: ... }}` 필수 (uri 이미지).
8. **FlatList 의 keyExtractor 누락** — index 사용 → 성능 / state 엉킴.

## 18. 다음 단계

- [[stylesheet-and-flexbox]] — StyleSheet 와 Flexbox
- [[platform-differences]] — Platform 분기

## 19. 관련

- [[core-concepts]]
- [[../performance/flatlist]] — FlatList 의 깊은 최적화
- [[safe-area-and-status-bar]]
