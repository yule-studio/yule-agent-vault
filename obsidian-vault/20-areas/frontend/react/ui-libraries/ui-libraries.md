---
title: "UI Libraries — MUI / antd / Chakra / shadcn"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [frontend, react, ui-library, mui, antd, hub]
---

# UI Libraries

**[[../react|↑ React Hub]]**

> 완성된 컴포넌트 모음. **`masterway-dev` 매핑**:
> - `answer-fe` = MUI (Material UI).
> - `job-answer-fe` = antd (Ant Design).

## 1. 선택지

| | 디자인 | 한국 |
| --- | --- | --- |
| **MUI (Material UI)** | Google Material | 가장 흔함 |
| **Ant Design (antd)** | 중국 / B2B 풍 | 흔함 |
| **Chakra UI** | 깔끔 / 접근성 강조 | 점차 증가 |
| **Mantine** | 최신 / 풍부 | 증가 |
| **shadcn/ui** | Tailwind 기반 / 코드 복사 | 트렌드 |
| **Radix UI** | headless (스타일 없음) | shadcn 의 기반 |
| **HeadlessUI** | tailwind 팀 headless | Tailwind 와 자연 |
| **Naver SDS / 토스 디자인** | 사내 | 사내 |

→ 학습 우선: **자기 프로젝트 사용 라이브러리**.

## 2. headless vs styled

- **styled (MUI, antd, Chakra)**: 디자인 + 동작 둘 다 제공. 빠르고 일관성. 디자인 커스텀 비용.
- **headless (Radix, HeadlessUI)**: 동작 (포커스, 키보드, ARIA) 만 제공, 스타일은 자유. 디자인 자유, 코드 양 많음.

→ shadcn/ui = Radix + Tailwind 로 양 끝의 절충.

## 3. 도입 시 고려

1. **bundle size** — MUI / antd 는 큼 (수백 KB). tree-shaking + lazy import.
2. **디자인 커스텀** — theme 시스템 vs 직접 변경.
3. **i18n 한국어** — DatePicker / Pagination 라벨.
4. **접근성** — Radix / Chakra 가 강함.
5. **타입스크립트 지원** — 거의 모두 OK.

## 4. 학습 우선순위

1. **[[mui-material]]** — answer-fe 쓰면.
2. **[[antd]]** — job-answer-fe 쓰면.
3. **theme 커스터마이즈** — 디자인 시스템 적용.
4. **lazy import** — 번들 최적화.

## 5. masterway-dev 매핑 — 빠른 확인

```bash
grep -E "@mui|antd" ~/masterway-dev/answer-fe/package.json
grep -E "@mui|antd" ~/masterway-dev/job-answer-fe/package.json
```

`answer-fe`:
```json
"@mui/material": "^5.x",
"@mui/icons-material": "^5.x",
"@emotion/react": "^11.x",
"@emotion/styled": "^11.x",
```

`job-answer-fe`:
```json
"antd": "^5.x",
"@ant-design/icons": "...",
```

## 6. 라이브러리 vs 자체 컴포넌트 — 언제 어떤?

- **자주 쓰는 기본 (Button, Input, Modal, Toast)** → 라이브러리.
- **도메인 특화 (QuestionCard, AnswerForm)** → 자체.
- **디자인 다른 (브랜드 톤)** → 라이브러리 컴포넌트 wrap.

→ 라이브러리 → wrapper → 자체 컴포넌트 의 3 단 구조 흔함.

## 7. 라이브러리 + Tailwind / styled-components 공존

```tsx
// MUI + styled-components
import { Button as MuiButton } from '@mui/material';
import styled from 'styled-components';

const Button = styled(MuiButton)`
  background: black;
  &:hover { background: #333; }
`;
```

```tsx
// antd + Tailwind
<Button className="px-6 bg-blue-500">Click</Button>
```

→ 우선순위 충돌 주의. `!important` 또는 `& {}` selector.

## 8. shadcn/ui — 최근 흐름

- 라이브러리가 아닌 **코드 복사 방식**.
- Radix (동작) + Tailwind (스타일) 의 컴포넌트.
- 내 프로젝트로 들여와 자유 수정.

```bash
npx shadcn-ui@latest init
npx shadcn-ui@latest add button
```

→ `components/ui/button.tsx` 가 생성됨. 내가 소유 + 수정.

## 9. 함정

1. **번들 크기 폭주** — MUI / antd 의 모든 컴포넌트 import 금지. tree-shaking + named import.
2. **theme 충돌** — MUI 의 ThemeProvider 와 styled-components ThemeProvider 동시 → key 충돌.
3. **disabled / readonly** — 라이브러리마다 prop 이름 다름. 통일 wrapper.
4. **z-index 충돌** — modal / dropdown 의 z-index. 라이브러리 theme.
5. **a11y** — `aria-label` / `role` 직접 추가 잊지 말기.
6. **version upgrade** — major upgrade 시 prop / 동작 변경. lock 후 적용 시기 신중.

## 10. 외부 자료

- [MUI](https://mui.com/)
- [Ant Design](https://ant.design/)
- [shadcn/ui](https://ui.shadcn.com/)
- [Radix UI](https://www.radix-ui.com/)

## 11. 관련

- [[mui-material]]
- [[antd]]
- [[../styling/styling]]
