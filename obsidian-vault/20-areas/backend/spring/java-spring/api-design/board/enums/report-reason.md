---
title: "ReportReason — 8 신고 사유"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T13:14:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - board
  - enums
  - report
---

# ReportReason — 8 신고 사유

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[enums|↑ enums hub]]**

---

## 1. 코드

```java
public enum ReportReason {
    SPAM,              // 광고 / 도배
    INAPPROPRIATE,     // 부적절 / 음란
    HARASSMENT,        // 괴롭힘 / 욕설
    HATE_SPEECH,       // 혐오 발언
    VIOLENCE,          // 폭력 위협
    COPYRIGHT,         // 저작권 침해
    PERSONAL_INFO,     // 개인정보 노출
    OTHER;             // 기타 (text 입력)

    public String label() {
        return switch (this) {
            case SPAM -> "광고/도배";
            case INAPPROPRIATE -> "부적절/음란";
            case HARASSMENT -> "괴롭힘/욕설";
            case HATE_SPEECH -> "혐오 발언";
            case VIOLENCE -> "폭력 위협";
            case COPYRIGHT -> "저작권 침해";
            case PERSONAL_INFO -> "개인정보 노출";
            case OTHER -> "기타";
        };
    }
}
```

---

## 2. 왜 enum (자유 text X)

- 통계 / 자동 처리 가능 ("spam 신고가 폭증").
- admin 의 처리 정책 분기 (저작권 = 법적 절차).
- UI 선택 — 일관성.

OTHER 만 추가 `detail` 컬럼 (자유 text).

---

## 3. 신고 정책

| 사유 | 자동 hide threshold | admin 우선순위 |
| --- | --- | --- |
| SPAM | 3회 (낮음) | 빠른 처리 |
| INAPPROPRIATE | 5회 | 일반 |
| HARASSMENT | 5회 | 빠른 처리 |
| HATE_SPEECH | 3회 | 빠른 처리 |
| VIOLENCE | 2회 (낮음) | 즉시 |
| COPYRIGHT | 1회 (즉시 hide) | 법적 |
| PERSONAL_INFO | 1회 (즉시 hide) | 법적 |
| OTHER | 5회 | 일반 |

---

## 4. 함정

### 함정 1 — 자유 text 사유
통계 X, 자동 처리 X.
→ enum.

### 함정 2 — 모든 사유의 threshold 동일
COPYRIGHT 가 5회까지 노출 — 법적 위험.
→ 사유별 threshold.

### 함정 3 — OTHER 의 detail 누락
"기타" 만으로 admin 판단 X.
→ detail TEXT 컬럼 필수 (OTHER 시).

---

## 5. 관련

- [[enums|↑ hub]]
- [[../database/reports-table]]
- [[../design-decisions/moderation-policy]]
