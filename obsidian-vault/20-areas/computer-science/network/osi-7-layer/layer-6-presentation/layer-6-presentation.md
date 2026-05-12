---
title: "L6 — 표현 계층 (Presentation Layer)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T17:10:00+09:00
tags:
  - network
  - osi
  - layer-6
  - encoding
  - compression
---

# L6 — 표현 계층 (Presentation Layer)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 인코딩 / 압축 / 암호화 / MIME / 직렬화 |

**[[../osi-7-layer|↑ OSI 7 계층]]** · **[[../../network|↑↑ network hub]]**

---

## 메타 표

| 항목 | 값 |
| --- | --- |
| **전송 단위** | data |
| **주요 기능** | 인코딩 / 압축 / 암호화 / 직렬화 |
| **대표 프로토콜** | ASCII, UTF-8, MIME, JPEG, MP3, MPEG, TLS (애매), JSON, Protobuf |

---

## 1. 한 줄 정의

**응용 데이터의 표현 / 변환** — 송신자와 수신자의 표현 차이 해소. 인코딩,
압축, 암호화가 여기 들어가지만 실제로는 응용 (L7) 에 흡수.

---

## 2. L6 의 3 가지 기능

1. **인코딩** (Translation) — ASCII ↔ UTF-8, big/little endian
2. **압축** (Compression) — 데이터 크기 축소
3. **암호화** (Encryption) — TLS 의 일부

---

## 3. 문자 인코딩

### 3.1 ASCII (1963)
- 7 bit (128 문자)
- 영문 + 숫자 + 제어
- 옛 표준, 호환성 최고

### 3.2 ISO 8859 시리즈
- 8 bit (256 문자)
- ISO 8859-1 (Latin-1), 8859-7 (그리스), 8859-5 (러시아)
- 지역별 — 다국어 어려움

### 3.3 한국 인코딩

| 인코딩 | 설명 |
| --- | --- |
| **KS X 1001** | 1987, KS C 5601 → KS X 1001 |
| **EUC-KR** | KS X 1001 기반, 2 byte 한글 |
| **CP949** (MS) | EUC-KR 확장, 11172 한글 글자 |
| **UTF-8** | 모던 표준 |
| **UTF-16** | Java 내부 / Windows API |

### 3.4 일본 인코딩
- **Shift_JIS** — 옛 Microsoft
- **EUC-JP**
- **ISO-2022-JP** — 메일
- **UTF-8** — 모던

### 3.5 중국 인코딩
- **GB2312** — 옛 (간체)
- **GBK** — 확장
- **GB18030** — 정부 표준 (유니코드 완전)
- **Big5** — 번체 (대만 / 홍콩)
- **UTF-8**

### 3.6 Unicode

- **U+0000 ~ U+10FFFF** (1.1M 코드포인트)
- 17 plane × 65536
- 한글: U+AC00 ~ U+D7A3 (11172 자)

#### UTF-8 (Ken Thompson / Rob Pike 1992)
- 가변 길이 1-4 byte
- ASCII 호환 (7 bit ASCII 그대로)
- **웹 표준** (HTTP, HTML, JSON, ...)

```
U+0041 (A) → 0x41              (1 byte)
U+00E9 (é) → 0xC3 0xA9          (2 byte)
U+4F60 (你) → 0xE4 0xBD 0xA0     (3 byte)
U+1F600 (😀) → 0xF0 0x9F 0x98 0x80   (4 byte)
```

#### UTF-16
- 2 byte 기본 + Surrogate Pair (4 byte)
- Java `String`, JavaScript `string`, Windows API

#### UTF-32
- 4 byte 고정
- 메모리 낭비, 잘 안 씀

자세히 → [[../../../data-structure/arrays-and-strings/arrays-and-strings|↗ arrays-and-strings]] 의 인코딩 섹션

---

## 4. 데이터 직렬화 (Serialization)

### 4.1 텍스트 기반

| 형식 | 특징 |
| --- | --- |
| **JSON** | 웹 표준, 자기 설명, 사람 읽기 OK |
| **XML** | 무거움, schema 풍부 (XSD) |
| **YAML** | JSON 슈퍼셋, 사람 친화 |
| **TOML** | 설정 파일 |
| **CSV** | 단순, 표 데이터 |

### 4.2 바이너리 기반

| 형식 | 특징 |
| --- | --- |
| **Protocol Buffers** (Google) | 빠름, 작음, schema |
| **MessagePack** | "binary JSON" |
| **Apache Avro** | Hadoop, schema evolution |
| **Apache Thrift** (Facebook) | RPC + 직렬화 |
| **FlatBuffers** (Google) | Zero-copy |
| **Cap'n Proto** | FlatBuffers 의 영감 |
| **BSON** | MongoDB |
| **ASN.1** | LDAP, X.509, SNMP |

### 4.3 비교

| 형식 | 크기 | 속도 | 사람 읽기 |
| --- | --- | --- | --- |
| JSON | 큼 | 보통 | ✅ |
| XML | 가장 큼 | 느림 | △ |
| Protobuf | 작음 | 빠름 | ❌ |
| MessagePack | 작음 | 빠름 | ❌ |
| FlatBuffers | 작음 | 매우 빠름 (zero-copy) | ❌ |

---

## 5. MIME (Multipurpose Internet Mail Extensions)

### 5.1 개요
- 1992 RFC 1521
- 메일 첨부 / HTTP Content-Type 의 기반
- 임의 binary 데이터를 텍스트 (Base64) 또는 binary 로

### 5.2 형식

```
Content-Type: text/html; charset=UTF-8
Content-Transfer-Encoding: 7bit | 8bit | base64 | quoted-printable
```

### 5.3 주요 타입

| Type | 예 |
| --- | --- |
| `text/` | text/plain, text/html, text/css |
| `image/` | image/jpeg, image/png, image/webp |
| `audio/` | audio/mpeg, audio/wav |
| `video/` | video/mp4, video/webm |
| `application/` | application/json, application/pdf, application/octet-stream |
| `multipart/` | multipart/form-data, multipart/mixed |

### 5.4 Base64

```
binary 3 byte → text 4 byte
A-Z, a-z, 0-9, +, / (총 64), = padding

원본:  Man  (3 byte)
2진수: 01001101 01100001 01101110
6 bit씩: 010011 010110 000101 101110
값: 19 22 5 46
문자: T  W  F  u
결과: "TWFu"
```

오버헤드: 33%. 메일 / data URL / JWT.

---

## 6. 압축 (Compression)

### 6.1 무손실 (Lossless)

| 알고리즘 | 사용 |
| --- | --- |
| **gzip** (DEFLATE) | HTTP `Content-Encoding`, .tar.gz |
| **DEFLATE** | gzip 의 알고리즘, ZIP |
| **Brotli** (Google) | HTTP — gzip 보다 20% 작음 |
| **Zstandard (zstd)** | Facebook, 빠른 + 좋은 압축 |
| **LZ4** | 매우 빠름, 압축 ↓ |
| **LZMA / XZ** | 고압축, 느림 |
| **Snappy** (Google) | RocksDB, 빠름 |
| **bzip2** | gzip 보다 작음, 느림 |

### 6.2 손실 (Lossy)

| 알고리즘 | 데이터 |
| --- | --- |
| **JPEG** | 이미지 |
| **WebP** | 이미지 (Google) |
| **AVIF** | 이미지 (AV1 기반) |
| **HEIC** | 이미지 (Apple) |
| **MP3** | 오디오 |
| **AAC** | 오디오 |
| **Opus** | 오디오 (low latency) |
| **H.264 (AVC)** | 비디오 |
| **H.265 (HEVC)** | 비디오 |
| **AV1** | 비디오 (Open) |
| **VP9** | 비디오 (Google) |

### 6.3 HTTP 압축

```
Request:
  Accept-Encoding: gzip, br, zstd

Response:
  Content-Encoding: br
  ...
```

---

## 7. 암호화 (L6 측면)

엄격히는 L6 의 역할:
- 송신 시 암호화, 수신 시 복호화
- 응용 코드는 평문으로 작업

### 7.1 TLS (Transport Layer Security)
- L5/L6 의 경계 — 학문적
- TCP 위, 응용 아래
- 자세히 → [[../../tls-ssl/tls-ssl]]

### 7.2 응용 레벨 암호화
- AES-GCM, ChaCha20-Poly1305
- 자세히 → [[../../../security-theory/security-theory]]

---

## 8. Endianness (바이트 순서)

### 8.1 Big-endian vs Little-endian

```
32-bit 정수 0x12345678 메모리 저장:

Big-endian (네트워크 표준):
addr+0: 12
addr+1: 34
addr+2: 56
addr+3: 78

Little-endian (x86, ARM):
addr+0: 78
addr+1: 56
addr+2: 34
addr+3: 12
```

### 8.2 Network Byte Order
- **Big-endian** — RFC 1700
- IP 헤더, TCP 헤더 모두 big-endian
- C 함수: `htonl()`, `htons()`, `ntohl()`, `ntohs()`

### 8.3 함정
- 다른 endianness 호스트 간 통신 시 변환 필수
- Protobuf, JSON 등은 자동 (텍스트 또는 명시적)

---

## 9. 함정

### 함정 1 — 인코딩 가정
한글 파일 UTF-8 ↔ EUC-KR 혼동 → 깨진 글자. 명시적 인코딩 선언.

### 함정 2 — UTF-16 의 surrogate pair
😀 같은 이모지가 length 2 — JavaScript 함정.

### 함정 3 — BOM (Byte Order Mark)
UTF-8 의 BOM (0xEF 0xBB 0xBF) — 일부 파서가 안 처리. 보통 없는 게 좋음.

### 함정 4 — Base64 의 줄바꿈
RFC 일부는 76 자마다 줄바꿈, 일부는 안 함. 호환성 확인.

### 함정 5 — JSON 의 정밀도
JavaScript `Number` 는 IEEE 754 → BigInt 손실. 큰 ID 는 string 으로.

### 함정 6 — Endianness 가정
"이 시스템은 항상 little-endian" — ARM 도 양쪽 모드 지원. 명시적 변환.

### 함정 7 — 이중 압축
이미 압축된 데이터 (JPEG, gzip) 를 또 gzip → 크기 증가, CPU 낭비.

---

## 10. 학습 자료

- **Unicode Standard** 14.0
- RFC 3629 (UTF-8), RFC 1521 (MIME)
- **Programming with Unicode** (Victor Stinner)
- **High Performance Browser Networking** Ch. 13 (Compression)

---

## 11. 관련

- [[layer-5-session]] — 하위
- [[../layer-7-application/layer-7-application]] — 상위
- [[../../tls-ssl/tls-ssl]] — 암호화
- [[../../../data-structure/arrays-and-strings/arrays-and-strings]] — UTF-8 깊이
- [[../osi-7-layer]] — OSI hub
