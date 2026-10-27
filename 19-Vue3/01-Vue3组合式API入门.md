---
title: 01-Vue 3 组合式 API 入门
publish: true
---

<ArticleTopAd></ArticleTopAd>

> 本章为 2026 年增补章节。原教程「12-Vue基础」（17 篇 / 16.5 万字）基于 **Vue 2 + vue-cli**，其核心概念（`filters`、`beforeCreate`、`this.$refs` 式写法、`Vue.use`）在 Vue 3 中已移除或改写。学 Vue 2 的思想仍有用，但**代码不要照抄**。

## 一、Vue 2 与 Vue 3 的差异定位

先建立正确预期：Vue 3 **不是**小版本升级。

| 维度 | Vue 2 | Vue 3 |
|---|---|---|
| API 风格 | 选项式（Options API） | 选项式 + **组合式（Composition API）** |
| 组件写法 | `export default { data(){}, methods:{} }` | `<script setup>` |
| 响应式 | `Object.defineProperty` | **`Proxy`**（可监听新增/删除属性、数组索引） |
| 过滤器 `filters` | ✅ 支持 | ❌ **已移除** |
| 多根节点 | 必须单根 | ✅ Fragment 支持多根 |
| 打包体积 | ~20KB+ | 支持 Tree-shaking，按需引入更小 |
| 性能 | diff 优化有限 | 编译时优化（静态提升、Patch Flag） |
| 脚手架 | `vue-cli`（已停止维护） | **`create-vue` + Vite** |
| 状态管理 | Vuex | **Pinia**（Vuex 5 已不再推进） |
| 类型支持 | 较弱 | 原生 TS 友好 |

**结论**：新项目用 Vue 3 + `<script setup>` + Vite + Pinia + TypeScript。Vue 2 已于 2023-12-31 停止官方维护。

## 二、创建项目

```bash
npm create vue@latest
```

交互式选择（推荐配置）：

```
✔ Project name: my-app
✔ Add TypeScript? … Yes
✔ Add JSX Support? … No
✔ Add Vue Router? … Yes
✔ Add Pinia? … Yes
✔ Add Vitest? … Yes
✔ Add ESLint? … Yes
✔ Add Prettier? … Yes
```

然后：

```bash
cd my-app
npm install
npm run dev
```

## 三、`<script setup>` 是什么

Vue 3 最重要的书写方式。它把"定义组件"变成"写一个函数"。

### Vue 2 的写法（原教程 12 章）

```js
export default {
  data() {
    return { count: 0 };
  },
  computed: {
    double() { return this.count * 2; }
  },
  methods: {
    inc() { this.count++; }
  }
}
```

### Vue 3 的写法（现在）

```vue
<script setup lang="ts">
import { ref, computed } from 'vue';

const count = ref(0);
const double = computed(() => count.value * 2);

function inc() {
  count.value++;
}
</script>

<template>
  <div>
    <p>{{ count }} / {{ double }}</p>
    <button @click="inc">+1</button>
  </div>
</template>
```

**关键差异**：

- 顶层声明的变量、函数**自动暴露给模板**，不需要 `return`。
- 不需要 `this`。想 `this.count` 的地方，直接写 `count`。
- 逻辑可以按**功能**聚合，而不是被 `data` / `methods` / `computed` 切碎。

## 四、响应式核心：`ref` 与 `reactive`

这是 Vue 3 最需要想清楚的一对。

### `ref` —— 推荐默认使用

```ts
import { ref } from 'vue';

const count = ref(0);        // Ref<number>
const user = ref({ name: 'a' });  // Ref<{ name: string }>

count.value++;               // 在 script 中必须 .value
user.value.name = 'b';
```

```vue
<template>
  <!-- 模板中自动解包，不用 .value -->
  <p>{{ count }} {{ user.name }}</p>
</template>
```

**为什么推荐 `ref`**：可以包装任何类型（原始值、对象、数组、`null`），解构不丢响应性，类型推断稳定。

### `reactive` —— 只适合对象

```ts
import { reactive } from 'vue';

const state = reactive({ count: 0, list: [] as string[] });
state.count++;               // 不需要 .value
```

**坑**：`reactive` 解构会**丢失响应性**。

```ts
const state = reactive({ count: 0 });
const { count } = state;      // ❌ count 是普通数字，不再响应
```

需要解构时用 `toRefs`：

```ts
import { toRefs } from 'vue';
const { count } = toRefs(state);  // ✅ count 是 Ref
```

### 选择建议

| 场景 | 选择 |
|---|---|
| 单个原始值（数字、字符串、布尔） | `ref` |
| 一组相关状态 | `reactive` 或普通对象 + 多个 `ref` |
| 需要整体替换的对象（如 API 响应） | `ref`（`reactive` 替换会丢代理） |
| 在函数间传递 | `ref` |

**实用原则**：拿不准就 `ref`，团队统一更省心。

## 五、计算属性与侦听

### `computed`

```ts
import { ref, computed } from 'vue';

const list = ref([1, 2, 3]);
const total = computed(() => list.value.reduce((a, b) => a + b, 0));

// 可写计算属性
const fullName = computed({
  get: () => `${firstName.value} ${lastName.value}`,
  set: (v) => {
    [firstName.value, lastName.value] = v.split(' ');
  }
});
```

`computed` 有缓存，依赖不变不会重算，优于在模板里直接算。

### `watch` 与 `watchEffect`

```ts
import { ref, watch, watchEffect } from 'vue';

const keyword = ref('');
const page = ref(1);

// 侦听单个源
watch(keyword, (newVal, oldVal) => {
  console.log(newVal, oldVal);
});

// 侦听多个源
watch([keyword, page], ([kw, pg]) => {
  fetchList(kw, pg);
});

// 深层侦听对象
watch(user, (v) => { /* ... */ }, { deep: true });

// 立即执行 + 只需一次
watch(keyword, (v) => { /* ... */ }, { immediate: true, once: true });
```

`watchEffect` 自动收集依赖，适合"有依赖就重跑"的场景：

```ts
watchEffect(() => {
  fetchList(keyword.value, page.value);  // 引用即被追踪
});
```

**选择**：需要新旧值或精确控制时机 → `watch`；逻辑简单、依赖多 → `watchEffect`。

## 六、生命周期

Vue 2 的钩子大多改名了：

| Vue 2 | Vue 3（组合式） |
|---|---|
| `beforeCreate` / `created` | 直接用 `<script setup>` 顶层代码 |
| `beforeMount` | `onBeforeMount` |
| `mounted` | `onMounted` |
| `beforeUpdate` | `onBeforeUpdate` |
| `updated` | `onUpdated` |
| `beforeDestroy` | **`onBeforeUnmount`** |
| `destroyed` | **`onUnmounted`** |
| `activated` / `deactivated` | `onActivated` / `onDeactivated` |

```vue
<script setup lang="ts">
import { onMounted, onUnmounted } from 'vue';

let timer: number | undefined;

onMounted(() => {
  timer = window.setInterval(() => console.log('tick'), 1000);
});

onUnmounted(() => {
  clearInterval(timer);   // 清理是必做项
});
</script>
```

**注意**：`created` 的等价物就是 `<script setup>` 顶层语句，不需要钩子。

## 七、组件通信

### 1. props

```vue
<!-- 子组件 Child.vue -->
<script setup lang="ts">
interface Props {
  title: string;
  count?: number;
  list: string[];
}
const props = withDefaults(defineProps<Props>(), {
  count: 0
});
</script>

<template>
  <h2>{{ props.title }}</h2>
</template>
```

### 2. emit

```vue
<!-- 子组件 -->
<script setup lang="ts">
const emit = defineEmits<{
  change: [value: string];
  submit: [];
}>();

emit('change', 'hello');
</script>
```

父组件：

```vue
<Child @change="onChange" @submit="onSubmit" />
```

### 3. `defineModel` —— Vue 3.4+ 的双向绑定

替代 `v-model` 的手写 `props` + `emit` 套路：

```vue
<!-- 子组件 -->
<script setup lang="ts">
const modelValue = defineModel<string>({ required: true });
</script>

<template>
  <input v-model="modelValue" />
</template>
```

父组件直接 `<Child v-model="text" />`。

### 4. `defineExpose` —— 父组件访问子组件

```vue
<!-- 子组件 -->
<script setup lang="ts">
import { ref } from 'vue';
const inputRef = ref<HTMLInputElement>();

function focus() {
  inputRef.value?.focus();
}

defineExpose({ focus });   // 显式导出
</script>
```

```vue
<!-- 父组件 -->
<script setup lang="ts">
const childRef = ref();
onMounted(() => childRef.value?.focus());
</script>

<template>
  <Child ref="childRef" />
</template>
```

**与 Vue 2 的差异**：`<script setup>` 默认**完全封闭**，不 `defineExpose` 的话父组件拿不到任何东西。这比 Vue 2 的 `this.$refs` 全开放更安全。

### 5. provide / inject —— 跨层传递

```ts
// 祖先
import { provide, ref } from 'vue';
const theme = ref('dark');
provide('theme', theme);

// 后代（任意深度）
import { inject, ref } from 'vue';
const theme = inject<Ref<string>>('theme', ref('light'));
```

### 6. 状态管理用 Pinia

跨组件、跨路由共享状态，不要靠层层 props 或 event bus，用 Pinia（见下节）。

## 八、Pinia 替代 Vuex

Vuex 在 Vue 3 时代被 Pinia 取代。Pinia 无 mutations、类型推断好、支持组合式写法。

```ts
// stores/counter.ts
import { defineStore } from 'pinia';
import { ref, computed } from 'vue';

export const useCounterStore = defineStore('counter', () => {
  const count = ref(0);
  const double = computed(() => count.value * 2);

  function inc() {
    count.value++;
  }

  async function load() {
    const res = await fetch('/api/count');
    count.value = (await res.json()).count;
  }

  return { count, double, inc, load };
});
```

使用：

```vue
<script setup lang="ts">
import { storeToRefs } from 'pinia';
import { useCounterStore } from '@/stores/counter';

const store = useCounterStore();
const { count, double } = storeToRefs(store);  // 解构状态要用 storeToRefs

store.inc();
</script>
```

**坑**：直接 `const { count } = store` 会丢失响应性，必须 `storeToRefs`。方法可以正常解构。

## 九、路由 Vue Router 4

```ts
// router/index.ts
import { createRouter, createWebHistory } from 'vue-router';

const router = createRouter({
  history: createWebHistory(),
  routes: [
    { path: '/', component: () => import('@/views/Home.vue') },
    { path: '/user/:id', component: () => import('@/views/User.vue'), props: true },
  ]
});

export default router;
```

```vue
<script setup lang="ts">
import { useRoute, useRouter } from 'vue-router';

const route = useRoute();
const router = useRouter();

console.log(route.params.id);   // 响应式
router.push('/');
</script>

<template>
  <RouterView />
</template>
```

**与 Vue 2 的差异**：不再有 `this.$router` / `this.$route`，改用 `useRouter()` / `useRoute()`。

## 十、组合式函数（Composables）

这是 Vue 3 相对 Vue 2 最大的**架构收益**：逻辑可以提取复用。

```ts
// composables/useFetch.ts
import { ref, shallowRef } from 'vue';

export function useFetch<T>(url: string) {
  const data = shallowRef<T | null>(null);
  const error = ref<Error | null>(null);
  const loading = ref(false);

  async function execute() {
    loading.value = true;
    error.value = null;
    try {
      const res = await fetch(url);
      if (!res.ok) throw new Error(`HTTP ${res.status}`);
      data.value = await res.json();
    } catch (e) {
      error.value = e as Error;
    } finally {
      loading.value = false;
    }
  }

  return { data, error, loading, execute };
}
```

组件里：

```vue
<script setup lang="ts">
const { data, loading, error, execute } = useFetch<User[]>('/api/users');
onMounted(execute);
</script>
```

对比 Vue 2：同样的复用要写 mixin，而 mixin 存在**命名冲突、来源不清、隐式依赖**三大问题。Composables 彻底解决了这些。

### 编写规范

- 文件名以 `use` 开头（`useUser.ts`）
- 内部可用 `ref` / `computed` / 生命周期钩子
- 返回对象，便于按需解构
- 注意清理副作用（`onUnmounted` 中移除监听）

## 十一、迁移对照速查

| Vue 2 写法 | Vue 3 等价 |
|---|---|
| `data()` | `const x = ref()` |
| `this.xxx` | 直接 `xxx`（`.value` 视场景） |
| `computed: {}` | `computed(() => ...)` |
| `watch: {}` | `watch(source, cb)` |
| `methods: {}` | 普通函数 |
| `mounted()` | `onMounted()` |
| `beforeDestroy` | `onBeforeUnmount` |
| `filters` | `computed` 或函数调用 |
| `Vue.prototype.$x` | `app.config.globalProperties` 或 `provide/inject` |
| `this.$refs.x` | `ref` + **`defineExpose`** |
| `Vuex` | `Pinia` |
| `this.$router` | `useRouter()` |
| `v-model` 手写 | `defineModel` |
| `event bus` | `provide/inject` 或 Pinia |

## 十二、小结

- 新项目：Vue 3 + `<script setup>` + Vite + Pinia + TS。
- 默认用 `ref`；`reactive` 只用于对象，且别解构。
- `filters` 已删除，用 `computed` 替代。
- `<script setup>` 默认封闭，要用 `defineExpose` 显式暴露。
- Composables 是 Vue 3 最有价值的架构能力，优先于 mixin。
- Vue 2 已于 2023 年底停止维护，存量项目应尽快迁移。

## 参考

- Vue 3 官方文档：<https://cn.vuejs.org/guide/introduction.html>
- 组合式 API FAQ：<https://cn.vuejs.org/guide/extras/composition-api-faq.html>
- Pinia：<https://pinia.vuejs.org/zh/>
- Vue Router 4：<https://router.vuejs.org/zh/>