---
title: 01-现代前端工程化：Vite + pnpm + CI
publish: true
---

<ArticleTopAd></ArticleTopAd>

> 本章为 2026 年增补章节。原教程「15-前端工程化」整章仅 5800 余字，以名词罗列为主，未覆盖构建工具、包管理、TypeScript 编译、代码门禁与持续集成。而这部分恰恰是"能独立交付项目"与"只会写页面"的分水岭。

## 一、工程化到底解决什么

一句话：**把"人写的源码"可靠地变成"浏览器能高效加载的产物"，并让这件事在团队里可重复。**

拆开是五件事：

| 环节 | 问题 | 手段 |
|---|---|---|
| 开发 | 改一行要等很久 | 本地开发服务器 + HMR |
| 构建 | 源码不兼容浏览器、体积大 | 打包、转译、压缩、Tree-shaking |
| 依赖 | 版本冲突、安装慢 | 包管理器 + lockfile |
| 质量 | 风格不一、类型错误、无测试 | Lint、类型检查、单测 |
| 交付 | 手动发布易错 | CI/CD 流水线 |

## 二、包管理器：pnpm

### 1. npm / yarn / pnpm 的区别

| | npm | yarn (classic) | pnpm |
|---|---|---|---|
| 依赖存储 | 每个项目各自复制 | 各自复制 | **全局内容寻址仓库 + 硬链接** |
| 磁盘占用 | 高 | 高 | **极低**（多项目共享） |
| 幽灵依赖 | 有 | 有 | **无**（严格 node_modules 结构） |
| 安装速度 | 中 | 中 | **快** |
| monorepo | 有 workspaces | 有 | **workspaces 更成熟** |

**幽灵依赖**是 npm/yarn 的大问题：你 `import` 了某个包，但它只是别人的间接依赖，未在 `package.json` 声明——某天上游升级就炸。pnpm 的严格结构让这种代码直接报错。

### 2. 常用命令

```bash
pnpm install              # 安装（严格按 lockfile）
pnpm add vue              # 生产依赖
pnpm add -D vite          # 开发依赖
pnpm remove vue
pnpm update               # 按范围更新
pnpm why lodash           # 谁引入了它
pnpm dlx create-vue       # 类似 npx，不污染 node_modules
```

### 3. 必须提交 lockfile

`pnpm-lock.yaml` **一定要进 git**。它锁死整棵依赖树的精确版本，保证"我这能跑"等于"构建机也能跑"。

### 4. `.npmrc` 建议

```ini
# 严格 peer 依赖检查，避免版本错配
strict-peer-dependencies=false

# 不用 hoist，保持严格结构（默认就是 false，不要开）
hoist=false
```

国内加速（可选）：

```ini
registry=https://registry.npmmirror.com
```

### 5. monorepo

pnpm 的 `workspaces` 是当前 monorepo 的主流方案：

```yaml
# pnpm-workspace.yaml
packages:
  - 'apps/*'
  - 'packages/*'
```

```bash
pnpm --filter web dev          # 只跑 web 应用
pnpm --filter './packages/*' build
pnpm -r build                  # 递归所有包
```

典型结构：

```
repo/
├─ apps/
│  ├─ web/          # 用户端
│  └─ admin/        # 后台
├─ packages/
│  ├─ ui/           # 组件库
│  ├─ utils/        # 工具函数
│  └─ tsconfig/     # 共享配置
├─ pnpm-workspace.yaml
└─ package.json
```

## 三、构建工具：Vite

### 1. 为什么 Vite 取代了 webpack

webpack 的工作方式：启动时把**全部**源码打包成 bundle，再起 dev server。项目一大，冷启动几分钟。

Vite 的方式：

- **开发**：不打包，直接利用浏览器原生 ESM 按需加载；改动只失效单个模块（真正的 HMR 即时）。预构建仅处理 `node_modules`（用 esbuild）。
- **生产**：用 **Rollup** 打包，输出高度优化。

| 对比项 | webpack | Vite |
|---|---|---|
| 冷启动 | 慢（全量打包） | 快（按需编译） |
| HMR | 秒级 | 毫秒级 |
| 配置复杂度 | 高 | 低 |
| 生态成熟度 | 最成熟 | 已完全成熟 |
| 适用 | 遗留项目、特殊定制 | **新项目默认选** |

webpack 仍然存在于大量存量项目中，需要时会遇到，但新项目没有理由再选它。

### 2. `vite.config.ts`

```ts
import { defineConfig } from 'vite';
import vue from '@vitejs/plugin-vue';
import { fileURLToPath, URL } from 'node:url';

export default defineConfig({
  plugins: [vue()],
  resolve: {
    alias: {
      '@': fileURLToPath(new URL('./src', import.meta.url)),
    },
  },
  server: {
    port: 5173,
    open: true,
    proxy: {
      '/api': {
        target: 'http://localhost:8080',
        changeOrigin: true,
      },
    },
  },
  build: {
    target: 'es2022',
    sourcemap: false,
    chunkSizeWarningLimit: 800,
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['vue', 'vue-router', 'pinia'],
        },
      },
    },
  },
});
```

### 3. 环境变量

Vite 只暴露 `VITE_` 前缀的变量到客户端：

```ini
# .env.development
VITE_API_BASE=http://localhost:8080

# .env.production
VITE_API_BASE=https://api.example.com
```

```ts
const base = import.meta.env.VITE_API_BASE;
if (import.meta.env.DEV) { /* 仅开发 */ }
```

**注意**：`VITE_` 变量会被**打进产物**，别放密钥。

### 4. 常用优化手段

```ts
// ① 按需引入组件库（如 Element Plus 自动导入）
// ② 路由懒加载（最重要，见下）
const User = () => import('@/views/User.vue');

// ③ 依赖预构建显式声明
optimizeDeps: { include: ['lodash-es'] }
```

**懒加载是收益最大的一步**：把首屏不需要的路由/组件拆出去，首屏 JS 能降一大截。

```bash
pnpm build          # 产物到 dist/
pnpm preview        # 本地预览产物
```

用 `rollup-plugin-visualizer` 生成体积分析图，找出真正的体积大头。

## 四、代码质量门禁

### 1. ESLint（新版扁平配置）

ESLint 9 起用 `eslint.config.js`，取代旧的 `.eslintrc`：

```js
// eslint.config.js
import js from '@eslint/js';
import tseslint from 'typescript-eslint';
import pluginVue from 'eslint-plugin-vue';

export default [
  js.configs.recommended,
  ...tseslint.configs.recommended,
  ...pluginVue.configs['flat/recommended'],
  {
    rules: {
      'no-unused-vars': 'off',
      '@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_' }],
      '@typescript-eslint/no-explicit-any': 'warn',
    },
  },
];
```

React 项目额外加 `eslint-plugin-react-hooks` 与 `react-refresh`。

### 2. Prettier

```json
{
  "semi": true,
  "singleQuote": true,
  "printWidth": 100,
  "trailingComma": "all"
}
```

### 3. 提交前自动跑：lint-staged + husky

```json
// package.json
{
  "scripts": {
    "lint": "eslint .",
    "format": "prettier --write .",
    "typecheck": "vue-tsc --noEmit",
    "test": "vitest run"
  },
  "lint-staged": {
    "*.{ts,tsx,vue,js}": ["eslint --fix", "prettier --write"]
  }
}
```

```bash
pnpm add -D husky lint-staged
pnpm exec husky init
echo "pnpm exec lint-staged" > .husky/pre-commit
```

### 4. 提交信息规范：Commitlint

```bash
pnpm add -D @commitlint/cli @commitlint/config-conventional
echo "export default { extends: ['@commitlint/config-conventional'] };" > commitlint.config.js
echo "pnpm exec commitlint --edit \$1" > .husky/commit-msg
```

格式：`feat: 新增登录`、`fix: 修复分页越界`、`chore: 升级依赖`。规范化的提交信息能自动生成 CHANGELOG。

## 五、测试

| 层级 | 工具 | 测什么 |
|---|---|---|
| 单元测试 | **Vitest** | 工具函数、composables、hooks |
| 组件测试 | Vitest + Testing Library | 组件渲染与交互 |
| E2E | **Playwright** | 真实浏览器跑完整流程 |

```ts
// sum.test.ts
import { describe, it, expect } from 'vitest';
import { sum } from './sum';

describe('sum', () => {
  it('累加', () => {
    expect(sum([1, 2, 3])).toBe(6);
  });
});
```

```ts
// e2e/login.spec.ts
import { test, expect } from '@playwright/test';

test('登录流程', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('用户名').fill('admin');
  await page.getByLabel('密码').fill('123456');
  await page.getByRole('button', { name: '登录' }).click();
  await expect(page).toHaveURL('/dashboard');
});
```

**不必追求覆盖率数字**。优先测：纯函数、状态转换逻辑、关键业务流程。UI 细节不必测。

## 六、CI/CD

### GitHub Actions 最小可用流水线

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: pnpm/action-setup@v4
        with:
          version: 9

      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: pnpm

      - run: pnpm install --frozen-lockfile
      - run: pnpm lint
      - run: pnpm typecheck
      - run: pnpm test
      - run: pnpm build
```

要点：

- `--frozen-lockfile`：lockfile 与 `package.json` 不一致时**直接失败**，防止 CI 悄悄装了不同版本。
- 顺序按"快→慢"：lint → typecheck → test → build，快速失败省时间。
- 缓存 `pnpm` 与 `node_modules`，CI 时间能砍一半。

### 部署产物

`pnpm build` 产出的 `dist/` 是**纯静态文件**，可部署到任意静态托管：

| 平台 | 方式 |
|---|---|
| Vercel / Netlify | 连仓库自动构建 |
| 对象存储 + CDN | 上传 `dist/`，配置 SPA 回退 |
| Nginx | 见下 |

**SPA 必须配置回退**，否则刷新子路由 404：

```nginx
location / {
  try_files $uri $uri/ /index.html;
}

location /assets/ {
  expires 1y;
  add_header Cache-Control "public, immutable";
}
```

`index.html` 不要长缓存（否则发版不生效），带 hash 的静态资源可以设一年。

## 七、目录结构参考

```
src/
├─ assets/          # 静态资源（会被处理）
├─ components/      # 通用组件
├─ composables/     # 组合式函数（Vue）
├─ hooks/           # 自定义 Hook（React）
├─ layouts/         # 布局
├─ router/          # 路由
├─ stores/          # 状态
├─ api/             # 接口层
├─ utils/           # 纯函数
├─ views/ 或 pages/ # 页面
├─ types/           # 全局类型
└─ main.ts
```

约定优于配置：团队内一致就好，不必照搬。

## 八、checklist：新项目初始化

- [ ] `pnpm` + lockfile 入库
- [ ] Vite + TS，`tsconfig` 开 `strict`
- [ ] 路径别名 `@` 同时配到 `vite.config` 与 `tsconfig`
- [ ] ESLint 9 扁平配置 + Prettier
- [ ] husky + lint-staged + commitlint
- [ ] Vitest + Playwright
- [ ] GitHub Actions：lint / typecheck / test / build
- [ ] `.env.*` 约定，密钥不写进 `VITE_` 变量
- [ ] Nginx/托管平台配好 SPA 回退与静态资源缓存

## 九、小结

- 包管理用 **pnpm**，lockfile 必入库，monorepo 用 workspaces。
- 构建用 **Vite**：开发靠原生 ESM 按需编译，生产靠 Rollup。
- 质量门禁三层：**Lint → 类型检查 → 测试**，全部进 CI。
- 提交前 lint-staged 自动修，提交信息用 conventional commits。
- 部署是静态产物 + SPA 回退 + 合理的缓存策略。

## 参考

- Vite 中文文档：<https://cn.vite.dev/>
- pnpm 文档：<https://pnpm.io/zh/>
- ESLint 扁平配置：<https://eslint.org/docs/latest/use/configure/configuration-files>
- Playwright：<https://playwright.dev/zh/docs/intro>