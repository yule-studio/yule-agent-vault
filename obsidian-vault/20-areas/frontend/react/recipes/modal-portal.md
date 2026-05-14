---
title: "Modal with React Portal — 정석 구현"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [frontend, react, recipes, modal, portal, a11y]
---

# Modal with React Portal

**[[recipes|↑ Recipes Hub]]**

> Modal / Dialog 의 표준 구현. **Portal + state + a11y**.

## 1. Portal — 다른 DOM tree 에 렌더

```tsx
import { createPortal } from 'react-dom';

createPortal(
  <div>이건 다른 곳에 그려짐</div>,
  document.getElementById('modal-root')!
);
```

→ 컴포넌트 tree 안에 있지만 DOM 의 다른 위치에. modal / tooltip / dropdown 의 표준.

```html
<!-- index.html -->
<body>
  <div id="root"></div>
  <div id="modal-root"></div>     <!-- 추가 -->
</body>
```

## 2. 왜 Portal 인가?

```tsx
<header style={{ overflow: 'hidden' }}>
  <Modal />     {/* ❌ overflow 잘림, z-index 영향 받음 */}
</header>
```

→ Modal 이 부모의 CSS 영향 받지 않게.

## 3. 기본 Modal

```tsx
// Modal.tsx
import { createPortal } from 'react-dom';
import { useEffect } from 'react';

interface Props {
  open: boolean;
  onClose: () => void;
  children: React.ReactNode;
}

export function Modal({ open, onClose, children }: Props) {
  useEffect(() => {
    if (!open) return;
    const handleEsc = (e: KeyboardEvent) => {
      if (e.key === 'Escape') onClose();
    };
    window.addEventListener('keydown', handleEsc);
    return () => window.removeEventListener('keydown', handleEsc);
  }, [open, onClose]);

  useEffect(() => {
    if (open) document.body.style.overflow = 'hidden';
    return () => { document.body.style.overflow = ''; };
  }, [open]);

  if (!open) return null;

  return createPortal(
    <div
      role="dialog"
      aria-modal="true"
      onClick={onClose}
      style={{
        position: 'fixed', inset: 0,
        background: 'rgba(0,0,0,0.5)',
        display: 'flex', alignItems: 'center', justifyContent: 'center',
        zIndex: 1000,
      }}
    >
      <div
        onClick={e => e.stopPropagation()}
        style={{
          background: 'white',
          borderRadius: 8,
          padding: 24,
          minWidth: 320,
          maxWidth: '90vw',
          maxHeight: '90vh',
          overflow: 'auto',
        }}
      >
        {children}
      </div>
    </div>,
    document.getElementById('modal-root')!
  );
}
```

### 사용
```tsx
function Page() {
  const [open, setOpen] = useState(false);
  return (
    <>
      <button onClick={() => setOpen(true)}>열기</button>
      <Modal open={open} onClose={() => setOpen(false)}>
        <h2>제목</h2>
        <p>내용</p>
        <button onClick={() => setOpen(false)}>닫기</button>
      </Modal>
    </>
  );
}
```

## 4. 핵심 기능

| | 설명 |
| --- | --- |
| **Esc 키 닫기** | `keydown` 이벤트 + `onClose()` |
| **Backdrop 클릭 닫기** | 바깥 `onClick={onClose}` |
| **본문 클릭 닫힘 방지** | 안쪽 `stopPropagation` |
| **body scroll lock** | `overflow: hidden` 토글 |
| **`role="dialog"` + `aria-modal`** | 스크린리더 |
| **z-index 1000+** | 다른 요소 위 |

## 5. focus trap — 접근성

```tsx
import { useEffect, useRef } from 'react';

useEffect(() => {
  if (!open || !ref.current) return;

  const focusable = ref.current.querySelectorAll<HTMLElement>(
    'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
  );
  const first = focusable[0];
  const last = focusable[focusable.length - 1];

  first?.focus();

  const trap = (e: KeyboardEvent) => {
    if (e.key !== 'Tab') return;
    if (e.shiftKey ? document.activeElement === first : document.activeElement === last) {
      e.preventDefault();
      (e.shiftKey ? last : first)?.focus();
    }
  };

  window.addEventListener('keydown', trap);
  return () => window.removeEventListener('keydown', trap);
}, [open]);
```

→ Tab 키 가 modal 안에서만 순환.

## 6. 열렸을 때 trigger element 로 focus 복귀

```tsx
const previousFocus = useRef<HTMLElement | null>(null);

useEffect(() => {
  if (open) {
    previousFocus.current = document.activeElement as HTMLElement;
  } else {
    previousFocus.current?.focus();   // 닫을 때 원래 element 로
  }
}, [open]);
```

## 7. 라이브러리로 — radix-ui Dialog

```bash
yarn add @radix-ui/react-dialog
```

```tsx
import * as Dialog from '@radix-ui/react-dialog';

<Dialog.Root>
  <Dialog.Trigger>열기</Dialog.Trigger>
  <Dialog.Portal>
    <Dialog.Overlay className="bg-black/50 fixed inset-0" />
    <Dialog.Content className="bg-white rounded p-6 ...">
      <Dialog.Title>제목</Dialog.Title>
      <Dialog.Description>내용</Dialog.Description>
      <Dialog.Close>닫기</Dialog.Close>
    </Dialog.Content>
  </Dialog.Portal>
</Dialog.Root>
```

→ a11y / focus trap / esc / scroll lock 자동.

## 8. 전역 modal manager — Promise 기반

```tsx
// useModal store (zustand)
interface ModalState {
  modals: Array<{ id: string; component: React.ReactNode; resolve: (v: any) => void }>;
  open: (component: React.ReactNode) => Promise<any>;
  close: (id: string, value?: any) => void;
}

const useModalStore = create<ModalState>((set, get) => ({
  modals: [],
  open: (component) => new Promise((resolve) => {
    const id = crypto.randomUUID();
    set(s => ({ modals: [...s.modals, { id, component, resolve }] }));
  }),
  close: (id, value) => {
    const modal = get().modals.find(m => m.id === id);
    modal?.resolve(value);
    set(s => ({ modals: s.modals.filter(m => m.id !== id) }));
  },
}));

// ModalRoot — 어딘가 한 곳에서
function ModalRoot() {
  const modals = useModalStore(s => s.modals);
  return (
    <>{modals.map(m => <Modal key={m.id} open={true} onClose={() => useModalStore.getState().close(m.id)}>{m.component}</Modal>)}</>
  );
}

// 사용
const result = await useModalStore.getState().open(<ConfirmDialog message="삭제?" />);
if (result === true) { /* ... */ }
```

→ Promise 로 modal 결과 await. 복잡하지만 강력.

## 9. confirm dialog 패턴

```tsx
// 사용
const ok = await confirm({ message: '정말 삭제?', confirmText: '삭제' });
if (ok) doDelete();
```

→ 위 store + ConfirmDialog 컴포넌트.

## 10. answer-fe 의 흔한 패턴

```tsx
// hooks/useModal.ts
export function useModal() {
  const [open, setOpen] = useState(false);
  return {
    open,
    show: () => setOpen(true),
    hide: () => setOpen(false),
    toggle: () => setOpen(o => !o),
  };
}

// 사용
const { open, show, hide } = useModal();
<button onClick={show}>열기</button>
<Modal open={open} onClose={hide}>...</Modal>
```

## 11. animation — exit animation

```tsx
// 닫기 시 즉시 unmount → 애니메이션 못 봄
// 해결: framer-motion 의 AnimatePresence

import { AnimatePresence, motion } from 'framer-motion';

<AnimatePresence>
  {open && (
    <motion.div
      initial={{ opacity: 0 }}
      animate={{ opacity: 1 }}
      exit={{ opacity: 0 }}
    >
      ...
    </motion.div>
  )}
</AnimatePresence>
```

## 12. 함정

1. **`modal-root` div 없음** — `null` 에 portal → 에러. index.html 확인.
2. **body scroll lock 누락** — 모달 뒤 페이지 스크롤됨.
3. **Esc 키 미지원** — 사용성 떨어짐.
4. **focus trap 없음** — Tab 으로 모달 밖 요소 focus.
5. **a11y 누락** — role / aria. 스크린리더 사용자.
6. **multiple modal stacking** — z-index 충돌. modal manager 또는 라이브러리.
7. **closing animation 누락** — `open=false` → 즉시 unmount. framer-motion 등.

## 13. 외부 자료

- [react.dev — createPortal](https://react.dev/reference/react-dom/createPortal)
- [Radix UI Dialog](https://www.radix-ui.com/primitives/docs/components/dialog)
- [WAI-ARIA Dialog](https://www.w3.org/WAI/ARIA/apg/patterns/dialog-modal/)

## 14. 관련

- [[recipes]]
- [[../core-concepts/hooks-essentials]]
