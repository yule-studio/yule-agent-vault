---
title: "InAppBrowser — react-native-inappbrowser-reborn"
kind: knowledge
project: mobile
agent: engineering-agent/tech-lead
status: current
tags: [mobile, react-native, inappbrowser, webview, sso]
---

# InAppBrowser

**[[native-features|↑ Native Features Hub]]**

> 외부 web 페이지를 앱 안의 브라우저로. SSO callback, 약관 / 정책 표시.

## 1. WebView vs InAppBrowser

| | WebView | InAppBrowser (SFSafariViewController / Chrome Custom Tabs) |
| --- | --- | --- |
| API | `react-native-webview` | `react-native-inappbrowser-reborn` |
| 쿠키 | 우리 앱과 isolated | **시스템 브라우저 공유** |
| 자동 로그인 | X | **이미 로그인된 SSO 가능** |
| UX | 우리 컴포넌트 | 시스템 표준 (안전감) |
| 커스텀 | 자유 | 제한 |

→ **OAuth / SSO** = InAppBrowser (쿠키 공유).
→ **약관 / 정책 표시** = InAppBrowser (시스템 표준).
→ **앱 안 내장 페이지** = WebView (커스텀).

## 2. 설치

```bash
yarn add react-native-inappbrowser-reborn
cd ios && pod install
```

## 3. 기본

```ts
import InAppBrowser from 'react-native-inappbrowser-reborn';

async function openUrl(url: string) {
  try {
    if (await InAppBrowser.isAvailable()) {
      await InAppBrowser.open(url, {
        // iOS — SFSafariViewController
        dismissButtonStyle: 'cancel',
        preferredBarTintColor: '#0064FF',
        preferredControlTintColor: 'white',
        readerMode: false,
        animated: true,
        modalPresentationStyle: 'fullScreen',
        enableBarCollapsing: false,

        // Android — Chrome Custom Tabs
        showTitle: true,
        toolbarColor: '#0064FF',
        secondaryToolbarColor: 'black',
        navigationBarColor: 'black',
        enableUrlBarHiding: true,
        enableDefaultShare: true,
        forceCloseOnRedirection: false,
      });
    } else {
      Linking.openURL(url);    // fallback
    }
  } catch (e) {
    console.error(e);
  }
}

<Pressable onPress={() => openUrl('https://example.com/terms')}>
  <Text>이용약관</Text>
</Pressable>
```

## 4. OAuth — openAuth (callback 받기)

```ts
async function loginWithOAuth() {
  const url = `https://auth.example.com/login?redirect_uri=myapp://callback`;
  const callbackUrl = 'myapp://callback';

  try {
    const result = await InAppBrowser.openAuth(url, callbackUrl, {
      // iOS / Android 옵션
    });

    if (result.type === 'success' && result.url) {
      const params = new URLSearchParams(result.url.split('?')[1]);
      const code = params.get('code');
      // BE 에 code 전달
      await api.post('/auth/callback', { code });
    }
  } catch (e) {
    console.error(e);
  }
}
```

→ deep link callback + 자동 close.
→ Apple / Google / Kakao native SDK 가 있는 경우는 그쪽 우선. InAppBrowser 는 **그 외 OAuth provider** (Naver, Github 등).

## 5. close

```ts
await InAppBrowser.close();
```

→ 코드로 닫기. 보통 사용자가 닫음 (Done / X 버튼).

## 6. WebView — 자체 페이지

```bash
yarn add react-native-webview
cd ios && pod install
```

```tsx
import { WebView } from 'react-native-webview';

<WebView
  source={{ uri: 'https://example.com' }}
  style={{ flex: 1 }}
  onNavigationStateChange={(navState) => console.log(navState.url)}
  onMessage={(event) => console.log(event.nativeEvent.data)}   // JS bridge
  injectedJavaScript={`window.ReactNativeWebView.postMessage('hi')`}
/>
```

→ deep customization 가능. JS 양방향 bridge.

## 7. job-answer-app-rn 패턴

```tsx
// utils/openLink.ts
import { Linking } from 'react-native';
import InAppBrowser from 'react-native-inappbrowser-reborn';

export async function openInAppBrowser(url: string) {
  try {
    if (await InAppBrowser.isAvailable()) {
      await InAppBrowser.open(url, {
        dismissButtonStyle: 'close',
        toolbarColor: '#0064FF',
        preferredBarTintColor: '#0064FF',
      });
    } else {
      Linking.openURL(url);
    }
  } catch {
    Linking.openURL(url);
  }
}

// 사용
<Pressable onPress={() => openInAppBrowser('https://example.com/terms')}>
  <Text style={{ textDecorationLine: 'underline' }}>이용약관</Text>
</Pressable>
```

## 8. 함정

1. **`isAvailable` 안 체크** — 시스템에 브라우저 없는 경우 fail.
2. **deep link callback 등록 누락** — openAuth 의 redirect 안 받음.
3. **SFSafariViewController 의 limited customization** — iOS UI 변경 제한.
4. **android 의 Chrome 없음** — fallback 동작.
5. **WebView 의 cookie / localStorage 격리** — OAuth 에 부적합.
6. **WebView 의 XSS** — 사용자 입력 페이지 표시 시 sanitize.

## 9. 외부 자료

- [react-native-inappbrowser-reborn](https://github.com/proyecto26/react-native-inappbrowser)
- [react-native-webview](https://github.com/react-native-webview/react-native-webview)

## 10. 관련

- [[native-features]]
- [[../auth/auth]]
- [[../navigation/deep-linking]]
