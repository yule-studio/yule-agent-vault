---
title: "보안 이론 (Security Theory)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T13:00:00+09:00
tags:
  - security
  - cryptography
  - authentication
  - oauth
  - tls
---

# 보안 이론 (Security Theory)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 사전 수준 |

**[[../computer-science|↑ computer-science]]**

---

## 1. 한 줄 정의

**자원 (데이터 / 시스템) 을 인가되지 않은 접근 / 변경 / 거부로부터 보호** 하는
이론과 실무. CIA 삼각형이 핵심.

---

## 2. CIA Triad

- **Confidentiality (기밀성)** — 인가된 사람만 접근
- **Integrity (무결성)** — 변조 감지
- **Availability (가용성)** — 필요 시 사용 가능

추가:
- **Authenticity** — 출처 진위
- **Non-repudiation** — 부인 방지

---

## 3. 위협 모델 / STRIDE

Microsoft 의 STRIDE 분류:

| 위협 | 의미 | 대응 |
| --- | --- | --- |
| **S** poofing | 신원 위조 | 인증 |
| **T** ampering | 변조 | 무결성, 해시 |
| **R** epudiation | 부인 | 감사 로그, 서명 |
| **I** nformation Disclosure | 정보 유출 | 암호화 |
| **D** enial of Service | 서비스 거부 | Rate limit, redundancy |
| **E** levation of Privilege | 권한 상승 | 최소 권한 |

---

## 4. 암호학 기본

### 4.1 대칭 키 암호화

- 같은 키로 암호화 / 복호화
- 빠름 — 대용량 데이터
- **AES** (Advanced Encryption Standard, 2001) — 128/192/256 비트
- 운영 모드: **GCM** (인증 포함, 권장), CBC (deprecated), ECB (절대 X)

### 4.2 비대칭 키 (공개키)

- 공개키 (암호화) / 개인키 (복호화)
- 키 교환 / 서명에 사용
- **RSA** (1977) — 2048 비트 이상
- **ECC** (Elliptic Curve) — 짧은 키로 동등 보안 (P-256)
- **Ed25519** — 서명, 빠름

### 4.3 해시 함수

| 특성 | 의미 |
| --- | --- |
| Pre-image resistance | h(x) → x 찾기 어려움 |
| Second pre-image | x → x' 동일 해시 어려움 |
| Collision resistance | 임의 충돌 어려움 |

| 알고리즘 | 출력 | 상태 |
| --- | --- | --- |
| MD5 | 128 bit | ❌ 깨짐 |
| SHA-1 | 160 bit | ❌ Google 2017 충돌 |
| SHA-256 | 256 bit | ✅ |
| SHA-3 | 가변 | ✅ |
| BLAKE3 | 256 bit | ✅ 빠름 |

### 4.4 비밀번호 해싱

일반 해시 X (GPU 로 빠름). 느린 해시 사용:

- **bcrypt** (1999) — cost factor
- **scrypt** (2009) — 메모리 hard
- **Argon2** (2015) — PHC 우승, 권장
- **PBKDF2** — 표준이지만 GPU 약함

### 4.5 HMAC

- Hash + 비밀 키
- 메시지 인증 + 무결성
- `HMAC-SHA256` 이 표준

### 4.6 디지털 서명

- 개인키로 서명, 공개키로 검증
- **RSA-PSS, ECDSA, Ed25519**

---

## 5. TLS / HTTPS

[[../network/network#7 TLS / HTTPS]] 참조.

### 5.1 TLS 1.3 (2018)

- 1-RTT handshake (이전 2-RTT)
- 0-RTT 재방문
- 약한 알고리즘 제거 (RC4, 3DES, SHA-1)
- Forward Secrecy 강제

### 5.2 인증서 검증

- CA 체인
- Hostname 확인
- 유효 기간
- 폐기 (CRL / OCSP)
- Certificate Transparency (CT)

### 5.3 mTLS (Mutual TLS)

- 서버 + 클라이언트 모두 인증서
- Zero Trust 의 기반

---

## 6. 인증 (Authentication)

### 6.1 인증 요소

1. **Knowledge** — 비밀번호
2. **Possession** — 토큰, 폰, USB 키
3. **Inherence** — 생체

**MFA** (Multi-Factor) — 2 이상 결합.

### 6.2 비밀번호 정책

- 길이 > 복잡도 (8 자 복잡 < 16 자 단순)
- HIBP (Have I Been Pwned) 체크
- 주기적 변경 강제 X (NIST 2017 권고)
- bcrypt/Argon2 로 해싱

### 6.3 OTP

- **HOTP** (RFC 4226) — Counter
- **TOTP** (RFC 6238) — Time-based, Google Authenticator

### 6.4 WebAuthn / Passkey

- FIDO2 표준
- 공개키 기반, 피싱 면역
- Apple/Google/Microsoft 모두 지원

### 6.5 SSO

- **SAML** — XML, 엔터프라이즈
- **OAuth 2.0 / OIDC** — 웹 표준
- **Kerberos** — 기업 내부

---

## 7. 인가 (Authorization)

### 7.1 모델

- **DAC** (Discretionary) — 소유자가 결정 (UNIX 파일)
- **MAC** (Mandatory) — OS 강제 (SELinux)
- **RBAC** (Role-Based) — 역할 기반
- **ABAC** (Attribute-Based) — 속성 (시간, 위치)
- **PBAC** (Policy-Based) — 정책 엔진 (OPA)
- **ReBAC** (Relationship-Based) — Zanzibar / Google Drive

### 7.2 OAuth 2.0

```
Resource Owner → Authorization Server → Access Token → Resource Server
```

#### 주요 Grant Type

| 타입 | 용도 |
| --- | --- |
| **Authorization Code** | 웹 앱 (PKCE 필수) |
| **PKCE** | 모바일 / SPA |
| **Client Credentials** | 서비스 간 (M2M) |
| **Refresh Token** | Access token 갱신 |
| ~~Implicit~~ | deprecated |
| ~~Password~~ | deprecated |
| **Device Code** | TV / IoT |

### 7.3 OIDC (OpenID Connect)

- OAuth 2.0 위의 인증 레이어
- **ID Token** (JWT)
- userinfo 엔드포인트

### 7.4 JWT (JSON Web Token)

```
header.payload.signature
{alg:HS256, typ:JWT}.{sub:..., exp:...}.HMAC(...)
```

**주의**:
- `alg: none` 취약점 — 라이브러리 검증
- 만료 짧게 + Refresh Token
- 민감 정보 payload 에 X (base64 = 평문)
- Symmetric (HS256) vs Asymmetric (RS256)

### 7.5 세션 vs 토큰

| 측면 | 세션 | 토큰 (JWT) |
| --- | --- | --- |
| 저장 | 서버 | 클라이언트 |
| 무효화 | 즉시 | 만료 까지 |
| 확장 | 공유 저장소 | Stateless |
| 보안 | 쿠키 (CSRF) | XSS |
| 용도 | 웹 | API / 모바일 |

---

## 8. 웹 보안 (OWASP Top 10, 2021)

| 순위 | 위협 |
| --- | --- |
| 1 | Broken Access Control |
| 2 | Cryptographic Failures |
| 3 | Injection (SQL, XSS, Command) |
| 4 | Insecure Design |
| 5 | Security Misconfiguration |
| 6 | Vulnerable & Outdated Components |
| 7 | Identification & Authentication Failures |
| 8 | Software & Data Integrity Failures |
| 9 | Security Logging & Monitoring Failures |
| 10 | SSRF (Server-Side Request Forgery) |

### 8.1 SQL Injection

```python
# Bad
cursor.execute(f"SELECT * FROM users WHERE id = {user_id}")

# Good
cursor.execute("SELECT * FROM users WHERE id = ?", (user_id,))
```

### 8.2 XSS (Cross-Site Scripting)

- **Reflected** — URL 의 페이로드
- **Stored** — DB 에 저장된 페이로드
- **DOM-based** — JS 가 URL 을 DOM 에 삽입

방어: 출력 인코딩 (HTML escape), CSP, HttpOnly cookie.

#### Content Security Policy
```
Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-abc'
```

### 8.3 CSRF (Cross-Site Request Forgery)

- 사용자 인증된 상태에서 다른 사이트가 요청 위조
- 방어:
  - **SameSite cookie** (`Strict` / `Lax`)
  - **CSRF Token** (Synchronizer pattern)
  - **Origin / Referer 검증**

### 8.4 SSRF

- 서버가 임의 URL 요청 → 내부망 / 메타데이터 (AWS IMDS) 노출
- 방어: URL allowlist, 메타데이터 차단

### 8.5 Path Traversal

```python
# Bad
open(f"/uploads/{filename}", "rb")    # ../../../etc/passwd

# Good
import os
safe = os.path.basename(filename)
open(f"/uploads/{safe}", "rb")
```

### 8.6 Open Redirect

```
https://app.com/login?next=https://evil.com
```

`next` 검증 — 자신의 도메인만.

---

## 9. 시스템 / 네트워크 보안

### 9.1 방화벽

- **Stateless** — 패킷 단위 (iptables)
- **Stateful** — 연결 추적
- **WAF** (Web Application Firewall) — L7

### 9.2 IDS / IPS

- **IDS** — Intrusion Detection (감지)
- **IPS** — Intrusion Prevention (차단)
- Snort, Suricata

### 9.3 Zero Trust

- "Never trust, always verify"
- 네트워크 위치 기반 신뢰 X
- 매 요청 인증 + 인가
- mTLS, ZTNA, BeyondCorp (Google)

### 9.4 VPN

- IPsec — L3
- OpenVPN, WireGuard — TLS / UDP
- Zero Trust 가 대체 중

---

## 10. 비밀 관리 (Secrets Management)

### 10.1 절대 금지

- 코드에 하드코딩
- Git 에 커밋 (.env)
- 평문 환경변수 (production)
- 로그에 출력

### 10.2 권장

- **Vault** (HashiCorp), AWS Secrets Manager, GCP Secret Manager
- **KMS** (Key Management Service)
- **Envelope encryption** — Master key + Data key
- **Rotation** — 정기 교체
- **Audit log**

### 10.3 Git 에 실수로 커밋한 경우

```bash
# 즉시: 비밀 무효화 (rotate)
# 그 후: git filter-repo / BFG Repo-Cleaner
# 단순 reset 으론 GitHub 캐시에 남음
```

---

## 11. 컨테이너 / 클라우드 보안

### 11.1 컨테이너

- 최소 베이스 이미지 (alpine, distroless)
- non-root 사용자
- read-only filesystem
- `--cap-drop=ALL --cap-add=NET_BIND_SERVICE` 등
- 이미지 스캔 (Trivy, Snyk)
- 서명 (Cosign)

### 11.2 K8s

- RBAC
- NetworkPolicy
- Pod Security Standards
- Service Mesh (Istio + mTLS)

### 11.3 클라우드 IAM

- 최소 권한
- 서비스 계정 vs 사용자 계정
- 자격 증명 없는 (workload identity)
- IMDSv2 (AWS) — SSRF 방어

---

## 12. 함정 / 안티패턴

### 함정 1 — Roll-your-own crypto
표준 라이브러리 사용. 직접 구현 X.

### 함정 2 — 평문 비밀번호 저장
bcrypt/Argon2 필수.

### 함정 3 — MD5 / SHA-1
무결성 / 비밀번호 모두 부적합.

### 함정 4 — JWT alg=none
명시적 알고리즘 검증.

### 함정 5 — Open Redirect 무시
`next` 파라미터 호스트 검증.

### 함정 6 — 로그에 비밀 출력
Authorization 헤더, password, token 마스킹.

### 함정 7 — CORS 의 wildcard + credentials
보안 무력화.

### 함정 8 — Timing attack
비밀번호 / 토큰 비교는 constant-time (`hmac.compare_digest`).

### 함정 9 — SQL 의 ORDER BY 사용자 입력
파라미터 바인딩 X — allowlist 사용.

### 함정 10 — XML 파서의 XXE
`disable_entity_loading=True`.

---

## 13. 보안 도구

| 종류 | 도구 |
| --- | --- |
| SAST | SonarQube, Semgrep |
| DAST | OWASP ZAP, Burp |
| SCA | Snyk, Dependabot |
| Container Scan | Trivy, Grype |
| Secret Scan | gitleaks, trufflehog |
| Pentest | nmap, metasploit |
| Crypto | OpenSSL, GnuPG |
| Cert | Let's Encrypt, certbot |

---

## 14. 면접 질문

1. **CIA Triad**.
2. **대칭 vs 비대칭 암호화**.
3. **bcrypt 가 SHA-256 보다 나은 이유**.
4. **HTTPS 의 동작**.
5. **JWT vs Session**.
6. **OAuth 2.0 의 Authorization Code flow**.
7. **XSS / CSRF / SQL Injection 차이와 방어**.
8. **CORS vs CSRF**.
9. **Zero Trust**.
10. **OWASP Top 10 5 개 이상**.

---

## 15. 학습 자료

- **Cryptography Engineering** (Ferguson / Schneier / Kohno)
- **The Web Application Hacker's Handbook** (Stuttard / Pinto)
- **Security Engineering** (Ross Anderson) — 무료 공개
- **OWASP** — owasp.org Top 10, Cheat Sheets
- **HackerOne / Bugcrowd** 보고서
- **PortSwigger Web Security Academy** — 무료 실습
- [HackTricks](https://book.hacktricks.xyz/)

---

## 16. 관련

- [[../network/network]] — TLS, HTTPS, DNS
- [[../operating-system/operating-system]] — 권한, 격리, 컨테이너
- [[../distributed-systems/distributed-systems]] — mTLS, Zero Trust
- [[../../security/security|↑ security (실무)]]
- [[../computer-science|↑ computer-science]]
