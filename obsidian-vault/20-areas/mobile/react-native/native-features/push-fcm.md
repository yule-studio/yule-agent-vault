---
title: "Push FCM — Firebase Cloud Messaging + Notifee"
kind: knowledge
project: mobile
agent: engineering-agent/tech-lead
status: current
tags: [mobile, react-native, push, fcm, firebase, notifee, notification]
---

# Push FCM

**[[native-features|↑ Native Features Hub]]**

> Firebase Cloud Messaging — Android + iOS 통합 푸시. job-answer-app-rn 사용.

## 1. 라이브러리

```bash
yarn add @react-native-firebase/app
yarn add @react-native-firebase/messaging
yarn add @notifee/react-native           # 로컬 알림 + display
cd ios && pod install
```

→ FCM 으로 전송 + Notifee 로 표시.

## 2. Firebase 콘솔 설정

1. **console.firebase.google.com** → 프로젝트 생성.
2. **iOS 앱 추가** → Bundle ID 입력 → `GoogleService-Info.plist` 다운로드 → `ios/<AppName>/` 에 배치.
3. **Android 앱 추가** → package name + SHA-1 → `google-services.json` 다운로드 → `android/app/` 에 배치.
4. **Cloud Messaging** 활성화.

## 3. iOS — APNs 설정

1. Apple Developer Console → Keys → `+` → Apple Push Notifications service (APNs) → 생성 → `.p8` 다운로드.
2. Firebase Console → 프로젝트 설정 → Cloud Messaging → APNs 인증 키 업로드.
3. Xcode → Signing & Capabilities → `+ Capability` → Push Notifications + Background Modes (Remote notifications).

## 4. Android — `google-services` plugin

`android/build.gradle`:
```gradle
buildscript {
  dependencies {
    classpath 'com.google.gms:google-services:4.4.0'
  }
}
```

`android/app/build.gradle`:
```gradle
apply plugin: 'com.google.gms.google-services'
```

`AndroidManifest.xml`:
```xml
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />   <!-- 13+ -->

<application ...>
  <meta-data
    android:name="com.google.firebase.messaging.default_notification_channel_id"
    android:value="default" />
</application>
```

## 5. 권한 요청

```ts
import messaging from '@react-native-firebase/messaging';
import { Platform, PermissionsAndroid } from 'react-native';

async function requestNotificationPermission() {
  if (Platform.OS === 'ios') {
    const status = await messaging().requestPermission();
    return (
      status === messaging.AuthorizationStatus.AUTHORIZED ||
      status === messaging.AuthorizationStatus.PROVISIONAL
    );
  } else {
    // Android 13+
    if (Platform.Version >= 33) {
      const status = await PermissionsAndroid.request(
        PermissionsAndroid.PERMISSIONS.POST_NOTIFICATIONS,
      );
      return status === PermissionsAndroid.RESULTS.GRANTED;
    }
    return true;
  }
}
```

## 6. FCM 토큰 받기 + BE 등록

```ts
import messaging from '@react-native-firebase/messaging';

const token = await messaging().getToken();
console.log('FCM token:', token);

// BE 에 등록
await api.post('/devices', { fcmToken: token, platform: Platform.OS });

// 토큰 변경 감지
messaging().onTokenRefresh((newToken) => {
  api.put('/devices', { fcmToken: newToken });
});
```

→ token 은 디바이스 / 앱 install 단위. 새로 깔면 새 token.

## 7. foreground 메시지 — 알림 표시 안 됨 (기본)

iOS / Android 의 기본 — 앱이 켜져 있으면 알림 안 보임. **수동 표시 필요**.

```ts
import messaging from '@react-native-firebase/messaging';
import notifee from '@notifee/react-native';

useEffect(() => {
  const unsub = messaging().onMessage(async (msg) => {
    // foreground 에서 받음 — 직접 표시
    await notifee.displayNotification({
      title: msg.notification?.title,
      body: msg.notification?.body,
      android: { channelId: 'default' },
    });
  });
  return unsub;
}, []);
```

### Android 채널 (필수)
```ts
useEffect(() => {
  notifee.createChannel({
    id: 'default',
    name: 'Default',
    importance: AndroidImportance.HIGH,
  });
}, []);
```

## 8. background / quit 메시지

```ts
// background — 자동 알림 표시 (앱은 닫혀 있지만 OS 가 표시)
messaging().setBackgroundMessageHandler(async (msg) => {
  console.log('background', msg);
  // 데이터 처리
});

// 알림 클릭으로 앱 진입 (background → foreground)
messaging().onNotificationOpenedApp((msg) => {
  if (msg.data?.screen === 'detail') {
    navigation.navigate('Detail', { id: msg.data.id });
  }
});

// 알림 클릭으로 앱 시작 (quit → 처음 launch)
messaging().getInitialNotification().then((msg) => {
  if (msg) handleNotification(msg);
});
```

→ 3 가지 상태:
1. **foreground** — `onMessage`.
2. **background** — OS 가 자동 표시 (notification payload 만), 클릭 시 `onNotificationOpenedApp`.
3. **quit** — `getInitialNotification`.

## 9. notification vs data payload

```json
// FCM 페이로드
{
  "notification": {
    "title": "안녕",
    "body": "내용"
  },
  "data": {
    "screen": "detail",
    "id": "5"
  }
}
```

- `notification` — OS 가 자동 표시 (background / quit).
- `data` — 우리 코드가 처리. silent push 가능.

→ 둘 다 보내고 우리 코드에서 추가 처리.

## 10. notifee — 로컬 알림

```bash
yarn add @notifee/react-native
```

```ts
import notifee, { AndroidImportance, TriggerType } from '@notifee/react-native';

// 즉시 표시
await notifee.displayNotification({
  title: '제목',
  body: '본문',
  android: {
    channelId: 'default',
    smallIcon: 'ic_notification',
    pressAction: { id: 'default' },
  },
  ios: {
    sound: 'default',
    foregroundPresentationOptions: { alert: true, badge: true, sound: true },
  },
});

// 스케줄 (예: 1시간 뒤)
await notifee.createTriggerNotification(
  { title: '리마인드', body: '...' },
  { type: TriggerType.TIMESTAMP, timestamp: Date.now() + 60 * 60 * 1000 },
);
```

## 11. 배지 (앱 아이콘의 숫자)

```ts
import notifee from '@notifee/react-native';

await notifee.setBadgeCount(5);
await notifee.incrementBadgeCount();
await notifee.decrementBadgeCount();
await notifee.setBadgeCount(0);    // 클리어
```

## 12. 알림 클릭 처리

```ts
import notifee, { EventType } from '@notifee/react-native';

// foreground
notifee.onForegroundEvent(({ type, detail }) => {
  if (type === EventType.PRESS) {
    const screen = detail.notification?.data?.screen;
    if (screen === 'detail') navigation.navigate('Detail', { id: detail.notification.data.id });
  }
});

// background
notifee.onBackgroundEvent(async ({ type, detail }) => {
  // 백그라운드 처리 (analytics 등)
});
```

## 13. job-answer-app-rn 패턴

```tsx
// hooks/useFCM.ts
import { useEffect } from 'react';
import messaging from '@react-native-firebase/messaging';
import notifee, { AndroidImportance } from '@notifee/react-native';

export function useFCM() {
  useEffect(() => {
    (async () => {
      await notifee.createChannel({
        id: 'default',
        name: 'Default',
        importance: AndroidImportance.HIGH,
      });

      const status = await messaging().requestPermission();
      if (status !== messaging.AuthorizationStatus.AUTHORIZED) return;

      const token = await messaging().getToken();
      await api.post('/devices', { fcmToken: token });

      messaging().onTokenRefresh((t) => api.put('/devices', { fcmToken: t }));
    })();

    const unsubMsg = messaging().onMessage(async (msg) => {
      await notifee.displayNotification({
        title: msg.notification?.title,
        body: msg.notification?.body,
        android: { channelId: 'default' },
        data: msg.data,
      });
    });

    const unsubOpened = messaging().onNotificationOpenedApp((msg) => {
      handleNotificationOpen(msg);
    });

    return () => {
      unsubMsg();
      unsubOpened();
    };
  }, []);
}
```

## 14. 함정

1. **APNs key 미등록** — iOS 푸시 안 옴.
2. **`google-services.json` / `.plist` 누락** — Firebase init 실패.
3. **Android 13+ POST_NOTIFICATIONS** — 권한 요청 안 하면 표시 X.
4. **Android 채널 누락** — 8.0+ 부터 채널 없으면 표시 X.
5. **foreground 알림 안 보임** — 의도된 동작. notifee 로 표시.
6. **token refresh 누락** — 토큰 변경 후 BE 안 알면 알림 못 받음.
7. **sandbox APNs vs production** — TestFlight = production, debug = sandbox. Firebase 설정 확인.
8. **simulator 푸시 X** — iOS sim 은 푸시 직접 불가. iOS 16+ 의 `.apns` 파일 drop 가능.

## 15. 외부 자료

- [@react-native-firebase/messaging](https://rnfirebase.io/messaging/usage)
- [@notifee/react-native](https://notifee.app/react-native/docs/overview)
- [FCM 문서](https://firebase.google.com/docs/cloud-messaging)

## 16. 관련

- [[native-features]]
- [[permissions]]
- [[../navigation/deep-linking]]
