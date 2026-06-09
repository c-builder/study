# React 19 新特性

## 学习目标

- 读懂 React 19 的核心变化：`use`、Actions、Server Components、Compiler
- 能在项目中落地表单 Actions、乐观更新、异步数据读取
- 理解从 React 18 升级的 breaking change 与迁移路径
- 知道每个新特性「解决什么旧痛点、怎么验证生效」

## 为什么需要

React 18 时代，表单提交要手写 `loading/error/pending` 三件套，数据请求要 `useEffect + useState`，性能要手动 `useMemo/useCallback`。React 19 用 **Actions + use + Compiler** 系统性消除这些样板。

> 核心心法：**React 19 的主线是「少写胶水代码」**——异步状态、表单、缓存、memo 都交给框架。

---

## 一、心智模型：旧痛点 → React 19 方案

| React 18 你要手写的 | React 19 方案 | 收益 |
|--------------------|--------------|------|
| `loading/error/data` 三状态 | Actions + `useActionState` | 自动管理 pending/error |
| `useEffect` 里 fetch | `use(promise)` + Suspense | 声明式读取 |
| 提交后手动回滚 UI | `useOptimistic` | 乐观更新自动回滚 |
| `forwardRef` 包裹 | `ref` 作为普通 prop | 少一层包装 |
| `useMemo/useCallback` 满屏 | React Compiler | 自动 memo |
| `<Helmet>` 管 title | 原生 `<title>/<meta>` | 内置 SSR 友好 |

---

## 二、use()：读取 Promise 与 Context

### 读懂

`use` 可以在**渲染期间**读取 Promise（配合 Suspense）或 Context，且**允许在条件语句中调用**（与其他 Hook 不同）。

### 用法

```jsx
import { use, Suspense } from 'react';

function Comments({ commentsPromise }) {
  // promise resolve 前组件挂起，由最近的 Suspense 显示 fallback
  const comments = use(commentsPromise);
  return comments.map(c => <p key={c.id}>{c.text}</p>);
}

function Page({ commentsPromise }) {
  return (
    <Suspense fallback={<Spinner />}>
      <Comments commentsPromise={commentsPromise} />
    </Suspense>
  );
}
```

```jsx
// 条件读取 Context（旧 useContext 不允许）
function Heading({ children }) {
  if (children == null) return null;
  const theme = use(ThemeContext); // 提前 return 之后仍可调用
  return <h1 className={theme}>{children}</h1>;
}
```

> 注意：传给 `use` 的 promise 应来自缓存或 Server Component，**不要在 render 里 new Promise**（每次渲染都会变）。

---

## 三、Actions 与表单：消除提交样板

### 读懂

`<form action={fn}>` 接受一个函数；React 自动管理提交的 pending 状态、错误、并发，提交期间表单进入 transition。

### 3.1 useActionState：表单状态一把梭

```jsx
import { useActionState } from 'react';

function UpdateName() {
  const [error, submitAction, isPending] = useActionState(
    async (prevState, formData) => {
      const name = formData.get('name');
      const err = await updateName(name);
      if (err) return err;       // 返回值成为新 state
      redirect('/profile');
      return null;
    },
    null // 初始 state
  );

  return (
    <form action={submitAction}>
      <input name="name" />
      <button disabled={isPending}>更新</button>
      {error && <p className="error">{error}</p>}
    </form>
  );
}
```

**对比 React 18：** 省去手写 `const [pending,setPending]`、`try/catch`、`disabled` 联动。

### 3.2 useFormStatus：子组件读取父表单状态

```jsx
import { useFormStatus } from 'react-dom';

// 通用提交按钮，无需 props 透传 pending
function SubmitButton() {
  const { pending } = useFormStatus();
  return <button disabled={pending}>{pending ? '提交中…' : '提交'}</button>;
}
```

**验证：** 提交时按钮自动禁用并显示「提交中」，无需父组件传 prop。

---

## 四、useOptimistic：乐观更新

### 读懂

提交异步操作时，先**立即**显示预期结果；若失败，React 自动回滚到真实 state。

```jsx
import { useOptimistic, useState } from 'react';

function MessageList({ messages, sendMessage }) {
  const [optimisticMessages, addOptimistic] = useOptimistic(
    messages,
    (state, newMsg) => [...state, { text: newMsg, sending: true }]
  );

  async function action(formData) {
    const text = formData.get('text');
    addOptimistic(text);          // 立即上屏（sending 态）
    await sendMessage(text);       // 真正发送，完成后用真实 messages
  }

  return (
    <>
      {optimisticMessages.map((m, i) => (
        <div key={i}>{m.text} {m.sending && '(发送中…)'}</div>
      ))}
      <form action={action}><input name="text" /></form>
    </>
  );
}
```

**验证：** 弱网下消息秒上屏；接口失败后该条自动消失（回滚）。

---

## 五、Server Components 与 Server Actions

### 5.1 RSC（Server Components）

默认在**服务端**渲染、零客户端 JS bundle；交互组件用 `'use client'` 标记。

```jsx
// 默认 Server Component — 直接 async 取数，不进客户端 bundle
async function ProductPage({ id }) {
  const product = await db.product.findById(id); // 直连数据库
  return (
    <article>
      <h1>{product.name}</h1>
      <AddToCart id={id} /> {/* 客户端组件 */}
    </article>
  );
}
```

```jsx
'use client';
export function AddToCart({ id }) {
  return <button onClick={() => addToCart(id)}>加入购物车</button>;
}
```

### 5.2 Server Actions（'use server'）

```jsx
// actions.ts
'use server';
export async function createTodo(formData) {
  await db.todo.create({ text: formData.get('text') });
  revalidatePath('/todos');
}
```

```jsx
import { createTodo } from './actions';
// 客户端/服务端组件均可直接把 server action 作为 form action
<form action={createTodo}><input name="text" /><button>添加</button></form>
```

> RSC/Server Actions 需框架支持（Next.js App Router 等），纯 CSR 项目用不到。

---

## 六、其他实用改进

### 6.1 ref 作为 prop（告别 forwardRef）

```jsx
// React 19：函数组件直接接收 ref
function Input({ ref, ...props }) {
  return <input ref={ref} {...props} />;
}
// 旧写法 forwardRef 仍兼容，但不再必需
```

### 6.2 文档元数据（原生支持）

```jsx
function Article({ post }) {
  return (
    <article>
      <title>{post.title}</title>
      <meta name="description" content={post.excerpt} />
      <link rel="canonical" href={post.url} />
      <h1>{post.title}</h1>
    </article>
  );
}
// React 自动把这些标签提升到 <head>，SSR 友好
```

### 6.3 资源预加载 API

```jsx
import { preload, preinit, prefetchDNS } from 'react-dom';

preinit('https://cdn.example.com/style.css', { as: 'style' });
preload('https://cdn.example.com/font.woff2', { as: 'font' });
prefetchDNS('https://api.example.com');
```

### 6.4 Context 简写

```jsx
// React 19：<Context> 直接作为 Provider
<ThemeContext value="dark">{children}</ThemeContext>
// 旧：<ThemeContext.Provider value="dark">
```

### 6.5 cleanup for refs

```jsx
<input ref={(node) => {
  // 挂载逻辑
  return () => { /* 卸载清理，类似 useEffect cleanup */ };
}} />
```

---

## 七、React Compiler（自动优化）

### 读懂

编译器在**构建期**自动插入 memo，免去手写 `useMemo/useCallback/React.memo`。

```bash
npm install -D babel-plugin-react-compiler
```

```javascript
// babel.config.js
module.exports = {
  plugins: ['babel-plugin-react-compiler'],
};
```

```jsx
// 写普通代码，编译器自动记忆化
function Cart({ items }) {
  const total = items.reduce((s, i) => s + i.price, 0); // 自动缓存
  return <div>{total}</div>;
}
```

**验证：** React DevTools 中组件标记 ✨「Memo ✨」徽章；Profiler 中无关更新不再 re-render。

> 渐进采用：可先对部分目录开启；前提是代码遵守 Rules of React（无副作用 render、不 mutate props/state）。

---

## 八、从 React 18 升级（迁移要点）

```bash
npm install react@19 react-dom@19
```

| 变更 | 处理 |
|------|------|
| 移除 `propTypes`/`defaultProps`（函数组件） | 用 TS 类型 + 默认参数 |
| 移除 legacy Context (`contextTypes`) | 用 `createContext` |
| `ReactDOM.render` 已移除 | 用 `createRoot`（18 起） |
| `ref` 回调返回值被当 cleanup | 不要隐式返回非清理值 |
| 字符串 ref 彻底移除 | 用回调/`useRef` |

**升级步骤：**

1. 先升到 18.3（带废弃警告）→ 清理所有 warning
2. 升 19，跑测试 + 类型检查
3. 用 codemod：`npx codemod@latest react/19/migration-recipe`
4. 验证：CI 绿、关键路径 e2e 通过

---

## 排查实战

### 案例 A：表单重复提交

- **现象：** 快速点两次提交两条
- **原因：** 仍用手写 onClick + fetch
- **修复：** 改 `<form action={submitAction}>` + `useFormStatus` 禁用按钮
- **验证：** pending 期间按钮禁用，无重复

### 案例 B：use(promise) 无限挂起

- **原因：** render 里 `use(fetch(...))`，每次渲染新 promise
- **修复：** promise 来自缓存/Query/Server Component
- **验证：** Suspense 正常 resolve 一次

### 案例 C：开启 Compiler 后行为异常

- **原因：** render 中 mutate 了 props 或有副作用
- **修复：** 遵守 Rules of React；用 eslint-plugin-react-compiler 排查
- **验证：** lint 0 报错，行为恢复

---

## 常见误区与最佳实践

| 误区 | 正确理解 |
|------|---------|
| "Server Components 是 SSR" | RSC 是「服务端组件零客户端 JS」，比 SSR 更进一步 |
| "use 能替代所有 Hook" | use 仅读 Promise/Context，不管理状态 |
| "有 Compiler 就删所有 useMemo" | 渐进迁移，遵守规则后再删 |
| "Actions 只能在 Server" | 客户端 `<form action>` 也支持，Server Action 需框架 |
| "升级 19 无痛" | 先清 18.3 warning，跑 codemod |

**可执行清单：**

- [ ] 新表单用 Actions + `useActionState` + `useFormStatus`
- [ ] 异步交互加 `useOptimistic` 提升体感
- [ ] 升级先到 18.3 清 warning 再上 19
- [ ] 评估接入 React Compiler（先小范围）
- [ ] 新组件不再写 `forwardRef`，`ref` 直接当 prop

## 小结

- React 19 主线：用 Actions/`use`/Compiler 消除胶水代码
- 表单三件套被 `useActionState` + `useFormStatus` 取代
- `useOptimistic` 让乐观更新与回滚开箱即用
- RSC/Server Actions 需框架支持，CSR 项目按需
- 升级靠 codemod + 先清 18.3 warning

## 延伸阅读

- [React 19 发布说明](https://react.dev/blog/2024/12/05/react-19)
- [use API](https://react.dev/reference/react/use)
- [useActionState](https://react.dev/reference/react/useActionState) / [useOptimistic](https://react.dev/reference/react/useOptimistic)
- [React Compiler](https://react.dev/learn/react-compiler)
- [Server Components](https://react.dev/reference/rsc/server-components)
