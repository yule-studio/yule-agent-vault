---
title: "Voice STT — @react-native-voice/voice"
kind: knowledge
project: mobile
agent: engineering-agent/tech-lead
status: current
tags: [mobile, react-native, voice, stt, speech-to-text]
---

# Voice STT (Speech-to-Text)

**[[native-features|↑ Native Features Hub]]**

> iOS Speech Framework + Android SpeechRecognizer 의 native bridge. job-answer-app-rn 의 음성 답변.

## 1. 라이브러리

```bash
yarn add @react-native-voice/voice
cd ios && pod install
```

## 2. 권한 (필수)

### iOS — Info.plist
```xml
<key>NSMicrophoneUsageDescription</key>
<string>음성 답변을 위해 마이크가 필요합니다.</string>
<key>NSSpeechRecognitionUsageDescription</key>
<string>음성을 텍스트로 변환하기 위해 음성 인식이 필요합니다.</string>
```

→ **두 권한 모두 필수** (마이크 + 음성인식).

### Android — Manifest
```xml
<uses-permission android:name="android.permission.RECORD_AUDIO" />
<uses-permission android:name="android.permission.INTERNET" />
```

Android 의 음성 인식은 Google 의 서비스 사용 — **인터넷 필요** (오프라인 X 일반적).

## 3. 기본 사용

```tsx
import Voice from '@react-native-voice/voice';
import { useEffect, useState } from 'react';

function VoiceInput() {
  const [text, setText] = useState('');
  const [isListening, setIsListening] = useState(false);

  useEffect(() => {
    Voice.onSpeechStart = () => setIsListening(true);
    Voice.onSpeechEnd = () => setIsListening(false);
    Voice.onSpeechResults = (e) => {
      setText(e.value?.[0] ?? '');
    };
    Voice.onSpeechPartialResults = (e) => {
      setText(e.value?.[0] ?? '');
    };
    Voice.onSpeechError = (e) => {
      console.error(e);
      setIsListening(false);
    };

    return () => {
      Voice.destroy().then(Voice.removeAllListeners);
    };
  }, []);

  const start = async () => {
    try {
      await Voice.start('ko-KR');     // 또는 'en-US'
    } catch (e) {
      console.error(e);
    }
  };

  const stop = async () => {
    try {
      await Voice.stop();
    } catch (e) {
      console.error(e);
    }
  };

  return (
    <View>
      <Text>{text}</Text>
      <Pressable onPress={isListening ? stop : start}>
        <Text>{isListening ? '중지' : '말하기'}</Text>
      </Pressable>
    </View>
  );
}
```

## 4. 이벤트 종류

| | 의미 |
| --- | --- |
| `onSpeechStart` | 음성 인식 시작 |
| `onSpeechRecognized` | 음성이 감지됨 |
| `onSpeechEnd` | 음성 인식 종료 |
| `onSpeechResults` | 최종 결과 (배열, value[0] 가 best) |
| `onSpeechPartialResults` | 중간 결과 (실시간 표시) |
| `onSpeechError` | 에러 |
| `onSpeechVolumeChanged` | 마이크 입력 볼륨 (UI 시각화) |

## 5. 권한 + start

```tsx
import { request, PERMISSIONS, RESULTS } from 'react-native-permissions';

const start = async () => {
  const status = await request(
    Platform.select({
      ios: PERMISSIONS.IOS.MICROPHONE,
      android: PERMISSIONS.ANDROID.RECORD_AUDIO,
    })!
  );

  if (Platform.OS === 'ios') {
    const speechStatus = await request(PERMISSIONS.IOS.SPEECH_RECOGNITION);
    if (speechStatus !== RESULTS.GRANTED) return;
  }

  if (status !== RESULTS.GRANTED) return;

  await Voice.start('ko-KR');
};
```

## 6. 언어 코드

| 코드 | 언어 |
| --- | --- |
| `ko-KR` | 한국어 |
| `en-US` | 영어 (미국) |
| `en-GB` | 영어 (영국) |
| `ja-JP` | 일본어 |
| `zh-CN` | 중국어 (간체) |
| `es-ES` | 스페인어 |

```ts
// 사용 가능한 언어 확인
const services = await Voice.getSpeechRecognitionServices();    // Android
const availableLanguages = ...  // iOS 는 시스템 의존
```

## 7. 볼륨 시각화

```tsx
const [volume, setVolume] = useState(0);

useEffect(() => {
  Voice.onSpeechVolumeChanged = (e) => {
    setVolume(e.value ?? 0);
  };
}, []);

<View style={{ width: 50 + volume * 10, height: 50, borderRadius: 25, backgroundColor: 'blue' }} />
```

→ 또는 [[../ui/lottie|Lottie]] 의 마이크 펄스 애니메이션.

## 8. 결과 처리

```tsx
Voice.onSpeechResults = (e) => {
  const transcripts = e.value ?? [];
  // transcripts[0] = 가장 confident
  // transcripts[1], [2] = alternative
  setText(transcripts[0] ?? '');
};
```

→ 사용자가 alternative 중 선택할 UI 가능.

## 9. 길게 듣기 / 자동 종료

- iOS — 약 1분 또는 침묵 후 자동 종료.
- Android — 약 30초 또는 침묵 후 자동 종료.

긴 음성:
- 종료 후 자동 재시작 (체인).
- 또는 다른 lib (e.g. Whisper local model, 자체 서버).

## 10. job-answer-app-rn 패턴 — 음성 답변

```tsx
// components/VoiceInputBar
import Voice from '@react-native-voice/voice';
import LottieView from 'lottie-react-native';

export function VoiceInputBar({ onResult }) {
  const [isRecording, setIsRecording] = useState(false);
  const [partial, setPartial] = useState('');

  useEffect(() => {
    Voice.onSpeechStart = () => setIsRecording(true);
    Voice.onSpeechEnd = () => setIsRecording(false);
    Voice.onSpeechPartialResults = (e) => setPartial(e.value?.[0] ?? '');
    Voice.onSpeechResults = (e) => {
      const final = e.value?.[0] ?? '';
      onResult(final);
      setPartial('');
    };
    Voice.onSpeechError = (e) => {
      Alert.alert('음성 인식 실패');
      setIsRecording(false);
    };

    return () => {
      Voice.destroy().then(Voice.removeAllListeners);
    };
  }, [onResult]);

  const toggle = async () => {
    if (isRecording) {
      await Voice.stop();
    } else {
      // 권한 ensure
      await Voice.start('ko-KR');
    }
  };

  return (
    <View>
      {partial && <Text style={{ color: 'gray' }}>{partial}</Text>}
      <Pressable onPress={toggle}>
        {isRecording ? (
          <LottieView source={require('@assets/lottie/recording.json')} autoPlay loop style={{ width: 60, height: 60 }} />
        ) : (
          <Icon name="mic" size={32} />
        )}
      </Pressable>
    </View>
  );
}
```

## 11. 함정

1. **두 권한 누락** — iOS 는 마이크 + speech 두 권한.
2. **인터넷 없음 (Android)** — 음성 인식 fail.
3. **이벤트 unmount cleanup 누락** — 메모리 누수 + 이중 처리.
4. **Voice.start 의 throw** — 권한 거부 / busy. try/catch.
5. **partial vs final 혼동** — partial 은 매번 갱신. final 은 1 회.
6. **언어 지원 안 됨** — `getSpeechRecognitionServices` 로 확인.
7. **시뮬레이터 의 인식** — 시뮬레이터는 시스템 음성 사용. 실기기 권장.
8. **백그라운드** — 종료됨. continuous recording 은 native module 추가 필요.

## 12. 다른 옵션

- **OpenAI Whisper** — 서버 측 또는 ONNX 모델. 더 정확.
- **Google Cloud Speech-to-Text** — 서버 측 API.
- **Apple Speech Framework** native (직접 호출).

→ `@react-native-voice/voice` 는 가장 간단. 정확도 필요 시 다른 옵션.

## 13. 외부 자료

- [@react-native-voice/voice](https://github.com/react-native-voice/voice)
- [iOS Speech Framework](https://developer.apple.com/documentation/speech)
- [Android SpeechRecognizer](https://developer.android.com/reference/android/speech/SpeechRecognizer)

## 14. 관련

- [[native-features]]
- [[permissions]]
- [[../ui/lottie]] — 녹음 애니메이션
