# pnpm + Turborepo 的 React + TypeScript 单仓库（Monorepo）搭建指南

## 摘要

从零搭建一个 React + TypeScript 单仓库：pnpm workspace 管理多包，Turborepo 编排 `build/dev/lint/typecheck/test` 任务，`apps/` 放可独立运行的应用，`packages/` 放共享库（共享 TS 配置、UI 组件、工具函数），并覆盖质量校验与 CI。本文按步骤给出可直接复制的配置，并对每个关键字段逐条注解其作用。

## 核心内容

### 架构总览

```
react-ts-monorepo/
├── apps/
│   └── web/                    @repo/web      React 19 + Vite 8（可独立运行）
├── packages/
│   ├── ui/                     @repo/ui       共享 React 组件（JIT，导出 src）
│   ├── utils/                  @repo/utils    共享 TS 工具（JIT，导出 src）
│   └── tsconfig/               @repo/tsconfig 共享 tsconfig 预设（纯配置包）
├── package.json                根脚本 + 共享工具链
├── pnpm-workspace.yaml         workspace 定义 + allowBuilds
├── turbo.json                  任务编排 / 缓存
├── eslint.config.js            根 flat config（子包复用）
├── .prettierrc.json
├── .gitignore
└── .github/workflows/ci.yml
```

关键设计取舍：

- **apps = 可独立部署的应用**，**packages = 被复用的库**。两者都放在 workspace glob 下。
- **JIT Packages（即时包）**：内部包不预构建，`package.json` 的 `exports` 直接把 `./src/index.ts` 暴露给消费方，由消费方（Vite）负责转译。优点是改共享库源码后应用 HMR 立即生效、无需 watch 构建；缺点是若要发布到 npm，需改为 tsc/tsup 产物包。
- 所有跨包引用一律用 `workspace:*`，不写具体版本号。
- 每个要跑 `tsc` / `vitest` 的包，必须在**自己的** `devDependencies` 里声明 `typescript` / `vitest`，不能依赖根提升（pnpm 默认严格 node_modules）。

### 版本矩阵（2026-09 核实）

| 包 | 版本 | 说明 |
| --- | --- | --- |
| pnpm / node | 12.5.1 / 24 | 环境基线 |
| turbo | ^2.11.2 | |
| typescript | **^6.0.3** | 最新是 7.0.2，但 `typescript-eslint@8.70` peer 要求 `<6.1.0`，不要装 7 |
| vite | ^8.3.0 | Node ≥22.12 |
| @vitejs/plugin-react | ^6.1.1 | |
| react / react-dom | ^19.3.0 | |
| @types/react(-dom) | ^19.3.0 | |
| vitest | ^5.0.1 | |
| jsdom | ^30.1.0 | |
| @testing-library/react | ^16.3.3 | |
| @testing-library/jest-dom | ^7.0.1 | |
| eslint / @eslint/js | ^10.11.0 / ^10.0.1 | |
| typescript-eslint | ^8.70.0 | |
| globals | ^17.12.0 | |
| eslint-plugin-react-hooks | ^7.1.1 | |
| eslint-plugin-react-refresh | ^0.5.7 | |
| eslint-config-prettier | ^10.1.8 | |
| prettier | ^3.9.8 | |
| @changesets/cli | ^3.0.3 | 可选，版本发布 |

## 示例

### Phase 1 — 工作区骨架

`pnpm-workspace.yaml`：

```yaml
packages:
  - "apps/*"
  - "packages/*"
allowBuilds:
  esbuild: true
```

| 字段 | 作用 |
| --- | --- |
| `packages` | 声明哪些目录是 workspace 成员。`apps/*`、`packages/*` 各匹配一层子目录。 |
| `allowBuilds.esbuild` | pnpm 10+ 默认禁止依赖执行 install/postinstall 脚本（供应链安全）。Vite 若用到 esbuild，需显式允许它执行二进制安装脚本；值为 `true` 允许、`false` 拒绝。 |

> pnpm 11+ 起 `.npmrc` 只保留 registry/auth 相关配置，`allowBuilds` 这类设置必须写在 `pnpm-workspace.yaml`。若安装时提示 `Ignored build scripts`，也可改用 `pnpm approve-builds` 交互批准。

根 `package.json`：

```json
{
  "name": "react-ts-monorepo",
  "version": "0.0.0",
  "private": true,
  "type": "module",
  "packageManager": "pnpm@12.5.1",
  "scripts": {
    "dev": "turbo run dev",
    "build": "turbo run build",
    "lint": "turbo run lint",
    "typecheck": "turbo run typecheck",
    "test": "turbo run test",
    "format": "prettier --write .",
    "format:check": "prettier --check ."
  },
  "devDependencies": {
    "@eslint/js": "^10.0.1",
    "eslint": "^10.11.0",
    "eslint-config-prettier": "^10.1.8",
    "eslint-plugin-react-hooks": "^7.1.1",
    "eslint-plugin-react-refresh": "^0.5.7",
    "globals": "^17.12.0",
    "prettier": "^3.9.8",
    "turbo": "^2.11.2",
    "typescript": "^6.0.3",
    "typescript-eslint": "^8.70.0"
  }
}
```

| 字段 | 作用 |
| --- | --- |
| `private: true` | 防止根包被误发布到 npm。 |
| `type: "module"` | 根内 `.js` 按 ESM 解析，配置脚本用 `import`。 |
| `packageManager` | 锁定包管理器及版本，Corepack / CI 据此选择。 |
| `scripts.*` | 全部委托 `turbo run <task>`，由 turbo 并发跑各子包的同名脚本。 |
| `devDependencies` | 仅根级的工具链（turbo、eslint 全家桶、prettier、typescript）。 |

`.gitignore`：

```
node_modules/
dist/
.turbo/
*.log
.DS_Store
```

| 项 | 作用 |
| --- | --- |
| `node_modules/` | 不提交依赖。 |
| `dist/` | 不提交构建产物。 |
| `.turbo/` | turbo 本地缓存与日志目录。 |

初始化：`git init`。

### Phase 2 — `packages/tsconfig`（共享 TS 预设）

`packages/tsconfig/package.json`：

```json
{
  "name": "@repo/tsconfig",
  "version": "0.0.0",
  "private": true,
  "files": ["base.json", "react-library.json", "react-app.json"],
  "exports": {
    "./base.json": "./base.json",
    "./react-library.json": "./react-library.json",
    "./react-app.json": "./react-app.json"
  }
}
```

| 字段 | 作用 |
| --- | --- |
| `files` | npm/pnpm 发布白名单，只在 `pnpm pack/publish` 时生效。该包 `private` 不发布，此字段是面向未来的语义声明；**不影响**本地 workspace 引用。 |
| `exports` | 子路径导出映射，其它包才能写 `"extends": "@repo/tsconfig/base.json"`。省略它则无法按路径引用。 |

`base.json`（所有包的共同基线）：

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2023"],
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "moduleDetection": "force",
    "verbatimModuleSyntax": true,
    "isolatedModules": true,
    "resolveJsonModule": true,
    "allowImportingTsExtensions": true,
    "noEmit": true,
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true
  }
}
```

| 字段 | 作用 |
| --- | --- |
| `target` | 编译目标语法版本（如 `??`、class fields 降级到哪一代）；也决定缺省 `lib`。 |
| `lib` | 类型层面可用的内置 API 声明（`Array`、`Promise` 等）。与 `target` 独立；**不含 DOM**，浏览器类型在 react-library 里加。 |
| `module` | 模块系统，`ESNext` 表示按最新 ESM 解析/产出 `import`/`export`。 |
| `moduleResolution` | 模块解析策略。`Bundler` 与 Vite/Rollup 一致：允许省略扩展名、读 `exports` 字段。 |
| `moduleDetection` | `force` 强制所有文件都当作模块，避免无 import 的文件被当全局脚本。 |
| `verbatimModuleSyntax` | 强制类型导入用 `import type`，类型/值不混用。 |
| `isolatedModules` | 每个文件独立转译（与 esbuild/oxc 一致）。 |
| `resolveJsonModule` | 允许 `import` JSON。 |
| `allowImportingTsExtensions` | 允许 import 时带 `.ts` 后缀；需配合 `noEmit`。 |
| `noEmit` | 只做类型检查，不产出文件（本项目由 Vite 产出）。 |
| `strict` | 严格模式总开关。 |
| `noUncheckedIndexedAccess` | 索引访问结果加上 `undefined`，更严谨。 |
| `noUnusedLocals` / `noUnusedParameters` | 报错未使用的局部变量/参数。 |
| `noFallthroughCasesInSwitch` | 禁止 switch 隐式贯穿。 |
| `forceConsistentCasingInFileNames` | 强制文件名大小写一致，避免跨平台问题。 |
| `skipLibCheck` | 跳过 `node_modules` 里 `.d.ts` 的检查，加快编译。 |

`react-library.json`（React 库）：

```json
{
  "extends": "./base.json",
  "compilerOptions": {
    "lib": ["ES2023", "DOM", "DOM.Iterable"],
    "jsx": "react-jsx"
  }
}
```

| 字段 | 作用 |
| --- | --- |
| `extends` | 继承 base，只覆盖差异项。 |
| `lib` 加 `DOM`/`DOM.Iterable` | 提供 `window`、`document` 等浏览器全局类型；没有它 `window` 会 `TS2304`。 |
| `jsx: "react-jsx"` | 新版 JSX 转换，无需在每个文件 `import React`。 |

`react-app.json`（React 应用，额外要 Vite 类型）：

```json
{
  "extends": "./react-library.json",
  "compilerOptions": {
    "types": ["vite/client"]
  }
}
```

| 字段 | 作用 |
| --- | --- |
| `types: ["vite/client"]` | 引入 Vite 的客户端类型：`import.meta.env`、`import.meta.hot`、`*.css` / `*.module.css` / 静态资源导入声明等。 |

### Phase 3 — `packages/utils`（JIT 工具包）

`package.json`：

```json
{
  "name": "@repo/utils",
  "version": "0.0.0",
  "private": true,
  "type": "module",
  "exports": { ".": "./src/index.ts" },
  "scripts": {
    "lint": "eslint .",
    "typecheck": "tsc --noEmit",
    "test": "vitest run"
  },
  "devDependencies": {
    "@repo/tsconfig": "workspace:*",
    "typescript": "^6.0.3",
    "vitest": "^5.0.1"
  }
}
```

| 字段 | 作用 |
| --- | --- |
| `exports` | JIT 核心：包入口直接指向 `./src/index.ts`，消费方拿到源码自行转译，无需构建。 |
| `type: "module"` | 该包按 ESM 解析。 |
| `scripts.typecheck` | `tsc --noEmit` 只检查不产出。 |
| `devDependencies` | 自带 `typescript`/`vitest`，满足 pnpm 严格依赖。 |
| `workspace:*` | 引用本仓其他包，不使用具体版本号。 |

`tsconfig.json`：

```json
{
  "extends": "@repo/tsconfig/base.json",
  "include": ["src"]
}
```

| 字段 | 作用 |
| --- | --- |
| `extends` | 通过包名解析共享预设（靠 `@repo/tsconfig` 的 `exports`）。 |
| `include` | 只检查 `src/`。 |

`src/index.ts`：

```ts
export function formatPrice(cents: number, currency = '¥'): string {
  return `${currency}${(cents / 100).toFixed(2)}`
}

export function clamp(value: number, min: number, max: number): number {
  return Math.min(Math.max(value, min), max)
}
```

`src/index.test.ts`：

```ts
import { describe, expect, it } from 'vitest'
import { clamp, formatPrice } from './index'

describe('utils', () => {
  it('formats price', () => {
    expect(formatPrice(1234)).toBe('¥12.34')
  })
  it('clamps', () => {
    expect(clamp(5, 0, 3)).toBe(3)
  })
})
```

`vitest.config.ts`：

```ts
import { defineConfig } from 'vitest/config'

export default defineConfig({ test: { environment: 'node' } })
```

| 字段 | 作用 |
| --- | --- |
| `test.environment` | 测试运行环境，纯逻辑用 `node`；需要 DOM 用 `jsdom`。 |

### Phase 4 — `packages/ui`（JIT React 组件包）

`package.json`：

```json
{
  "name": "@repo/ui",
  "version": "0.0.0",
  "private": true,
  "type": "module",
  "exports": { ".": "./src/index.ts" },
  "scripts": {
    "lint": "eslint .",
    "typecheck": "tsc --noEmit",
    "test": "vitest run"
  },
  "dependencies": {
    "@repo/utils": "workspace:*"
  },
  "peerDependencies": {
    "react": "^19.0.0",
    "react-dom": "^19.0.0"
  },
  "devDependencies": {
    "@repo/tsconfig": "workspace:*",
    "@testing-library/jest-dom": "^7.0.1",
    "@testing-library/react": "^16.3.3",
    "@types/react": "^19.3.0",
    "@types/react-dom": "^19.3.0",
    "jsdom": "^30.1.0",
    "react": "^19.3.0",
    "react-dom": "^19.3.0",
    "typescript": "^6.0.3",
    "vitest": "^5.0.1"
  }
}
```

| 字段 | 作用 |
| --- | --- |
| `dependencies` | 该组件库运行时真正依赖的包（如 `@repo/utils`）。 |
| `peerDependencies` | 期望消费方提供的 React，避免同一 React 被装两份导致 hooks 报错；本包开发时用 `devDependencies` 提供。 |

`tsconfig.json`：

```json
{
  "extends": "@repo/tsconfig/react-library.json",
  "include": ["src"]
}
```

`src/Button.tsx`（含 CSS Modules 样式）：

```tsx
import type { ReactNode } from 'react'
import { formatPrice } from '@repo/utils'
import styles from './Button.module.css'

export interface ButtonProps {
  children: ReactNode
  priceCents?: number
  onClick?: () => void
}

export function Button({ children, priceCents, onClick }: ButtonProps) {
  return (
    <button type="button" className={styles.button} onClick={onClick}>
      <span className={styles.label}>{children}</span>
      {priceCents !== undefined ? (
        <span className={styles.price}>{formatPrice(priceCents)}</span>
      ) : null}
    </button>
  )
}
```

`src/Button.module.css`：

```css
.button {
  display: inline-flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.5rem 1rem;
  border: 1px solid transparent;
  border-radius: 0.5rem;
  background: #2563eb;
  color: #ffffff;
  font: inherit;
  font-weight: 600;
  line-height: 1.2;
  cursor: pointer;
  transition:
    background-color 0.15s ease,
    transform 0.05s ease;
}

.button:hover {
  background: #1d4ed8;
}

.button:active {
  transform: translateY(1px);
}

.button:focus-visible {
  outline: 2px solid #93c5fd;
  outline-offset: 2px;
}

.label {
  white-space: nowrap;
}

.price {
  padding: 0.125rem 0.4rem;
  border-radius: 999px;
  background: rgba(255, 255, 255, 0.2);
  font-size: 0.85em;
}
```

`src/css.d.ts`（让 `tsc` 认识 CSS Modules；ui 未装 `vite/client`，用它避免额外依赖）：

```ts
declare module '*.module.css' {
  const classes: Record<string, string>
  export default classes
}
```

`src/index.ts`：

```ts
export { Button } from './Button'
export type { ButtonProps } from './Button'
```

`vitest.setup.ts`：

```ts
import '@testing-library/jest-dom/vitest'
```

`vitest.config.ts`：

```ts
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: ['./vitest.setup.ts']
  }
})
```

| 字段 | 作用 |
| --- | --- |
| `environment: 'jsdom'` | 提供 DOM 环境，组件测试必需。 |
| `globals: true` | 暴露全局 `afterEach` 等，Testing Library 才能自动在用例间 cleanup；否则第二个用例会看到上一个用例残留的 DOM。 |
| `setupFiles` | 每个测试文件前执行，这里注册 jest-dom 匹配器。 |

`src/Button.test.tsx`：

```tsx
import { render, screen } from '@testing-library/react'
import { describe, expect, it } from 'vitest'
import { Button } from './Button'

describe('Button', () => {
  it('renders children', () => {
    render(<Button>Buy</Button>)
    expect(screen.getByRole('button').textContent).toBe('Buy')
  })

  it('renders formatted price', () => {
    render(<Button priceCents={500}>Buy</Button>)
    expect(screen.getByRole('button').textContent).toContain('¥5.00')
  })
})
```

### Phase 5 — `apps/web`（React + Vite 应用）

`package.json`：

```json
{
  "name": "@repo/web",
  "version": "0.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview",
    "lint": "eslint .",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "@repo/ui": "workspace:*",
    "@repo/utils": "workspace:*",
    "react": "^19.3.0",
    "react-dom": "^19.3.0"
  },
  "devDependencies": {
    "@repo/tsconfig": "workspace:*",
    "@types/react": "^19.3.0",
    "@types/react-dom": "^19.3.0",
    "@vitejs/plugin-react": "^6.1.1",
    "typescript": "^6.0.3",
    "vite": "^8.3.0"
  }
}
```

| 字段 | 作用 |
| --- | --- |
| `dependencies` | 应用运行需要的 React 和两个内部包，内部包用 `workspace:*`。 |
| `@vitejs/plugin-react` | 提供 React Fast Refresh 与 JSX 转换。 |

`vite.config.ts`：

```ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  server: { port: 5173 }
})
```

`tsconfig.json`（**必需**，否则 `tsc --noEmit` 找不到配置会打印帮助并退出 1）：

```json
{
  "extends": "@repo/tsconfig/react-app.json",
  "include": ["src", "vite.config.ts"]
}
```

`index.html`（必须与 `src/`、`package.json` 同目录）：

```html
<!doctype html>
<html lang="zh-CN">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>react-ts-monorepo</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

`src/main.tsx`：

```tsx
import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import { App } from './App'

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <App />
  </StrictMode>
)
```

`src/App.tsx`：

```tsx
import { Button } from '@repo/ui'
import { clamp } from '@repo/utils'
import { useState } from 'react'

export function App() {
  const [count, setCount] = useState(0)
  return (
    <main>
      <h1>react-ts-monorepo</h1>
      <Button priceCents={999} onClick={() => setCount((c) => clamp(c + 1, 0, 10))}>
        count: {count}
      </Button>
    </main>
  )
}
```

### Phase 6 — Turborepo

`turbo.json`（放在**仓库根目录**，整个仓库一份）：

```json
{
  "$schema": "https://turbo.build/schema.json",
  "ui": "tui",
  "tasks": {
    "build": { "dependsOn": ["^build"], "outputs": ["dist/**"] },
    "dev": { "cache": false, "persistent": true },
    "lint": {},
    "typecheck": {},
    "test": {}
  }
}
```

| 字段 | 作用 |
| --- | --- |
| `$schema` | 编辑器 JSON 校验/补全。 |
| `ui: "tui"` | 使用交互式终端界面展示任务进度。 |
| `tasks` | v2 的任务定义（旧版叫 `pipeline`，已废弃）。 |
| `build.dependsOn: ["^build"]` | 先构建上游依赖包，再构建本包（拓扑顺序）。 |
| `build.outputs: ["dist/**"]` | 声明产物路径供 turbo 缓存命中/回放。 |
| `dev.cache: false` | 长驻进程不缓存。 |
| `dev.persistent: true` | 标记为常驻任务，turbo 不会等它结束。 |
| `lint/typecheck/test` | 空配置即可，turbo 会自动并发执行各包同名脚本并缓存。 |

### Phase 7 — ESLint + Prettier

根 `eslint.config.js`（flat config，子包 `eslint .` 会自动向上找到它并复用）：

```js
import js from '@eslint/js'
import prettier from 'eslint-config-prettier'
import reactHooks from 'eslint-plugin-react-hooks'
import reactRefresh from 'eslint-plugin-react-refresh'
import globals from 'globals'
import tseslint from 'typescript-eslint'

export default tseslint.config(
  { ignores: ['**/dist/**', '**/node_modules/**', '**/.turbo/**'] },
  js.configs.recommended,
  ...tseslint.configs.recommended,
  {
    files: ['**/*.{ts,tsx}'],
    languageOptions: { globals: { ...globals.browser, ...globals.node } },
    plugins: { 'react-hooks': reactHooks, 'react-refresh': reactRefresh },
    rules: {
      ...reactHooks.configs.recommended.rules,
      'react-refresh/only-export-components': 'warn'
    }
  },
  prettier
)
```

| 项 | 作用 |
| --- | --- |
| `ignores` | 忽略产物/依赖/缓存目录。 |
| `js.configs.recommended` | JS 基础推荐规则。 |
| `...tseslint.configs.recommended` | TS 推荐规则，并关闭与 TS 冲突的 JS 规则。 |
| `languageOptions.globals` | 声明全局变量，避免 `no-undef` 误报 `window`/`process`。 |
| `reactHooks` | React Hooks 规则（依赖数组等）。 |
| `react-refresh/only-export-components` | 保证文件只导出组件，让 Fast Refresh 正常工作。 |
| `prettier` | 放最后，关闭与 Prettier 冲突的格式规则。 |

`.prettierrc.json`：

```json
{ "semi": false, "singleQuote": true, "trailingComma": "none", "printWidth": 100 }
```

`.prettierignore`：

```
dist
.turbo
pnpm-lock.yaml
node_modules
```

### Phase 8 — 安装与验证

```bash
pnpm install
pnpm lint
pnpm typecheck
pnpm test
pnpm build
pnpm dev            # 浏览器打开 http://localhost:5173
```

验收点：应用能 import 到 `@repo/ui`、`@repo/utils` 源码；改 `utils` 后应用 HMR 生效；`turbo run build` 第二次命中缓存（FULL TURBO）。

### Phase 9 — CI（GitHub Actions）与 Changesets（可选）

#### 9.1 `ci.yml` 最终形态

文件必须放在仓库根的 `.github/workflows/` 下、扩展名为 `.yml`，GitHub 才会识别。

`.github/workflows/ci.yml`：

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

permissions:
  contents: read

jobs:
  ci:
    name: Lint / Typecheck / Test / Build
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup pnpm
        uses: pnpm/action-setup@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: 24
          cache: pnpm

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Format check
        run: pnpm format:check

      - name: Lint
        run: pnpm lint

      - name: Typecheck
        run: pnpm typecheck

      - name: Test
        run: pnpm test

      - name: Build
        run: pnpm build
```

#### 9.2 它是什么 / 为什么需要

CI（持续集成）：每次 push 或发起 PR 时，GitHub 自动在临时虚拟机上跑一遍 `format:check / lint / typecheck / test / build`。任一命令非零退出，该次运行失败，PR 上显示红叉；配合分支保护规则可禁止未通过检查的代码合入。

**配置不是固定的**：除顶层 `name` / `on` / `jobs` 三个结构外，触发条件、运行环境、Node 版本、执行哪些命令、是否用矩阵等全部可按需改。

#### 9.3 顶层字段

| 字段 | 作用 |
| --- | --- |
| `name` | 工作流显示名，出现在 Actions 页面。 |
| `on` | 触发条件（必需）。可写单个事件（`on: push`）或映射形式。 |
| `concurrency` | 并发控制。`group` 相同的运行互斥，`cancel-in-progress: true` 会取消同分支的旧运行，省资源。 |
| `permissions` | 授予 `GITHUB_TOKEN` 的权限，最小权限原则；只读代码用 `contents: read`。 |
| `env` | 全局环境变量，job / step 都能用。 |
| `jobs` | 一个或多个任务（必需），默认并行。 |

`on` 常用写法：

```yaml
on:
  push:
    branches: [main, develop]          # 只在这些分支 push 触发
    paths: ['apps/**', 'packages/**']  # 只有这些路径变更才触发
    tags: ['v*']
  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened]
  workflow_dispatch:                    # 允许在 Actions 页面手动点按钮运行
    inputs:
      node:
        description: 'Node version'
        default: '24'
  schedule:
    - cron: '0 3 * * *'                 # 每天 03:00 UTC 定时跑
  release:
    types: [published]
  workflow_call:                        # 供其他工作流调用
```

#### 9.4 `jobs` 字段

每个 job 在**独立虚拟机**上运行，默认并行；用 `needs` 建立依赖。

| 字段 | 作用 |
| --- | --- |
| `runs-on` | 运行环境：`ubuntu-latest` / `windows-latest` / `macos-latest` / `self-hosted`。 |
| `needs` | 依赖的 job（需其成功后才跑）。 |
| `if` | 条件表达式，决定是否执行该 job。 |
| `strategy.matrix` | 用变量组合生成多个并行 job（如多 Node 版本）。 |
| `strategy.fail-fast` | 矩阵中一个失败时是否取消其余（`false` 表示不取消）。 |
| `env` | 该 job 的环境变量。 |
| `timeout-minutes` | 超时上限，防止卡死。 |
| `continue-on-error` | 失败不计入整体失败。 |
| `outputs` | 暴露给下游 job 的值。 |
| `services` | 附带的服务容器（DB、Redis 等）。 |

矩阵示例：

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        node: [22, 24]
    steps:
      - run: echo "node ${{ matrix.node }}"
```

#### 9.5 `steps` 字段

步骤按顺序在同一 job 的同一工作区执行（文件会保留），但**环境变量不会自动跨步骤保留**。两种形态：

- `uses:` 复用现成 Action，用 `with:` 传参。
- `run:` 执行 shell 命令。

| 键 | 作用 |
| --- | --- |
| `name` | 步骤显示名。 |
| `uses` | 引用的 Action，如 `actions/checkout@v4`。 |
| `with` | 传给 Action 的输入参数。 |
| `run` | 执行的 shell 命令。 |
| `env` | 该步骤的环境变量。 |
| `if` | 条件执行。 |
| `id` | 步骤标识，供后续步骤引用其输出。 |
| `working-directory` | 指定命令工作目录。 |
| `shell` | 指定 shell。 |
| `continue-on-error` | 失败不影响整体。 |
| `timeout-minutes` | 超时上限。 |

跨步骤传值要写 `$GITHUB_ENV`（直接 `export` 不会带到下一步）：

```yaml
- run: echo "MY_VAR=value" >> "$GITHUB_ENV"
- run: echo "$MY_VAR"
```

#### 9.6 本仓各步骤为何这样写

| 步骤 | 作用 / 原因 |
| --- | --- |
| `actions/checkout@v4` | 把仓库代码拉到 runner，否则后续无代码可用。 |
| `pnpm/action-setup@v4` | 读取 `package.json` 的 `packageManager` 字段安装对应 pnpm 版本；**必须放在 `setup-node` 的 `cache: pnpm` 之前**，因为缓存需要先定位 pnpm store。 |
| `actions/setup-node@v4` | 安装指定 Node，并开启依赖缓存（`cache: pnpm` 缓存 pnpm store，加速安装）。 |
| `pnpm install --frozen-lockfile` | CI 中锁文件必须与 `package.json` 一致，不一致直接失败，保证可复现。 |
| `pnpm format:check` / `lint` / `typecheck` / `test` / `build` | 依次校验；任一非零退出即该 job 失败，并让 PR 检查变红。 |

#### 9.7 使用与注意事项

- 路径必须正确：`.github/workflows/*.yml`，YAML **只能用空格缩进、不能用 Tab**。
- Action 版本建议用大版本 tag（如 `@v4`）或提交 SHA（更安全）；升级前确认兼容性。
- 凭据用 `secrets` 注入（如 `${{ secrets.NPM_TOKEN }}`），**不要**写死在 YAML。
- `on: pull_request` 来自 fork 的 PR 默认拿不到 secret；涉及发布/部署要谨慎。
- 多 job 之间**不共享文件**（各自独立虚拟机），需用 artifact 或 cache 传递。
- 上传构建产物示例：

  ```yaml
  - uses: actions/upload-artifact@v4
    with:
      name: web-dist
      path: apps/web/dist
  ```

- 本地想模拟运行可用 [`act`](https://github.com/nektos/act)（基于 Docker）。
- Turborepo 远程缓存需额外配置 `env: TURBO_TOKEN` / `TURBO_TEAM`。
- 验证 YAML 合法性：`python3 -c "import yaml; yaml.safe_load(open('.github/workflows/ci.yml'))"`。

#### 9.8 与 Changesets 配合（可选）

Changesets：`pnpm add -D @changesets/cli -w`，`pnpm changeset init`，发布流程 `pnpm changeset` → `pnpm version-packages` → `pnpm release`（`private` 包默认不发布）。

## 注意事项

- **TypeScript 不要装 7.0.2**：`typescript-eslint@8.70` 的 peer 上限是 `<6.1.0`，用 `^6.0.3`。
- **pnpm 严格依赖**：要跑 `tsc`/`vitest` 的包必须自己声明 `typescript`/`vitest`，不能靠根提升。
- **JIT 包三要素**：`"type": "module"` + `exports` 指向 `./src/index.ts` + 消费方用 `workspace:*`。将来要发 npm 再改成产物包。
- **应用目录要完整**：`package.json`、`index.html`、`src/`、`tsconfig.json`、`vite.config.ts` 必须在同一目录。常见错误是把它们拆到 `apps/web` 与 `packages/web` 两处，或漏掉 `tsconfig.json`——症状是 `tsc --noEmit` 打印帮助文本并 `exit 1`。
- **turbo 报某个任务失败时**，先单独进该包 `pnpm exec tsc --noEmit` / `pnpm exec vitest run` 看真实错误；一个任务失败会让 turbo 连带取消并报告其他任务 `[ELIFECYCLE] Command failed`。
- **Testing Library 测试互相污染**：Vitest 未开 `globals: true` 时不会自动 cleanup，第二个用例会看到上一个用例的 DOM（`Found multiple elements with the role "button"`）。在 `test` 里开 `globals: true` 即可。
- **`turbo.json` 在仓库根**，不在子包内；子包只放各自 `package.json` 脚本。
- `turbo.json` v2 用 `tasks`，不是旧的 `pipeline`。
- 若 `pnpm install` 提示忽略构建脚本，运行 `pnpm approve-builds` 批准（对应 `allowBuilds`）。
- 内部包不要写版本号引用，一律 `workspace:*`。
- `apps/` = 可独立运行/部署的应用，`packages/` = 可复用库，页面属于应用内部的 `src/`。新增独立应用才在 `apps/` 下建目录。

## 相关知识

- 暂无所内其他相关笔记。

## 外部来源

- 由本机实践搭建得出（pnpm 12.5.1 / Node 24 / Vite 8 / React 19 / Turbo 2.11.2）。
- Turborepo 官方 schema：https://turbo.build/schema.json
