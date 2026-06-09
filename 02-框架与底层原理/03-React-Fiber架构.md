# React Fiber 架构

## 学习目标

- 读懂 Fiber 解决「长任务阻塞输入」的问题
- 会用 React Profiler 定位慢渲染，并应用 useTransition/memo
- 理解 useEffect vs useLayoutEffect 的选用

## 为什么需要

大列表过滤输入卡顿、Tab 切换掉帧——需要知道 React **何时渲染、能否中断、如何降优先级**。

> 核心心法：**Render 可中断，Commit 不可中断。** 优化要么减少 Render 工作量，要么降低更新优先级。

---

## 一、心智模型

| 用户抱怨 | 对应机制 | 工具/ API |
|---------|---------|-----------|
| 输入卡 | 长 Render 占主线程 | Profiler + useTransition |
| 闪一下 | 双重渲染 Strict Mode | 正常，生产无 |
| 测量 DOM 不准 | useEffect 太晚 | useLayoutEffect |
| 列表慢 | 全树 reconcile | memo + 虚拟列表 |

---

## 二、两阶段（排查用）

```mermaid
flowchart LR
    Render[Render Phase 可中断] --> Commit[Commit Phase 同步写 DOM]
```

- **Render：** 执行组件函数、Diff — Profiler 里 **render** 时间
- **Commit：** 改 DOM、useLayoutEffect、paint、useEffect

---

## 三、Profiler 实操（必会）

1. React DevTools → **Profiler** → 蓝色圆录制
2. 执行卡顿操作（输入搜索、切 Tab）
3. 看 **Commit** 火焰图：哪个组件最宽最红
4. 点击组件 → **Why did this render?**

**常见结论：**

- Parent re-rendered → memo 子组件或状态下放
- Props changed (object/function) → useMemo/useCallback 稳定引用
- Hooks changed → 检查 deps

---

## 四、Concurrent API 落地

```jsx
function Search() {
  const [q, setQ] = useState('');
  const [isPending, startTransition] = useTransition();

  return (
    <>
      <input value={q} onChange={e => {
        setQ(e.target.value);
        startTransition(() => setFiltered(filter(all, e.target.value)));
      }} />
      {isPending && <Spinner />}
      <List items={filtered} />
    </>
  );
}
```

**验证：** Profiler 中 input 的 commit 仍快，List 更新标记为 transition 低优先级。

---

## 五、Effect 选用

| API | 时机 | 用途 |
|-----|------|------|
| useLayoutEffect | paint 前同步 | 测量 DOM、同步改 layout 避免闪 |
| useEffect | paint 后异步 | 请求、订阅、非阻塞逻辑 |

---

## 六、Hooks 顺序（Fiber 链表）

条件/循环中调用 Hooks → 链表错位 → 运行时崩溃或 state 错乱。**ESLint `rules-of-hooks` 必开。**

---

## 排查实战

### 案例 A：搜索 500ms 卡顿

- **Profiler：** FilterList 80ms render × 每键
- **修复：** useTransition + memo(ListItem)
- **验证：** INP <200ms

### 案例 B：Modal 打开闪动

- **原因：** useEffect 里改 position，paint 后才改
- **修复：** useLayoutEffect 同步定位
- **验证：** 无视觉跳动

### 案例 C：Strict Mode 双请求

- **原因：** 开发环境 effect 执行两次
- **处理：** cleanup abort；生产只一次

---

## 手写 mini Fiber（理解用）

```javascript
function workLoop(fiber) {
  while (fiber && !shouldYield()) {
    fiber = performUnitOfWork(fiber);
  }
  if (!fiber && wipRoot) commitRoot();
}

function performUnitOfWork(fiber) {
  // beginWork: 处理当前 fiber，创建子 fiber
  const children = fiber.type(fiber.props);
  // reconcileChildren...
  if (fiber.child) return fiber.child;
  // 无子则找 sibling 或 return 找 parent.sibling
  let next = fiber;
  while (next) {
    if (next.sibling) return next.sibling;
    next = next.return;
  }
}
```

### Lane 模型与 Scheduler

React 18 用 Lane（位掩码）表示优先级：同步、默认、过渡、空闲等。Scheduler 根据过期时间调度 Fiber 工作单元。

**为何放弃 requestIdleCallback？** → 触发频率不稳定（约 1 次/秒）、浏览器可自行决定空闲时长，不适合 UI 调度。

## 面试高频问答（追问链）

> 大厂对一个考点通常追问到能力边界：**概念 → 机制 → 边界 → ⭐ 原理（触底）→ 实战（落地）**。实战层是区分「背过」和「做过」的关键。Fiber 是大厂 React 岗的深挖重灾区。

### 链一：Fiber 架构（必问）

1. **概念**：Fiber 解决什么问题？→ 大更新长任务阻塞主线程，引入可中断、可恢复的渲染。
2. **机制**：怎么实现可中断？→ 把递归拆成链表单元（child/sibling/return），每单元后检查时间片让出。
3. **边界**：Render 和 Commit 区别？→ Render 可中断算变更（不碰 DOM）；Commit 同步写 DOM 不可中断。
4. **应用**：双缓冲是什么？→ current 树（屏幕）+ workInProgress 树（构建中），完成后切换。
5. ⭐ **原理（触底）**：为什么 React 自己实现 Scheduler 而不用 requestIdleCallback？Lane 模型怎么调度优先级？→ rIC 兼容差、调用频率低不稳定；React 用 MessageChannel 模拟 5ms 时间片，Lane 用位运算表示多优先级、批量合并，保证高优先级（输入）插队。
6. **实战（落地）**：大列表筛选输入卡顿怎么治？→ Profiler 录输入过程看长任务；大列表加 useTransition 包 setState；优化后 INP/输入延迟明显下降，Profiler 高优更新先完成。

### 链二：并发特性

1. **概念**：useTransition 原理？→ 标记低优先级更新，高优先级先响应。
2. **机制**：useDeferredValue 和它区别？→ 一个包更新逻辑、一个包派生值，本质都是降优先级。
3. **应用**：什么场景用？→ 大列表筛选、Tab 切换避免输入卡顿。
4. ⭐ **原理（触底）**：并发模式下组件可能渲染多次/被丢弃，对副作用有什么要求？→ render 必须纯、副作用放 effect；否则中断重启会产生重复副作用或状态撕裂（tearing）。
5. **实战（落地）**：StrictMode 下 effect 执行两次怎么应对？→ 区分 mount 副作用与幂等请求；AbortController 取消重复 fetch；单测 + 线上监控确认无重复提交/双弹窗。

## 小结

- Fiber = 可中断 Render + 同步 Commit
- Profiler 是日常优化主工具
- useTransition 治输入卡；memo 治无效 render
- useLayoutEffect 治布局闪动

## 延伸阅读

- [React Profiler](https://react.dev/learn/react-developer-tools)
- [useTransition](https://react.dev/reference/react/useTransition)
