---
title: "Range Requests — 부분 다운로드"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T01:00:00+09:00
tags:
  - network
  - http
  - performance
  - range
---

# Range Requests — 부분 다운로드

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | Range / 206 / 416 / multipart/byteranges |

**[[performance|↑ Performance]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 한 줄 정의

자원의 **일부 byte 만 요청**. RFC 9110 Section 14. 영상 seeking / 다운로드 재개 /
병렬 다운로드의 기반.

---

## 2. Range 헤더 (요청)

```http
GET /video.mp4 HTTP/1.1
Range: bytes=1000-1999          (1000-1999, 1000 byte)
Range: bytes=0-499              (0-499, 500 byte)
Range: bytes=-500               (마지막 500 byte)
Range: bytes=9500-              (9500 부터 끝)
Range: bytes=0-499, 1000-1499   (multiple ranges)
```

---

## 3. 응답 — 206 Partial Content

```http
HTTP/1.1 206 Partial Content
Content-Range: bytes 1000-1999/5000
Content-Length: 1000
Content-Type: video/mp4

[1000 bytes]
```

### Content-Range
- `bytes <start>-<end>/<total>`
- `bytes 1000-1999/5000`

---

## 4. 응답 시나리오

### 4.1 정상 — 206

```http
GET /file HTTP/1.1
Range: bytes=0-999

HTTP/1.1 206 Partial Content
Content-Range: bytes 0-999/5000
[1000 bytes]
```

### 4.2 Range 무시 — 200 OK

```http
GET /file HTTP/1.1
Range: bytes=0-999

HTTP/1.1 200 OK
Content-Length: 5000
[5000 bytes]
```

→ 서버 / Proxy 가 Range 지원 X. 클라가 결과 전체 받음.

### 4.3 잘못된 Range — 416

```http
GET /file HTTP/1.1
Range: bytes=99999-

HTTP/1.1 416 Range Not Satisfiable
Content-Range: bytes */5000
```

→ 파일 크기 초과. `*/total` 로 알림.

---

## 5. Accept-Ranges (응답)

```http
HTTP/1.1 200 OK
Accept-Ranges: bytes
Content-Length: 5000
```

→ "Range 지원 — bytes 단위". 클라가 후속 Range 가능.

```http
Accept-Ranges: none
```

→ 지원 X.

---

## 6. Multiple Ranges — multipart/byteranges

```http
GET /file HTTP/1.1
Range: bytes=0-499, 1000-1499

HTTP/1.1 206 Partial Content
Content-Type: multipart/byteranges; boundary=X
Content-Length: ...

--X
Content-Type: video/mp4
Content-Range: bytes 0-499/5000

[500 bytes]
--X
Content-Type: video/mp4
Content-Range: bytes 1000-1499/5000

[500 bytes]
--X--
```

→ 응용 / 라이브러리가 multipart 파싱.

---

## 7. 조건부 Range — If-Range

```http
GET /file HTTP/1.1
Range: bytes=1000-1999
If-Range: "v1"          (ETag 또는 HTTP date)
```

### 동작
- ETag 매칭 → 206 (요청한 range)
- 매칭 X → **200 OK + 전체** (자원 변경됨 — 새로 받기)

→ 다운로드 재개 시 자원이 변경됐는지 확인.

---

## 8. 사용 사례

### 8.1 영상 / 오디오 Seeking

```javascript
// HTML5 <video> 가 자동 Range 사용
<video src="video.mp4" controls></video>

// 사용자가 1:30 클릭 → 그 위치부터 Range 요청
```

### 8.2 다운로드 재개

```
다운로드 중 끊김 (3 GB 중 2 GB 받음)
재개:
GET /file HTTP/1.1
Range: bytes=2147483648-
```

### 8.3 병렬 다운로드 (axel, aria2)

```
큰 파일 (1 GB) 을 4 chunk 동시 다운로드:
GET /file: Range: bytes=0-249999999      (250 MB)
GET /file: Range: bytes=250000000-499999999
GET /file: Range: bytes=500000000-749999999
GET /file: Range: bytes=750000000-
```

→ 더 빠른 다운로드 (서버 / 네트워크 허용 시).

### 8.4 PDF / 큰 파일의 부분 로딩

```
PDF.js — 첫 페이지만 Range 로 fetch
나머지는 사용자 스크롤 시
```

### 8.5 Update Diff

```
큰 파일의 변경된 일부만 fetch
```

---

## 9. Range 의 단위

### bytes — 표준
```http
Range: bytes=0-999
```

### 비표준 / 응용 정의
```http
Range: items=0-99       ← 페이지네이션의 일부 (드뭎)
Range: lines=100-200
```

→ 표준은 bytes 만. 다른 단위는 응용 별.

---

## 10. 서버 구현

### Nginx
- 정적 자원 — Range 자동 지원

### Express + 라이브러리
```javascript
const express = require('express');
app.use('/files', express.static('public', {
    // Range 자동
}));

// 또는 직접 구현
app.get('/video', (req, res) => {
    const stat = fs.statSync(filePath);
    const fileSize = stat.size;
    const range = req.headers.range;
    
    if (range) {
        const parts = range.replace(/bytes=/, "").split("-");
        const start = parseInt(parts[0], 10);
        const end = parts[1] ? parseInt(parts[1], 10) : fileSize - 1;
        const chunkSize = (end - start) + 1;
        const file = fs.createReadStream(filePath, {start, end});
        const head = {
            'Content-Range': `bytes ${start}-${end}/${fileSize}`,
            'Accept-Ranges': 'bytes',
            'Content-Length': chunkSize,
            'Content-Type': 'video/mp4',
        };
        res.writeHead(206, head);
        file.pipe(res);
    } else {
        // 전체
        ...
    }
});
```

### S3 / CDN
- 모두 Range 자동 지원
- CDN 의 Range cache 까지

---

## 11. 함정

### 함정 1 — 서버 미지원
일부 응용 / 옛 서버 — Range 무시 + 200 OK 전체. 클라가 fallback.

### 함정 2 — Multiple Range 의 복잡성
multipart/byteranges 파싱 — 라이브러리 의존.

### 함정 3 — Cache + Range
- Range 응답을 CDN 이 어떻게 캐시?
- 일부 CDN 은 첫 요청 시 전체 fetch + Range slice
- 옛 — 비효율

### 함정 4 — If-Range 의 ETag vs Date
ETag 가 정확. Date 는 초 단위.

### 함정 5 — 의도 X 의 큰 Range
악의적 요청 — 매우 작은 chunk 수천 개. Rate limit.

### 함정 6 — Authorization + Range
인증된 자원 — Range 도 동일 인증 필수.

### 함정 7 — Content-Encoding + Range
- gzip 응답에 Range — byte 가 압축 후 byte? 원본 byte?
- RFC: 압축 후 byte (실제는 응용 별)

---

## 12. 학습 자료

- **RFC 9110** Section 14 (Range Requests)
- "How HTTP/2 Pseudo Headers and Range Requests Work" — Cloudflare
- "Video Streaming with HTML5" — web.dev

---

## 13. 관련

- [[performance]] — Performance hub
- [[../status-codes/2xx-success]] — 206 Partial Content
- [[../status-codes/4xx-client-errors]] — 416
- [[../methods/get]] — Range Request
- [[../headers/request-headers]] — Range
- [[../headers/response-headers]] — Accept-Ranges
- [[../headers/entity-headers]] — Content-Range
