---
title: "Localize / i18n — 다국어 + 시스템 로캘"
kind: knowledge
project: mobile
agent: engineering-agent/tech-lead
status: current
tags: [mobile, react-native, i18n, localize, korean, english]
---

# Localize / i18n

**[[native-features|↑ Native Features Hub]]**

> 한 + 영 + 일 등 다국어. job-answer-app-rn 사용 (`i18n-js` + `react-native-localize`).

## 1. 라이브러리

```bash
yarn add i18n-js
yarn add react-native-localize
cd ios && pod install
```

- **i18n-js** — 번역 string 관리.
- **react-native-localize** — 시스템 로캘 / 통화 / 숫자 포맷.

## 2. 번역 파일

```ts
// i18n/translations/ko.ts
export const ko = {
  common: {
    save: '저장',
    cancel: '취소',
    delete: '삭제',
  },
  login: {
    title: '로그인',
    email: '이메일',
    password: '비밀번호',
    submit: '로그인',
  },
};

// i18n/translations/en.ts
export const en = {
  common: {
    save: 'Save',
    cancel: 'Cancel',
    delete: 'Delete',
  },
  login: {
    title: 'Login',
    email: 'Email',
    password: 'Password',
    submit: 'Log In',
  },
};
```

→ 동일 구조 + key 일치.

## 3. i18n setup

```ts
// i18n/index.ts
import { I18n } from 'i18n-js';
import * as Localize from 'react-native-localize';
import { ko } from './translations/ko';
import { en } from './translations/en';

const i18n = new I18n({ ko, en });

// 시스템 로캘 감지
const locales = Localize.getLocales();
i18n.locale = locales[0]?.languageCode ?? 'ko';

// fallback
i18n.enableFallback = true;
i18n.defaultLocale = 'ko';

export default i18n;
```

## 4. 사용

```tsx
import i18n from '@/i18n';

<Text>{i18n.t('common.save')}</Text>            // "저장"
<Text>{i18n.t('login.title')}</Text>             // "로그인"

// 변수
<Text>{i18n.t('greeting', { name: '유철' })}</Text>
// translations: greeting: '안녕, %{name}!'
```

## 5. 언어 변경

```ts
// 사용자가 설정에서 선택
import AsyncStorage from '@react-native-async-storage/async-storage';
import i18n from '@/i18n';

async function changeLanguage(lang: 'ko' | 'en') {
  i18n.locale = lang;
  await AsyncStorage.setItem('user_lang', lang);
  // 모든 화면 re-render — context 또는 zustand 의 trigger
}

// 앱 시작 시 복원
const savedLang = await AsyncStorage.getItem('user_lang');
if (savedLang) i18n.locale = savedLang;
```

→ **i18n.locale 변경만으로는 자동 re-render X**. context / zustand 로 trigger.

## 6. context + i18n — re-render

```tsx
import { createContext, useContext, useState } from 'react';
import i18n from '@/i18n';

const I18nContext = createContext({ t: i18n.t.bind(i18n), changeLanguage: (l: string) => {} });

export function I18nProvider({ children }) {
  const [locale, setLocale] = useState(i18n.locale);

  const changeLanguage = (l: string) => {
    i18n.locale = l;
    setLocale(l);
    AsyncStorage.setItem('user_lang', l);
  };

  return (
    <I18nContext.Provider value={{ t: i18n.t.bind(i18n), changeLanguage, locale }}>
      {children}
    </I18nContext.Provider>
  );
}

export const useI18n = () => useContext(I18nContext);
```

```tsx
// 사용
const { t, changeLanguage } = useI18n();
<Text>{t('common.save')}</Text>
<Button title="English" onPress={() => changeLanguage('en')} />
```

## 7. react-native-localize — 시스템 정보

```ts
import * as Localize from 'react-native-localize';

const locales = Localize.getLocales();
// [{ countryCode: 'KR', languageTag: 'ko-KR', languageCode: 'ko', isRTL: false }, ...]

const country = Localize.getCountry();        // 'KR'
const timezone = Localize.getTimeZone();      // 'Asia/Seoul'
const currencies = Localize.getCurrencies(); // ['KRW']
const calendar = Localize.getCalendar();      // 'gregorian'
const usesMetric = Localize.usesMetricSystem(); // true
const is24Hour = Localize.uses24HourClock();
const isRTL = Localize.isRTL();
const number = Localize.getNumberFormatSettings();
// { decimalSeparator: '.', groupingSeparator: ',' }

// 로캘 변경 감지
Localize.addEventListener('change', () => {
  // 시스템 언어 변경 시
});
```

## 8. 시스템 로캘 follow vs 사용자 선택

```ts
// 옵션 1 — 시스템 따라
i18n.locale = Localize.getLocales()[0].languageCode;

// 옵션 2 — 사용자 선택 우선, 없으면 시스템
const userLang = await AsyncStorage.getItem('user_lang');
i18n.locale = userLang ?? Localize.getLocales()[0].languageCode;
```

→ "설정 → 언어" 에서 사용자가 변경 가능하게.

## 9. 숫자 / 통화 / 날짜 — Intl API

```ts
// 통화
new Intl.NumberFormat('ko-KR', { style: 'currency', currency: 'KRW' }).format(1234567);
// "₩1,234,567"

// 숫자
new Intl.NumberFormat('ko-KR').format(1234567);
// "1,234,567"

// 날짜
new Intl.DateTimeFormat('ko-KR', { dateStyle: 'long' }).format(new Date());
// "2026년 5월 14일"

// 상대 시간
new Intl.RelativeTimeFormat('ko').format(-3, 'day');
// "3일 전"
```

→ RN 0.71+ 의 Hermes 가 Intl 지원. 더 옛 버전이면 `intl` polyfill.

## 10. 복수형 (plural)

```ts
// translations
// inbox: { unread: { one: '%{count} 개의 새 메시지', other: '%{count} 개의 새 메시지들' } }

i18n.t('inbox.unread', { count: 1 });    // "1 개의 새 메시지"
i18n.t('inbox.unread', { count: 5 });    // "5 개의 새 메시지들"
```

한국어는 단수/복수 구분이 약함. 영어 등은 의미 있음.

## 11. RTL (Right-to-Left) — 아랍어, 히브리어

```tsx
import { I18nManager } from 'react-native';

if (Localize.isRTL()) {
  I18nManager.forceRTL(true);
} else {
  I18nManager.forceRTL(false);
}

// 변경 후 reload 필요
RNRestart.Restart();
```

→ RTL 의 자동 flex direction 반전.

## 12. job-answer-app-rn 패턴

```ts
// src/i18n/index.ts
import { I18n } from 'i18n-js';
import * as Localize from 'react-native-localize';
import ko from './ko.json';
import en from './en.json';
import ja from './ja.json';

const i18n = new I18n({ ko, en, ja });
i18n.locale = Localize.getLocales()[0]?.languageCode ?? 'ko';
i18n.enableFallback = true;
i18n.defaultLocale = 'ko';

export default i18n;

// 사용
import i18n from '@/i18n';
<Text>{i18n.t('login.title')}</Text>
```

```tsx
// pages/setting/_pages/language
// 사용자가 ko/en/ja 선택 가능
<Pressable onPress={() => changeLanguage('en')}>
  <Text>English</Text>
</Pressable>
```

## 13. 다른 옵션 — i18next + react-i18next

```bash
yarn add i18next react-i18next
```

- 더 강력 (interpolation, namespace, lazy load).
- 학습 곡선 약간 더.
- 큰 앱에 권장.

`i18n-js` = 더 단순. 작은 앱에.

## 14. 함정

1. **locale 변경 후 re-render X** — context/store 로 trigger.
2. **key 누락** — 한 언어 파일에만 있고 다른 언어 없음. fallback 으로 영어 표시.
3. **하드코드 string** — 다국어 적용 누락. lint 으로 검사 (`eslint-plugin-i18n-json` 등).
4. **plural rule 잘못** — 언어마다 다름 (ICU 표준).
5. **시스템 로캘 만 따름** — 사용자 선택 옵션 없음. 한국 앱은 영어 OS 의 사용자도 한국어 원함.
6. **RTL forceRTL 의 restart** — 즉시 반영 안 됨. dev mode 와 production 다름.

## 15. 외부 자료

- [i18n-js](https://github.com/fnando/i18n)
- [react-native-localize](https://github.com/zoontek/react-native-localize)
- [i18next](https://www.i18next.com/)

## 16. 관련

- [[native-features]]
- [[../state/async-storage]] — 언어 설정 저장
