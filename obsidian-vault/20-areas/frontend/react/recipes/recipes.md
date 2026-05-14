---
title: "Recipes — 실전 패턴 모음"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [frontend, react, recipes, patterns, hub]
---

# Recipes

**[[../react|↑ React Hub]]**

> 자주 만나는 UI 패턴의 정석 구현.

## 1. 목록

| 패턴 | 노트 |
| --- | --- |
| Modal (Portal) | [[modal-portal]] |
| Infinite Scroll | [[infinite-scroll]] |

추가 예정 (수요에 따라):
- Debounced search
- Click outside (drawer / dropdown)
- Drag & Drop
- File upload + preview
- Skeleton UI
- Tabs
- Pagination
- Toast / Notification system
- Form wizard (multi-step)
- Print / PDF (react-pdf, html2canvas)

## 2. 흔한 UI 의 라이브러리 추천

| 패턴 | 직접 구현 | 라이브러리 |
| --- | --- | --- |
| Modal | Portal + state | radix-ui / headlessui |
| Dropdown | Portal + outside click | radix-ui |
| Toast | Portal + queue | react-hot-toast, sonner |
| Date picker | (복잡) | react-datepicker, MUI/antd 내장 |
| Drag&Drop | (복잡) | @dnd-kit/core, react-beautiful-dnd |
| Carousel | (복잡) | swiper, react-slick |
| Tooltip | Portal + position | radix-ui, react-tooltip |
| Tabs | state | radix-ui Tabs |
| Accordion | state | radix-ui Accordion |
| Charts | (복잡) | recharts, victory, nivo |

→ "복잡" 표시는 직접 구현 비추.

## 3. 직접 구현 vs 라이브러리

### 직접 구현
- 디자인 100% 컨트롤.
- bundle 절약.
- 학습 효과.

### 라이브러리
- 빠름.
- 접근성 (a11y) / 키보드 / 모바일 검증됨.
- bug 검증됨.

→ 비핵심 = 라이브러리. 핵심 / 차별화 = 직접.

## 4. answer-fe / job-answer-fe 의 흔한 lib

```bash
cat ~/masterway-dev/answer-fe/package.json | grep -E "dnd|swiper|datepicker|toast|tooltip"
cat ~/masterway-dev/job-answer-fe/package.json | grep -E "dnd|swiper|datepicker|toast|tooltip"
```

자주 보이는:
- swiper / react-slick — carousel.
- react-pdf — PDF 표시.
- html2canvas / html-to-image — 화면 캡처.
- react-cookie — cookie 관리.
- sendbird — 채팅.
- daily-co / zoom-websdk — 영상통화.
- dompurify — XSS 방어.

## 5. 학습 순서

1. **[[modal-portal]]** — Portal 의 가장 흔한 사용.
2. **[[infinite-scroll]]** — react-query infinite + IntersectionObserver.
3. 위 라이브러리들 1 개씩 도입 경험.

## 6. 관련

- [[modal-portal]]
- [[infinite-scroll]]
- [[../performance/virtual-list]]
- [[../server-state/react-query]]
