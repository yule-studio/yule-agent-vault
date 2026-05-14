---
title: "Configuration — Vite / TypeScript / ESLint / Prettier"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [frontend, react, configuration, vite, tsconfig, eslint, prettier]
---

# Configuration — Vite / TypeScript / ESLint / Prettier

**[[react|↑ React hub]]**

> 개발 환경 설정. 처음에는 이 파일들이 뭐 하는 건지 알기 어려움. 한 번 이해하면 어떤 React 프로젝트든 설정 파일 보고 "이 프로젝트는 이렇게 돌아가는구나" 즉시 파악.

## 1. `package.json` — 프로젝트의 명세서

`masterway-dev/job-answer-fe/package.json` 발췌:

```json
{
  "name": "job-answer-fe",
  "private": true,
  "version": "0.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite --host 0.0.0.0 --mode dev",
    "build": "tsc && vite build",
    "build:main": "yarn build --mode main",
    "build:stage": "yarn build --mode stage",
    "build:test": "yarn build --mode test"
  },
  "dependencies": {
    "react": "18",
    "react-dom": "18",
    "axios": "^1.13.2",
    ...
  },
  "devDependencies": {
    "vite": "^6.3.1",
    "typescript": "~5.7.2",
    ...
  }
}
```

### 핵심 필드
- **`scripts`**: `yarn dev` 또는 `npm run dev` 로 실행할 명령들.
- **`dependencies`**: production 에 필요한 라이브러리.
- **`devDependencies`**: 개발 / 빌드 에만 필요 (vite, typescript, eslint).
- **`type: "module"`**: ESM (import/export) 사용 (현대 표준).

### Version 표기
- `^1.13.2` — 1.13.2 이상, 2.x.x 미만 (minor + patch 업그레이드 허용).
- `~5.7.2` — 5.7.2 이상, 5.8.0 미만 (patch 만).
- `5.7.2` — 정확히 5.7.2.
- `18` — 18.x.x 의 최신.

### 자주 쓰는 scripts
```json
{
  "dev": "vite",                          // 개발 서버
  "build": "tsc && vite build",           // 프로덕션 빌드
  "preview": "vite preview",              // 빌드 결과 미리 보기
  "lint": "eslint ./src --ext .ts,.tsx",  // 린트 검사
  "lint:fix": "eslint ./src --fix",       // 자동 수정
  "format": "prettier --write \"src/**/*.{ts,tsx,css}\""  // 포맷
}
```

실행: `npm run dev` 또는 `yarn dev`.

## 2. `vite.config.ts` — Vite 설정

```ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig({
  plugins: [react()],

  // 절대 경로 alias
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
      '@components': path.resolve(__dirname, './src/components'),
      '@hooks': path.resolve(__dirname, './src/hooks'),
    },
  },

  // 개발 서버 설정
  server: {
    host: '0.0.0.0',     // 다른 기기에서 접근 가능
    port: 5173,
    open: true,          // 자동으로 브라우저 열기
    proxy: {
      // /api 로 시작하는 요청을 백엔드로 forward (CORS 회피)
      '/api': {
        target: 'http://localhost:8080',
        changeOrigin: true,
      },
    },
  },

  // 빌드 설정
  build: {
    outDir: 'dist',
    sourcemap: false,
    chunkSizeWarningLimit: 1000,    // 1MB 까지 경고 안 함
  },
});
```

### 핵심 항목
- **alias**: `import x from '@/components/Button'` 처럼 절대 경로.
- **proxy**: dev 시 백엔드 API 우회 (CORS 문제 해결).
- **mode**: `vite --mode test` 가 `.env.test` 자동 로드.

### 환경 별 mode
```bash
yarn dev                  # .env, .env.development 로드
yarn build --mode stage   # .env, .env.stage 로드
yarn build --mode main    # .env, .env.main 로드
```

`masterway-dev/job-answer-fe` 가 정확히 이 패턴: `build:main` / `build:stage` / `build:test`.

## 3. `tsconfig.json` — TypeScript 설정

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",

    "strict": true,                       // ★ 모든 strict 옵션 활성
    "noUnusedLocals": true,               // 안 쓴 변수 에러
    "noUnusedParameters": true,           // 안 쓴 함수 인자 에러
    "noFallthroughCasesInSwitch": true,
    "noImplicitReturns": true,

    "allowSyntheticDefaultImports": true,
    "esModuleInterop": true,
    "skipLibCheck": true,                 // 라이브러리 .d.ts 검사 skip (속도 ↑)
    "resolveJsonModule": true,            // .json import 가능
    "isolatedModules": true,              // 각 파일 독립 컴파일 (Vite 요구)

    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]                    // ★ Vite alias 와 일치
    }
  },
  "include": ["src", "vite.config.ts"],
  "exclude": ["node_modules", "dist"]
}
```

### 핵심 옵션 의미
- **`strict: true`** — TypeScript 의 모든 안전 옵션 활성. 절대 끄지 마.
- **`jsx: "react-jsx"`** — JSX 컴파일 모드. import React from 'react' 불필요.
- **`paths`** — alias 정의 (vite.config.ts 와 일치 필수).
- **`isolatedModules`** — Vite / esbuild 가 파일별 독립 컴파일 가능하도록.

### TypeScript 의 strict 옵션 (켜져야 하는 것들)
- `strictNullChecks` — `null` / `undefined` 명시.
- `noImplicitAny` — 타입 추론 안 되면 `any` 자동 X → 에러.
- `strictFunctionTypes` — 함수 시그니처 더 엄격.
- 모두 `"strict": true` 하나로 켜짐.

## 4. `.eslintrc` (또는 `eslint.config.js` Flat config) — 코드 품질

```js
// eslint.config.js (Flat config, ESLint 9+)
import js from '@eslint/js';
import tseslint from 'typescript-eslint';
import reactHooks from 'eslint-plugin-react-hooks';
import reactRefresh from 'eslint-plugin-react-refresh';

export default tseslint.config(
  { ignores: ['dist'] },
  {
    extends: [js.configs.recommended, ...tseslint.configs.recommended],
    files: ['**/*.{ts,tsx}'],
    languageOptions: {
      ecmaVersion: 2020,
    },
    plugins: {
      'react-hooks': reactHooks,
      'react-refresh': reactRefresh,
    },
    rules: {
      ...reactHooks.configs.recommended.rules,
      'react-refresh/only-export-components': [
        'warn',
        { allowConstantExport: true },
      ],
    },
  },
);
```

### ESLint 가 잡는 것
- 안 쓴 변수, 안 쓴 import.
- 잘못된 hook 사용 (`useEffect` 의존성 누락 등).
- React 의 안티패턴.
- TypeScript 의 type 위반.

### 실행
```bash
yarn lint              # 검사
yarn lint:fix          # 자동 수정
```

VS Code 에서 **ESLint 확장** 설치하면 코드 작성 중 실시간 표시.

## 5. `.prettierrc` — 코드 포맷

```json
{
  "semi": true,
  "trailingComma": "all",
  "singleQuote": true,
  "printWidth": 100,
  "tabWidth": 2,
  "arrowParens": "always",
  "endOfLine": "lf"
}
```

### Prettier vs ESLint
- **ESLint**: 코드 품질 (버그 잡기).
- **Prettier**: 코드 포맷 (들여쓰기 / 따옴표 / 세미콜론).
- 둘이 충돌하면 `eslint-config-prettier` 로 prettier 우선.

### VS Code 설정 — 저장 시 자동 포맷

`.vscode/settings.json`:
```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit"
  }
}
```

→ Cmd+S 누르면 자동 ESLint fix + Prettier 포맷.

## 6. 환경 변수 — `.env*`

```bash
# .env (모든 모드 공통)
VITE_APP_NAME=My App

# .env.development (yarn dev)
VITE_API_URL=http://localhost:8080

# .env.production (yarn build)
VITE_API_URL=https://api.production.com

# .env.local (git X — 개인 secret)
VITE_API_KEY=mysecret
```

### Vite 의 규칙
- **`VITE_` 로 시작하는 것만** client 코드에서 접근 가능 (보안).
- 사용:
  ```tsx
  const apiUrl = import.meta.env.VITE_API_URL;
  ```
- `.env.local` 은 절대 git 에 안 올림 (`.gitignore` 에 포함).

### `masterway-dev/answer-fe` 의 .env 패턴
```
.env.master   → 운영
.env.stage    → 스테이징
.env.test     → 테스트
```

`yarn build:master` → `.env.master` 사용.

## 7. `index.html` — HTML 진입점

```html
<!DOCTYPE html>
<html lang="ko">
  <head>
    <meta charset="UTF-8" />
    <link rel="icon" type="image/svg+xml" href="/favicon.svg" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>%VITE_APP_NAME%</title>
  </head>
  <body>
    <div id="root"></div>          ← ★ React 가 그릴 자리
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

- `<div id="root">` 가 React 의 root.
- `main.tsx` 가 그 안에 `<App />` 그림.
- meta / title 도 여기서 설정 (또는 `react-helmet` 같은 라이브러리로).

## 8. `.gitignore` — git 에 안 올릴 것

```
# 의존성
node_modules/

# 빌드 결과
dist/
build/

# 환경 변수 (개인 secret)
.env.local
.env*.local

# IDE
.vscode/*
!.vscode/extensions.json
.idea/

# 로그
*.log
npm-debug.log*
yarn-debug.log*
yarn-error.log*

# OS
.DS_Store
Thumbs.db
```

→ `node_modules/` 가 가장 중요. 절대 git 에 올리지 마 (수 GB 됨).

## 9. Yarn 의 추가 파일

```
.yarnrc.yml          ← yarn 설정
.pnp.cjs             ← Yarn Plug'n'Play (PnP) 모드 시
yarn.lock            ← 의존성 정확한 버전 lock
```

→ `yarn.lock` 은 **반드시 git 에 올림**. 팀이 같은 버전 의존성 보장.

## 10. 자주 마주치는 설정 함정

### 함정 1 — alias 설정 vite + tsconfig 불일치
- `vite.config.ts` 에 `'@': './src'` 설정.
- `tsconfig.json` 에 `paths` 안 함.
- 결과: 빌드 OK, IDE 에서 빨간 줄.
- 해법: 둘 다 동일하게.

### 함정 2 — `.env` 의 변경 인식 안 함
- Vite 가 `.env` 변경을 자동 reload 안 함.
- 해법: `Ctrl+C` 후 다시 `yarn dev`.

### 함정 3 — `VITE_` prefix 없는 환경 변수
- `MY_SECRET=...` 로 작성. `import.meta.env.MY_SECRET` 가 `undefined`.
- Vite 의 보안 — `VITE_` 없으면 client 노출 안 함.

### 함정 4 — TypeScript strict 끄기
- `strict: false` 로 두면 처음엔 편하지만 곧 type 안전성 잃음.
- 강제 `strict: true` 권장.

### 함정 5 — ESLint + Prettier 충돌
- ESLint 가 작은 따옴표 강제, Prettier 가 큰 따옴표.
- 해법: `eslint-config-prettier` 추가, ESLint 의 포맷 룰 끄기.

### 함정 6 — Yarn / npm 혼용
- 한 프로젝트에서 `npm install` + `yarn install` 섞으면 lock 파일 2 개.
- 해법: 1 개만 (lock 파일 하나 만)

## 11. 자기 프로젝트 분석 — `answer-fe`

```bash
cd ~/masterway-dev/answer-fe

# 어떤 매니저?
ls package-lock.json yarn.lock 2>/dev/null

# scripts 확인
cat package.json | python3 -c "import sys, json; print(json.load(sys.stdin)['scripts'])"

# Vite 설정 보기
cat vite.config.ts 2>/dev/null || cat vite.config.js

# TS 설정
cat tsconfig.json | head -30
```

## 12. 다음 단계

- [[core-concepts/components-and-jsx]] — 컴포넌트 깊이
- [[core-concepts/props-and-state]] — props 와 state

## 13. 관련

- [[react]]
- [[project-structure]]
