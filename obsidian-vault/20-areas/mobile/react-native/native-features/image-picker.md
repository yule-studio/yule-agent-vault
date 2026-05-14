---
title: "Image Picker — react-native-image-picker"
kind: knowledge
project: mobile
agent: engineering-agent/tech-lead
status: current
tags: [mobile, react-native, image-picker, camera, gallery]
---

# Image Picker

**[[native-features|↑ Native Features Hub]]**

> 카메라 / 갤러리에서 이미지 선택. job-answer-app-rn 사용.

## 1. 설치

```bash
yarn add react-native-image-picker
cd ios && pod install
```

## 2. iOS / Android 권한 (필수)

### iOS — Info.plist
```xml
<key>NSCameraUsageDescription</key>
<string>프로필 사진 촬영을 위해 카메라가 필요합니다.</string>
<key>NSPhotoLibraryUsageDescription</key>
<string>사진 첨부를 위해 사진 접근이 필요합니다.</string>
<key>NSMicrophoneUsageDescription</key>
<string>비디오 녹화를 위해 마이크가 필요합니다.</string>
```

### Android — Manifest
```xml
<uses-permission android:name="android.permission.CAMERA" />
<uses-permission android:name="android.permission.READ_MEDIA_IMAGES" />
```

→ 자세히 [[permissions]].

## 3. 카메라 촬영

```tsx
import { launchCamera } from 'react-native-image-picker';

async function takePhoto() {
  const result = await launchCamera({
    mediaType: 'photo',
    quality: 0.8,
    saveToPhotos: true,                // 갤러리에도 저장
    cameraType: 'back',
  });

  if (result.didCancel) return;
  if (result.errorCode) {
    Alert.alert('에러', result.errorMessage);
    return;
  }

  const asset = result.assets?.[0];
  if (!asset) return;

  console.log(asset.uri);              // 'file:///...' 또는 'content://...'
  console.log(asset.fileName);
  console.log(asset.type);             // 'image/jpeg'
  console.log(asset.width, asset.height);
  console.log(asset.fileSize);

  setPhoto(asset.uri);
}
```

## 4. 갤러리에서 선택

```tsx
import { launchImageLibrary } from 'react-native-image-picker';

async function pickFromGallery() {
  const result = await launchImageLibrary({
    mediaType: 'photo',
    quality: 0.8,
    selectionLimit: 1,                 // 0 = 무제한
  });

  if (result.didCancel) return;
  const asset = result.assets?.[0];
  if (!asset) return;
  setPhoto(asset.uri);
}
```

### 여러 장
```tsx
const result = await launchImageLibrary({
  mediaType: 'photo',
  selectionLimit: 5,
});

const photos = result.assets ?? [];
setPhotos(photos.map(a => a.uri!));
```

## 5. 옵션

```ts
launchImageLibrary({
  mediaType: 'photo' | 'video' | 'mixed',
  quality: 0.0 ~ 1.0,
  maxWidth: 1024,
  maxHeight: 1024,
  videoQuality: 'low' | 'medium' | 'high',
  durationLimit: 60,                   // 비디오 초
  saveToPhotos: false,
  selectionLimit: 1,
  presentationStyle: 'fullScreen' | 'pageSheet' | 'overFullScreen',  // iOS
  includeBase64: false,                // true 면 base64 string 도 반환 (큰 메모리)
  includeExtra: true,                  // EXIF, location 등
});
```

## 6. 미리보기 + 업로드

```tsx
function ProfilePhoto() {
  const [uri, setUri] = useState<string | null>(null);
  const [uploading, setUploading] = useState(false);

  const pick = async () => {
    const result = await launchImageLibrary({ mediaType: 'photo', quality: 0.8 });
    const asset = result.assets?.[0];
    if (!asset?.uri) return;
    setUri(asset.uri);

    // 업로드
    const fd = new FormData();
    fd.append('file', {
      uri: asset.uri,
      type: asset.type ?? 'image/jpeg',
      name: asset.fileName ?? 'photo.jpg',
    } as any);

    setUploading(true);
    try {
      const { data } = await api.post('/upload', fd, {
        headers: { 'Content-Type': 'multipart/form-data' },
        onUploadProgress: (e) => { /* progress */ },
      });
      console.log('uploaded', data.url);
    } finally {
      setUploading(false);
    }
  };

  return (
    <Pressable onPress={pick}>
      {uri ? <Image source={{ uri }} style={{ width: 100, height: 100 }} /> : <Text>+</Text>}
    </Pressable>
  );
}
```

## 7. 비디오

```tsx
launchCamera({ mediaType: 'video', videoQuality: 'medium', durationLimit: 30 });

launchImageLibrary({ mediaType: 'video' });

// 결과
asset.type    // 'video/mp4'
asset.duration // 초
```

## 8. 카메라 + 갤러리 선택 dialog

```tsx
const showImagePicker = () => {
  Alert.alert('사진 선택', '', [
    { text: '카메라', onPress: () => launchCamera(...) },
    { text: '갤러리', onPress: () => launchImageLibrary(...) },
    { text: '취소', style: 'cancel' },
  ]);
};
```

→ 또는 자체 BottomSheet 컴포넌트.

## 9. 권한 자동 처리 — 라이브러리가 알아서

`react-native-image-picker` 는 권한 자동 요청. 단 **Info.plist / Manifest 의 메시지는 우리가** 채워야.

직접 권한 처리 원하면 [[permissions]] 의 패턴 사용.

## 10. 이미지 압축 / 변환

큰 이미지 (수 MB) → 업로드 전 압축:

```bash
yarn add react-native-image-resizer
```

```tsx
import ImageResizer from '@bam.tech/react-native-image-resizer';

const result = await ImageResizer.createResizedImage(
  asset.uri,
  1024,    // maxWidth
  1024,    // maxHeight
  'JPEG',
  80,      // quality
);
console.log(result.uri);
```

## 11. 권한 없음 → BLOCKED 처리

```tsx
const result = await launchCamera({ ... });
if (result.errorCode === 'permission') {
  Alert.alert('권한 필요', '설정에서 카메라 권한을 켜주세요.', [
    { text: '취소', style: 'cancel' },
    { text: '설정 열기', onPress: () => Linking.openSettings() },
  ]);
}
```

## 12. job-answer-app-rn 패턴

```tsx
// hooks/useImagePicker.ts
import { launchCamera, launchImageLibrary } from 'react-native-image-picker';

export function useImagePicker() {
  const pickFromGallery = async () => {
    const result = await launchImageLibrary({
      mediaType: 'photo',
      quality: 0.8,
      maxWidth: 1024,
      maxHeight: 1024,
    });
    return result.assets?.[0];
  };

  const takePhoto = async () => {
    const result = await launchCamera({
      mediaType: 'photo',
      quality: 0.8,
      cameraType: 'back',
    });
    return result.assets?.[0];
  };

  return { pickFromGallery, takePhoto };
}
```

## 13. 함정

1. **권한 메시지 누락** — iOS 첫 사용 시 강제 종료.
2. **`includeBase64: true` 의 메모리** — 큰 이미지면 OOM. uri 만 사용.
3. **Android 13+ 의 READ_MEDIA_IMAGES** — 옛 `READ_EXTERNAL_STORAGE` 대신.
4. **iOS 의 maxWidth / Height 무시** — 일부 case 에서 동작 안 함. ImageResizer 권장.
5. **`saveToPhotos` 의 권한** — iOS 의 add usage 권한.
6. **video 의 큰 size** — 업로드 전 압축 (ffmpeg).
7. **사용자가 갤러리 일부만 허용 (iOS limited)** — 결과 동일하지만 다음 fetch 못 함.

## 14. 외부 자료

- [react-native-image-picker](https://github.com/react-native-image-picker/react-native-image-picker)
- [react-native-image-resizer](https://github.com/bamlab/react-native-image-resizer)

## 15. 관련

- [[native-features]]
- [[permissions]]
- [[../networking/axios-fetch]] — FormData 업로드
