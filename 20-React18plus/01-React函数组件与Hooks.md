---
title: 01-React 18+ 函数组件与 Hooks
publish: true
---

<ArticleTopAd></ArticleTopAd>

> 本章为 2026 年增补章节。原教程「13-React基础」（11 篇）写于 React 15 时代，仍在讲 `getDefaultProps`、`getInitialState`、`componentWillMount` 与"绑定 this"。这些 API 早已废弃或移除。**React 现在只有一种主流写法：函数组件 + Hooks。**

## 一、先纠正旧 API

原教程 13 章出现的这些，请直接跳过：

| 旧 API | 状态 | 现在怎么做 |
|---|---|---|
| `getDefaultProps` | React 0.14 时代，早已移除 | `defaultProps` → 现在是**默认参数** |
| `getInitialState` | 早已移除 | `useState` |
| `componentWillMount` | React 16.3 弃用，17 警告，18 移除 | 直接在函数体执行 |
| `componentWillReceiveProps` | 同上 | `useEffect` + 依赖 |
| `componentWillUpdate` | 同上 | `useEffect` |
| `React.createClass` | React 16 移除 | 函数组件 |
| `bind(this)` | 不再需要 | 函数组件无 `this` |

## 二、创建项目

```bash
npm create vite@latest my-app -- --template react-ts
cd my-app && npm install && npm run dev
```

或直接用 Next.js（带路由与服务端渲染）：

```bash
npx create-next-app@latest
```

## 三、函数组件基础

```tsx
interface Props {
  title: string;
  count?: number;
}

// props 默认值用默认参数，不用 defaultProps
export function Card({ title, count = 0 }: Props) {
  return (
    <div>
      <h2>{title}</h2>
      <p>{count}</p>
    </div>
  );
}
```

**JSX 与 Vue 模板的关键差异**：JSX 是表达式，`{}` 内是完整 JavaScript；`class` 要写 `className`；`for` 要写 `htmlFor`；事件是驼峰（`onClick` 而非 `@click`）。

## 四、核心 Hooks

### 1. `useState`

```tsx
const [count, setCount] = useState(0);

setCount(count + 1);        // 普通更新
setCount(c => c + 1);       // 函数式更新：依赖旧值时必须这样写

// 对象/数组要新建引用，不能就地修改
const [user, setUser] = useState({ name: 'a' });
setUser(u => ({ ...u, name: 'b' }));     // ✅
user.name = 'b';                         // ❌ 不会触发重渲染
```

**惰性初始化**：开销大的初始值传函数。

```tsx
const [data] = useState(() => computeExpensive());
```

### 2. `useEffect`

`useEffect` 是**副作用**的容器（请求、订阅、DOM 操作、定时器）——不是生命周期的搬运工。

```tsx
useEffect(() => {
  // 副作用
  const timer = setInterval(tick, 1000);

  // 清理函数：组件卸载或依赖变化前执行
  return () => clearInterval(timer);
}, []);  // 依赖数组
```

三种形态：

| 依赖数组 | 时机 |
|---|---|
| `[]` | 仅挂载后 + 卸载前（相当于 `mounted` / `unmounted`） |
| `[a, b]` | 挂载后，且 `a`、`b` 变化后 |
| 省略 | 每次渲染后（极少用） |

**最常见的坑**：漏依赖。ESLint 插件 `eslint-plugin-react-hooks` 会提示，**不要关掉这条规则**。

```tsx
// ❌ 闭包捕获了旧的 userId
useEffect(() => {
  fetchUser(userId);
}, []);            // userId 变了但不会重跑

// ✅
useEffect(() => {
  fetchUser(userId);
}, [userId]);
```

### 3. 数据请求的正常写法

```tsx
const [data, setData] = useState<User | null>(null);
const [loading, setLoading] = useState(false);
const [error, setError] = useState<Error | null>(null);

useEffect(() => {
  const ctrl = new AbortController();
  let aborted = false;          // 标记本次 effect 是否已被清理

  setLoading(true);
  setError(null);

  fetch(`/api/user/${id}`, { signal: ctrl.signal })
    .then(r => r.ok ? r.json() : Promise.reject(new Error(`HTTP ${r.status}`)))
    .then(res => { if (!aborted) setData(res); })
    .catch(e => { if (!aborted) setError(e); })
    .finally(() => { if (!aborted) setLoading(false); });   // 关键：别让旧请求关掉新请求的 loading

  return () => {
    aborted = true;      // 先标记，再中断
    ctrl.abort();
  };
}, [id]);
```

**为什么需要 `aborted` 标志**：`abort()` 只会让请求以 `AbortError` 失败，`finally` 依然会执行。当 `id` 快速切换时，旧请求的 `finally` 会把**新请求**正在展示的 loading 状态误关掉，界面看起来"加载完了但没数据"。`AbortController` 负责取消网络请求，`aborted` 标志负责拦截迟到的状态更新，两者缺一不可。

### 4. `useRef`

两种用途，别混用：

```tsx
// ① 引用 DOM
const inputRef = useRef<HTMLInputElement>(null);
useEffect(() => inputRef.current?.focus(), []);
// <input ref={inputRef} />

// ② 存储跨渲染的"可变值"，变化不触发渲染
const timerId = useRef<number>();
timerId.current = window.setInterval(tick, 1000);
```

### 5. `useMemo` 与 `useCallback`

```tsx
const sorted = useMemo(
  () => [...list].sort((a, b) => a.age - b.age),
  [list]
);

const onSelect = useCallback((id: number) => {
  setSelected(id);
}, []);
```

**不要滥用**。`useMemo` 有成本，只有在①计算确实昂贵、或②引用需要稳定（传给 `memo` 组件、作为 `useEffect` 依赖）时才用。React 编译器（React Compiler）成熟后，手写这些会进一步减少。

### 6. `useReducer`

状态逻辑复杂、或多字段联动时替代多个 `useState`：

```tsx
type State = { status: 'idle' | 'loading' | 'ok'; data: any[] };
type Action = { type: 'start' } | { type: 'ok'; data: any[] };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'start': return { ...state, status: 'loading' };
    case 'ok':    return { status: 'ok', data: action.data };
  }
}

const [state, dispatch] = useReducer(reducer, { status: 'idle', data: [] });
dispatch({ type: 'start' });
```

### 7. `useContext`

```tsx
const ThemeContext = createContext<'light' | 'dark'>('light');

// 提供
<ThemeContext.Provider value="dark">{children}</ThemeContext.Provider>

// 消费
const theme = useContext(ThemeContext);
```

`useContext` 的值变化会导致**所有消费者重渲染**，大对象建议配合 `useMemo` 或拆成多个 Context。

## 五、自定义 Hook

逻辑复用的正确方式（对比 Vue 的 composables）：

```tsx
export function useFetch<T>(url: string) {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    const ctrl = new AbortController();
    let aborted = false;

    setLoading(true);
    setError(null);

    fetch(url, { signal: ctrl.signal })
      .then(r => r.ok ? r.json() : Promise.reject(new Error(`HTTP ${r.status}`)))
      .then(res => { if (!aborted) setData(res); })
      .catch(e => { if (!aborted) setError(e); })
      .finally(() => { if (!aborted) setLoading(false); });

    return () => {
      aborted = true;
      ctrl.abort();
    };
  }, [url]);

  return { data, loading, error };
}
```

**规范**：函数名以 `use` 开头，内部可调用其他 Hook，Hook 调用必须在顶层（不能放在 `if` / `for` / 回调里）。

## 六、性能优化

### 1. `React.memo`

```tsx
const Row = memo(function Row({ item, onSelect }: RowProps) {
  return <div onClick={() => onSelect(item.id)}>{item.name}</div>;
});
```

只做**浅比较** props；配合 `useCallback` 传给子组件的函数才有效。

### 2. 列表 key

```tsx
// ❌ 用索引：增删时状态会错位
items.map((item, i) => <Row key={i} />)

// ✅ 用稳定 id
items.map(item => <Row key={item.id} item={item} />)
```

### 3. 代码分割

```tsx
const HeavyChart = lazy(() => import('./HeavyChart'));

<Suspense fallback={<Spinner />}>
  <HeavyChart />
</Suspense>
```

`Suspense` 也可用于数据获取（配合 RSC 或 TanStack Query 的 suspense 模式）。

### 4. 分析工具

```tsx
import { Profiler } from 'react';
<Profiler id="App" onRender={(id, phase, actualDuration) => console.log(id, phase, actualDuration)}>
  <App />
</Profiler>
```

## 七、状态管理怎么选（2026）

| 规模 | 方案 |
|---|---|
| 组件内部 | `useState` / `useReducer` |
| 跨组件少层级 | `useContext` |
| 全局应用状态 | **Zustand**（轻、无 Provider）或 **Jotai**（原子化） |
| 服务端数据（API 缓存） | **TanStack Query** 或 **SWR** |
| 大型企业级 | Redux Toolkit（仍在维护，但新项目多用 Zustand） |

**最重要的一条认知**：**服务端状态和客户端状态要分开管。**把 API 数据塞进 Redux 是过去十年的常见错误——它本质是缓存，该交给 TanStack Query。

```tsx
// TanStack Query 示例
const { data, isLoading, error } = useQuery({
  queryKey: ['user', id],
  queryFn: () => fetch(`/api/user/${id}`).then(r => r.json()),
});
```

## 八、React 18 / 19 的新东西

> 版本现状：React 已到 **19.x**。18 与 19 之间**没有破坏性的大改**，下面这些能力是叠加关系，18 项目可平滑升级到 19。

| 特性 | 作用 |
|---|---|
| **并发渲染** | 渲染可中断，交互不被大列表阻塞 |
| `useTransition` | 把非紧急更新降级，保持输入响应 |
| `useDeferredValue` | 延迟某个值的更新 |
| **自动批处理** | 多处 `setState` 自动合并（18 起在 Promise / setTimeout 中也生效） |
| **`useSyncExternalStore`** | 外部 store（Zustand 等）正确订阅 |
| **React 19：Actions** | 表单提交与异步状态内置（`useActionState`、`useFormStatus`） |
| **React 19：`use()`** | 在渲染中读取 Promise / Context |
| **Server Components（RSC）** | 组件在服务端运行，产物无 JS，首屏更小（主要在 Next.js App Router 中使用） |

RSC 是当前最大的方向性变化：**"默认服务端渲染、仅在需要交互处标记 `'use client'`"**。

## 九、现状速查

| 旧写法 | 现在 |
|---|---|
| class 组件 | 函数组件 + Hooks |
| `bind(this)` | 不需要 `this` |
| `getInitialState` | `useState` |
| `componentDidMount` | `useEffect(fn, [])` |
| `componentWillUnmount` | `useEffect` 的 `return` 清理 |
| `componentDidUpdate` | `useEffect(fn, [deps])` |
| `defaultProps` | 默认参数 |
| `setState` | `setX`（不可就地修改对象） |
| `connect()` + Redux | `useSelector` / Zustand |
| `class ErrorBoundary` | 仍主要用 class（ErrorBoundary 这一场景 class 仍是标准解法） |

## 十、小结

- 只写函数组件 + Hooks，class 组件除了 ErrorBoundary 基本不再需要。
- `useEffect` 是副作用容器，**依赖数组要写全**，清理函数是必做项。
- 状态不可变：对象/数组一律新建引用。
- 服务端数据用 TanStack Query，不要塞进全局 store。
- `useMemo` / `useCallback` 按需使用，别默认都加。
- 方向是 RSC + 并发渲染，Next.js App Router 是目前的主入口。

## 参考

- React 官方文档（新版，含 Hooks 全解）：<https://zh-hans.react.dev/>
- 你可能不需要 `useEffect`：<https://zh-hans.react.dev/learn/you-might-not-need-an-effect>
- TanStack Query：<https://tanstack.com/query/latest>
- Zustand：<https://zustand.docs.pmnd.rs/>