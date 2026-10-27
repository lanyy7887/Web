---
title: 01-TypeScript 入门：为什么现在必须学
publish: true
---

<ArticleTopAd></ArticleTopAd>

> 本章为 2026 年增补章节。原教程（04-JavaScript基础 ~ 13-React基础）以 2018 年前后的技术栈写就，未覆盖 TypeScript。而今天的实际项目中，TypeScript 已是默认选项，不是加分项。

## 一、TypeScript 是什么

TypeScript 是 JavaScript 的**超集**：任何合法的 `.js` 文件，改名为 `.ts` 后大概率仍然是合法的 `.ts` 文件（少数例外见下文）。它在 JS 之上加了一层**静态类型系统**，由编译器 `tsc` 剥离类型信息后，产出可运行的普通 JavaScript。

一句话理解：**TS 不改变运行时，只改变写代码时你能得到什么。**类型检查发生在编译期，产物里没有任何类型残留。

```ts
// 输入（.ts）
const add = (a: number, b: number): number => a + b;

// 输出（.js）——类型全没了
const add = (a, b) => a + b;
```

## 二、为什么 2026 年必须学

这不是"更高级"，而是**行业默认值已经变了**：

| 场景 | 现状 |
|---|---|
| Vue 3 / React 18+ 官方脚手架 | `npm create vue@latest`、`create-next-app` 默认询问是否启用 TS，Vue 3 的 `<script setup lang="ts">` 是主流写法 |
| 开源库 | 绝大多数新库直接以 TS 编写，类型定义 `.d.ts` 内置于包内，不再需要 `@types/xxx` |
| API 边界 | OpenAPI / GraphQL / tRPC 都能生成类型，前后端类型打通 |
| AI 编程助手 | 提供类型上下文后，生成代码的准确率显著高于裸 JS —— 类型等于给模型的规格说明 |
| 招聘 | 一线岗位几乎默认要求 TS |

一个容易被忽略的点：**TS 的真正价值不在"防写错"，而在"重构敢动"。**裸 JS 项目改一个字段名，要靠全局搜索和祈祷；TS 项目会让编译器把所有不一致的地方一次性指出来。

## 三、最小可用配置

不需要一上来就啃完整 tsc 文档。一个现代项目的最小 `tsconfig.json`：

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "verbatimModuleSyntax": true,
    "skipLibCheck": true,
    "noEmit": true
  }
}
```

逐项说明：

- `target: ES2022`：产物语法级别。现代浏览器与 Node 18+ 均可直接执行，不必降到 ES5。
  > 注意区分：`target` 是**产物语法**的下限，与你的**构建工具**所要求的 Node 版本是两件事。例如 Vite 8 要求 Node 20.19+ / 22.12+ 才能运行，但它产出的代码仍可以是 ES2022。
- `moduleResolution: "bundler"`：配合 Vite / esbuild 时代的解析规则，支持 `exports` 字段、无扩展名导入。
- `strict: true`：一次性开启 `strictNullChecks`、`noImplicitAny` 等一整套严格模式。**新项目务必打开**，中途打开代价极高。
- `noUncheckedIndexedAccess`：`arr[0]` 的类型变成 `T | undefined`，强制处理越界。严格但值得。
- `verbatimModuleSyntax`：要求类型导入必须写 `import type`，避免打包器把类型当值保留。
- `noEmit: true`：只做类型检查，产物交给 Vite / esbuild 生成（它们更快）。

运行类型检查：

```bash
npx tsc --noEmit
```

> **版本提示（2026）**：TypeScript 7 起编译器已用 Go 重写（项目代号 Corsa），**类型系统语义与 `tsconfig` 配置完全兼容**，但你学到的东西一句都不用改；变化只在**怎么跑**。
>
> - TypeScript 5.x / 6.x：`npx tsc --noEmit`
> - TypeScript 7+：改用原生二进制 `tsgo`（`npx tsgo --noEmit`），速度提升约 9–10 倍，内存占用下降约 60–70%
>
> 迁移期建议两个都跑一遍，按**诊断码**（如 `TS2322`）而非错误文案比对结果——报错码不变，文案会变。另外 TS7 不再提供 `ts.createProgram()` 这类编译器 API，依赖它做代码生成的工具（如 ts-morph）仍需留在旧版 `typescript` 上。

## 四、核心概念速览

### 1. 基础类型与类型推断

```ts
const n: number = 1;
const s: string = 'abc';
const b: boolean = true;
const arr: number[] = [1, 2, 3];
const tuple: [string, number] = ['age', 18];
const obj: { id: number; name?: string } = { id: 1 };
```

绝大多数情况下**不需要手写类型**，TS 的推断已经够用：

```ts
const n = 1;            // 推断为 1（字面量类型）
let m = 1;              // 推断为 number
const arr = [1, 2, 3];  // 推断为 number[]
```

### 2. 联合类型与交叉类型

```ts
type Status = 'idle' | 'loading' | 'success' | 'error';  // 字面量联合，替代枚举

type User = { id: number } & { name: string };            // 交叉：同时具备
```

字面量联合比 `enum` 更好用：产物零开销、可直接与字符串比较、易于序列化。

### 3. interface 与 type 如何选

| 维度 | interface | type |
|---|---|---|
| 声明合并 | ✅ 可重复声明自动合并 | ❌ 不可 |
| 联合 / 交叉 / 条件类型 | ❌ | ✅ |
| 性能 | 略优（缓存友好） | 稍慢 |
| 惯例 | 定义对象结构、对外 API | 组合类型、工具类型 |

**实践约定**：对外暴露的对象结构用 `interface`，其余类型运算用 `type`。不必纠结，两种都能覆盖 95% 场景。

### 4. 泛型

```ts
function first<T>(list: T[]): T | undefined {
  return list[0];
}

// 常见约束写法
function pick<T, K extends keyof T>(obj: T, keys: K[]): Pick<T, K> {
  const result = {} as Pick<T, K>;
  for (const k of keys) result[k] = obj[k];
  return result;
}
```

泛型的意义是**让类型跟着数据走**，而不是到处 `any`。

### 5. 类型收窄

```ts
function format(v: string | number | null): string {
  if (v === null) return '-';            // null 分支被排除
  if (typeof v === 'number') return v.toFixed(2);  // number 分支
  return v.trim();                        // 只剩 string
}
```

`typeof`、`in`、`instanceof`、`Array.isArray`、判别联合（discriminated union）都能触发收窄。判别联合是最常用的模式：

```ts
type Shape =
  | { kind: 'circle'; r: number }
  | { kind: 'rect'; w: number; h: number };

function area(s: Shape): number {
  switch (s.kind) {
    case 'circle': return Math.PI * s.r ** 2;
    case 'rect':   return s.w * s.h;   // 漏写 case 时会有编译错误
  }
}
```

### 6. 常用工具类型

```ts
interface User { id: number; name: string; email?: string }

type PartialUser  = Partial<User>;            // 全部可选
type RequiredUser = Required<User>;           // 全部必填
type PickedUser   = Pick<User, 'id' | 'name'>;
type OmittedUser  = Omit<User, 'email'>;
type ReadonlyUser = Readonly<User>;
type UserMap      = Record<string, User>;
type NameOrNull   = User['name'] | null;      // 索引访问
```

这些是日常写得最多的类型，比手写重复的 interface 强得多。

## 五、常见坑

### 1. `any` 会传染

`any` 一旦进入链路，下游全部失守。用 `unknown` 代替：

```ts
// ❌ 丢了类型保护
function parseBad(json: string): any {
  return JSON.parse(json);
}

// ✅ 强制使用方先收窄
function parseGood(json: string): unknown {
  return JSON.parse(json);
}
const data = parseGood('{"id":1}');
if (typeof data === 'object' && data !== null && 'id' in data) {
  // 此处才可安全访问
}
```

项目里建议开启 ESLint 规则 `@typescript-eslint/no-explicit-any`，把 `any` 控制在极少数边界处。

### 2. 类型断言不是类型转换

```ts
const el = document.querySelector('#app') as HTMLDivElement;  // 断言：我保证它是
```

`as` 只是**关闭检查**，运行时不会转换、失败也不会报错。能用收窄就别用断言；能用断言就别用 `as any`。

### 3. `?.` 和 `??` 是 ES2020 语法，不是 TS 语法

原教程（04 章、05 章）没有覆盖这两个，但它们极其常用：

```ts
const name = user?.profile?.name ?? '匿名';  // ?. 短路，?? 只对 null/undefined 生效
```

注意 `??` 与 `||` 的区别：`0 || 'default'` 得到 `'default'`，`0 ?? 'default'` 得到 `0`。数值和布尔场景必须用 `??`。

### 4. 类型只在编译期存在

```ts
interface A { x: number }
console.log(A);  // ❌ 报错：A 只表示类型，运行时不存在
```

要携带运行时信息，得用 `enum`（会产出对象）、`const` 对象、或 `class`。

## 六、类型体操要不要学

**先用好基础，再碰高级类型。**以下由浅入深的判断标准：

- 会用 `Partial` / `Pick` / `Omit` / `Record`：**必须**
- 会写泛型函数与泛型约束：**必须**
- 会写条件类型 `T extends U ? X : Y`、`infer`：**进阶，按需**
- 能把类型递归到嵌套层（如深层的 `DeepReadonly`）：**很少用到**

社区里大量"类型体操"内容属于炫技，实际项目收益低。真正高价值的是：**给 API 响应定义准确的结构**、**给组件 props 定义准确的约束**。

## 七、渐进迁移：已有 JS 项目怎么切入

不用推倒重来。推荐路径：

1. **先加配置**：`tsconfig.json` 中设 `"allowJs": true`、`"checkJs": false`，让 `.ts` 与 `.js` 共存。
2. **新文件一律 `.ts`**：老文件不动，新代码天然带类型。
3. **从边界开始改造**：先给 API 层、工具函数、状态定义加类型，收益最高。
4. **逐步打开严格项**：先 `strict: false` + `noImplicitAny: false`，再逐个文件修，最后统一开 `strict`。
5. **CI 里加门禁**：`tsc --noEmit` 必须通过，防止回退。

## 八、小结

- TS = JS + 编译期类型系统，运行时零开销。
- 2026 年它是**默认选项**，不是选修课。
- 真正高频的只有：基础类型、联合/交叉、泛型约束、类型收窄、五个工具类型。够用很久。
- `any` 是最大的敌人，`unknown` + 收窄是正解。
- 老项目靠"新增即 TS + 边界先行"渐进迁移。

> 下一章：在 Vue 3 与 React 中具体怎么用类型（见 19、20 章）。

## 参考

- TypeScript 官方手册：<https://www.typescriptlang.org/docs/handbook/intro.html>
- tsconfig 全选项说明：<https://www.typescriptlang.org/tsconfig>