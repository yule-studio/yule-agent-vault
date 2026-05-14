---
title: "MUI (Material UI) — Google Material 디자인"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [frontend, react, ui-library, mui, material-ui]
---

# MUI (Material UI)

**[[ui-libraries|↑ UI Libraries Hub]]**

> `masterway-dev/answer-fe` 사용. v5 기준.

## 1. 설치

```bash
yarn add @mui/material @emotion/react @emotion/styled
yarn add @mui/icons-material           # 아이콘
yarn add @mui/lab                       # experimental (DatePicker 등 v5 시절)
yarn add @mui/x-date-pickers @mui/x-data-grid    # 별도 패키지
```

## 2. ThemeProvider + CssBaseline

```tsx
import { ThemeProvider, createTheme, CssBaseline } from '@mui/material';

const theme = createTheme({
  palette: {
    primary: { main: '#0064FF' },
    secondary: { main: '#FF6B00' },
    mode: 'light',          // 'dark' 가능
  },
  typography: {
    fontFamily: 'Pretendard, sans-serif',
  },
});

function App() {
  return (
    <ThemeProvider theme={theme}>
      <CssBaseline />        {/* 전역 reset */}
      <BrowserRouter>...</BrowserRouter>
    </ThemeProvider>
  );
}
```

## 3. 자주 쓰는 컴포넌트

### Button
```tsx
import { Button } from '@mui/material';

<Button variant="contained" color="primary" size="large">Click</Button>
<Button variant="outlined" disabled>Outlined</Button>
<Button variant="text" startIcon={<SaveIcon />}>Save</Button>
```

variant: `contained` | `outlined` | `text`.

### TextField
```tsx
import { TextField } from '@mui/material';

<TextField
  label="이메일"
  variant="outlined"
  value={email}
  onChange={e => setEmail(e.target.value)}
  error={!!emailError}
  helperText={emailError}
  fullWidth
/>
```

### Modal / Dialog
```tsx
import { Dialog, DialogTitle, DialogContent, DialogActions, Button } from '@mui/material';

<Dialog open={open} onClose={() => setOpen(false)}>
  <DialogTitle>제목</DialogTitle>
  <DialogContent>본문</DialogContent>
  <DialogActions>
    <Button onClick={() => setOpen(false)}>취소</Button>
    <Button onClick={onConfirm} variant="contained">확인</Button>
  </DialogActions>
</Dialog>
```

### Snackbar (toast)
```tsx
<Snackbar
  open={open}
  autoHideDuration={3000}
  onClose={() => setOpen(false)}
  anchorOrigin={{ vertical: 'bottom', horizontal: 'right' }}
>
  <Alert severity="success">저장 완료</Alert>
</Snackbar>
```

### Stack / Box — 레이아웃
```tsx
import { Box, Stack } from '@mui/material';

<Stack direction="row" spacing={2} alignItems="center">
  <Button>A</Button>
  <Button>B</Button>
</Stack>

<Box sx={{ p: 2, bgcolor: 'background.paper', borderRadius: 1 }}>...</Box>
```

`Box` 의 `sx` prop — inline 스타일 (theme 접근).

### Typography
```tsx
<Typography variant="h1">Heading 1</Typography>
<Typography variant="body1" color="text.secondary">본문</Typography>
```

### Grid (v5)
```tsx
<Grid container spacing={2}>
  <Grid item xs={12} md={6}>왼쪽</Grid>
  <Grid item xs={12} md={6}>오른쪽</Grid>
</Grid>
```

→ v7+ 부터 Grid2 권장 (`item` 안 씀).

### Autocomplete
```tsx
<Autocomplete
  options={['React', 'Vue', 'Svelte']}
  renderInput={(params) => <TextField {...params} label="프레임워크" />}
  onChange={(_, value) => setValue(value)}
/>
```

### DatePicker (v5)
```tsx
import { LocalizationProvider } from '@mui/x-date-pickers/LocalizationProvider';
import { AdapterDayjs } from '@mui/x-date-pickers/AdapterDayjs';
import { DatePicker } from '@mui/x-date-pickers';

<LocalizationProvider dateAdapter={AdapterDayjs}>
  <DatePicker value={date} onChange={setDate} />
</LocalizationProvider>
```

## 4. sx prop — Box 의 power

```tsx
<Box sx={{
  p: 2,                          // padding: theme.spacing(2) = 16px
  m: { xs: 1, md: 3 },           // responsive
  bgcolor: 'primary.main',       // theme color
  color: 'primary.contrastText',
  borderRadius: 1,
  '&:hover': { opacity: 0.8 },   // pseudo-class
}}>
  ...
</Box>
```

- 숫자 = theme spacing 단위 (기본 8px).
- 객체 = breakpoint 별 값.

## 5. styled() — 커스텀 컴포넌트

```tsx
import { styled } from '@mui/material/styles';
import Button from '@mui/material/Button';

const RedButton = styled(Button)(({ theme }) => ({
  backgroundColor: 'red',
  '&:hover': { backgroundColor: 'darkred' },
}));
```

→ `@emotion/styled` 기반. styled-components 와 거의 같은 문법.

## 6. theme 커스터마이즈

```tsx
const theme = createTheme({
  palette: {
    primary: { main: '#0064FF', light: '#5A8FFF', dark: '#003ECC' },
    text: { primary: '#1A1A1A', secondary: '#666666' },
  },
  shape: { borderRadius: 8 },
  typography: {
    fontFamily: 'Pretendard, sans-serif',
    h1: { fontSize: 32, fontWeight: 700 },
  },
  components: {
    MuiButton: {
      defaultProps: { disableElevation: true },
      styleOverrides: {
        root: { textTransform: 'none' },
      },
    },
  },
});
```

→ 모든 컴포넌트의 기본값 / 스타일을 한 곳에서.

### TypeScript theme 확장
```ts
declare module '@mui/material/styles' {
  interface Theme {
    custom: { sidebar: { width: number } };
  }
  interface ThemeOptions {
    custom?: { sidebar?: { width?: number } };
  }
}
```

## 7. Icon

```tsx
import SaveIcon from '@mui/icons-material/Save';
import { Search, Delete } from '@mui/icons-material';

<SaveIcon />
<Search fontSize="small" color="primary" />
```

→ tree-shaking 위해 named import 권장.

## 8. answer-fe 패턴

```tsx
// theme.ts — 디자인 시스템
export const theme = createTheme({ ... });

// App.tsx
<ThemeProvider theme={theme}>
  <CssBaseline />
  <Routes>...</Routes>
</ThemeProvider>

// 컴포넌트
import { Button, TextField, Stack } from '@mui/material';

const LoginForm = () => (
  <Stack spacing={2} sx={{ maxWidth: 400 }}>
    <TextField label="이메일" />
    <TextField label="비밀번호" type="password" />
    <Button variant="contained">로그인</Button>
  </Stack>
);
```

## 9. styled-components 와 공존

```tsx
// MUI + styled-components 혼용
import { Button } from '@mui/material';
import styled from 'styled-components';

const StyledButton = styled(Button)`
  background: linear-gradient(45deg, #FE6B8B 30%, #FF8E53 90%);
`;
```

→ `answer-fe` 가 둘 다 사용. emotion (MUI 기본) 과 styled-components 동시 → CSS prefix 또는 우선순위 주의.

## 10. 번들 최적화

```ts
// ✅ named import (tree-shake)
import { Button, TextField } from '@mui/material';

// ❌ default
import Material from '@mui/material';   // 전체 import
```

→ Vite 가 자동 tree-shake. 단 큰 컴포넌트 (DataGrid 등) 는 lazy 권장.

```tsx
const DataGrid = lazy(() => import('@mui/x-data-grid'));
```

## 11. 함정

1. **`<Grid item xs={12} md={6}>` 의 v5 vs v7** — v7+ 의 Grid2 는 `item` 안 씀. 마이그레이션 시 주의.
2. **sx prop 안 함수** — `sx={(theme) => ({ ... })}` 도 가능.
3. **theme palette 누락** — `primary.main` 만 정의 시 `light/dark/contrastText` 자동 계산. 색상 정확히 원하면 명시.
4. **모든 컴포넌트 import** — `import Material from '@mui/material'` ❌. named only.
5. **CssBaseline 누락** — 일관된 reset 없으면 브라우저별 스타일.
6. **DatePicker locale** — 한국어 적용:
   ```tsx
   import 'dayjs/locale/ko';
   <LocalizationProvider dateAdapter={AdapterDayjs} adapterLocale="ko">
   ```
7. **MUI v6+ 의 emotion → pigment** — v6 부터 CSS-in-JS 변화. v5 → v6 마이그레이션 시 검토.

## 12. 외부 자료

- [MUI 공식](https://mui.com/)
- [MUI Templates](https://mui.com/templates/)
- [Material Icons](https://fonts.google.com/icons)

## 13. 관련

- [[ui-libraries]]
- [[antd]]
- [[../styling/styled-components]]
