---
title: "Permissions-Policy (구 Feature-Policy)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T00:30:00+09:00
tags:
  - network
  - http
  - security
  - permissions-policy
---

# Permissions-Policy (구 Feature-Policy)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 브라우저 기능 제한 |

**[[security|↑ Security]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 한 줄 정의

브라우저의 강력한 **기능 (geolocation / camera / microphone 등) 의 사용** 을 제한.
2020 Feature-Policy → 2021 Permissions-Policy 로 표준화.

```http
Permissions-Policy: geolocation=(), microphone=(), camera=()
```

---

## 2. 동기

### 시나리오
```
1. 사용자가 example.com 방문 (신뢰)
2. example.com 의 iframe 안 evil.com (광고)
3. evil.com 이 navigator.geolocation 시도
4. 사용자에게 권한 요청 — example.com 페이지에서?
   → 사용자 혼동 / 부적절
```

### 방어
```
example.com 의 응답:
Permissions-Policy: geolocation=()

→ example.com 자체 + 모든 iframe 에서 geolocation X
```

---

## 3. 형식

### 기본 — Origin 제한

```http
Permissions-Policy: geolocation=(self)
Permissions-Policy: geolocation=()                    ← 모두 X
Permissions-Policy: geolocation=*                     ← 모두 OK
Permissions-Policy: geolocation=(self "https://maps.com")    ← 특정
```

### 여러 기능

```http
Permissions-Policy:
  geolocation=(self),
  microphone=(),
  camera=(),
  payment=(self "https://stripe.com")
```

---

## 4. 제어 가능한 기능

### 디바이스 접근
- **geolocation** — 위치
- **camera** — 카메라
- **microphone** — 마이크
- **midi** — MIDI
- **usb** — USB
- **bluetooth** — Bluetooth
- **serial** — Serial port

### 센서
- **accelerometer** — 가속도
- **gyroscope** — 자이로
- **magnetometer** — 자기
- **ambient-light-sensor** — 광

### Web Platform
- **payment** — Payment Request API
- **fullscreen** — 전체화면
- **clipboard-read** / **clipboard-write**
- **autoplay** — 자동 재생
- **encrypted-media** — DRM
- **picture-in-picture**
- **screen-wake-lock**
- **xr-spatial-tracking** — VR/AR

### 광고 / 측정
- **interest-cohort** — Google FLoC (deprecated)
- **attribution-reporting**
- **browsing-topics**

### Privacy
- **publickey-credentials-get** — WebAuthn

### Network
- **execution-while-out-of-viewport**
- **execution-while-not-rendered**

→ 100+ 기능. 자세히 → https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Permissions-Policy

---

## 5. 권장 — 보수적

```http
Permissions-Policy:
  geolocation=(),
  microphone=(),
  camera=(),
  payment=(),
  usb=(),
  serial=(),
  bluetooth=(),
  midi=(),
  interest-cohort=(),
  browsing-topics=()
```

→ 모두 차단. 필요할 때 해제.

### 권장 (지도 / 카메라 응용)

```http
Permissions-Policy:
  geolocation=(self "https://maps.googleapis.com"),
  camera=(self),
  microphone=(self),
  payment=(),
  usb=(),
  ...
```

---

## 6. iframe 의 allow 속성

```html
<iframe src="https://maps.example.com"
        allow="geolocation 'self' https://maps.example.com">
</iframe>
```

→ iframe 안에서 geolocation 사용 허용. Permissions-Policy 가 차단 X 인 경우만.

---

## 7. 옛 — Feature-Policy

```http
Feature-Policy: geolocation 'self'; microphone 'none'
```

- 같은 의미 — 문법 다름
- Chrome 86+ — Permissions-Policy 권장
- 옛 호환성 위해 둘 다 보내기도

---

## 8. 서버 설정

### Nginx
```nginx
add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;
```

### Apache
```apache
Header always set Permissions-Policy "geolocation=(), microphone=(), camera=()"
```

### Express
```javascript
app.use((req, res, next) => {
    res.setHeader('Permissions-Policy', 'geolocation=(), microphone=(), camera=()');
    next();
});
```

---

## 9. interest-cohort / browsing-topics

### FLoC (Federated Learning of Cohorts)
- Google 의 광고 추적 대체 시도 (2021)
- 사용자를 "코호트" 로 분류

### 차단
```http
Permissions-Policy: interest-cohort=()
```

→ "이 사이트는 FLoC X". GitHub / Brave 등이 채택.

### 후속 — Topics API
```http
Permissions-Policy: browsing-topics=()
```

→ Topics API 도 차단.

---

## 10. 함정

### 함정 1 — 너무 엄격
정당한 기능 (지도, WebRTC) 차단. 응용 별 맞춤.

### 함정 2 — iframe 의 권한
부모 / iframe 둘 다 명시 필요 (allow 속성).

### 함정 3 — 옛 Feature-Policy
- Chrome 86 미만 — Feature-Policy 사용
- 둘 다 보내는 게 안전

### 함정 4 — 모든 응답에 동일
- 정적 / API / SPA 별 다른 정책
- 응용 코드에서 분기

### 함정 5 — 사용자 권한 vs Policy
- Policy 는 "API 자체" 차단
- 권한 prompt 는 사용자 거부 가능 — 별개

---

## 11. 학습 자료

- **Permissions Policy** — W3C
- MDN Permissions-Policy
- "Introducing Permissions Policy" — web.dev

---

## 12. 관련

- [[security]] — Security hub
- [[csp]] — CSP 와 비교 (다른 영역)
