# 虚拟 DOM 与 Diff

## 学习目标

- 读懂 VDOM 解决什么问题，何时 VDOM 反而是开销
- 能排查 key 错误、列表状态错乱等真实 bug
- 会用 React/Vue DevTools 验证列表渲染性能

## 为什么需要

列表乱序、输入框失焦、状态串行——多数和 **key** 或 **Diff 复用策略** 有关。

> 核心心法：**Diff 假设同层同类型可复用；key 标识「身份」。** key 错了，DOM 复用就错。

---

## 一、心智模型

| 现象 | 先怀疑 |
|------|--------|
| 列表 reorder 后 input 内容串了 | index 作 key |
| 删除一条后状态错位 | key 不稳定 |
| 列表卡顿 | 无虚拟化 + 全量 Diff |
| 组件 state 被重置 | 父 key 变化导致 remount |

---

## 二、key 规则：读懂 + 验证

```jsx
// 错误 — 插入/删除时 index 错位
{items.map((item, i) => <Row key={i} data={item} />)}

// 正确 — 稳定唯一 id
{items.map(item => <Row key={item.id} data={item} />)}

// 错误 — 随机 key 导致每次 remount
<Row key={Math.random()} />
```

**验证：** React DevTools → 勾选 **Highlight updates**，操作列表看是否整列表闪动。

---

## 三、Diff 策略（排查用）

1. **不同类型** → 整棵子树替换（state 丢失）
2. **同类型** → 比 props，更新 DOM
3. **列表** → key 映射，最小移动（Vue3 用 LIS）

**反模式：**

```jsx
// 切换 tab 用三元 — OK 若类型不同会卸载
{tab === 'a' ? <PanelA /> : <PanelB />}

// 若需保留 state，用 CSS 隐藏而非卸载，或 lifting state
```

---

## 四、性能：定位 + 优化

### React Profiler

录制列表滚动 → 看 List 组件 render 次数和耗时。

### 优化手段

```jsx
const Row = memo(function Row({ item }) { /* ... */ });

// 大列表
import { FixedSizeList } from 'react-window';
<FixedSizeList height={600} itemCount={n} itemSize={50}>
  {({ index, style }) => <div style={style}>{items[index].name}</div>}
</FixedSizeList>
```

---

## 五、手写 mini Diff（理解用）

```javascript
function patch(oldVNode, newVNode) {
  if (oldVNode.type !== newVNode.type) return replace(oldVNode, newVNode);
  patchProps(oldVNode.el, oldVNode.props, newVNode.props);
  patchChildren(oldVNode, newVNode);
}
```

理解即可，生产用框架实现。

---

## 排查实战

### 案例 A：Todo 删除后 checkbox 状态错乱

- **原因：** `key={index}`
- **修复：** `key={todo.id}`
- **验证：** 任意增删改顺序，状态正确

### 案例 B：路由切换表单未清空

- **原因：** 同组件不同路由复用，未 key route
- **修复：** `<Route element={<Form key={location.pathname} />} />`

### 案例 C：1000 行表格卡

- **原因：** 全量 DOM
- **修复：** 虚拟滚动
- **验证：** Profiler render 时间 <16ms

---

## 小结

- key 必须稳定唯一，禁用 index（动态列表）
- Diff 同层比较；类型变则 remount
- 性能：memo + 虚拟列表 + 稳定 props
- Profiler 验证优化效果

## 延伸阅读

- [React Preserving State](https://react.dev/learn/preserving-and-resetting-state)
- [Vue Rendering Mechanism](https://vuejs.org/guide/extras/rendering-mechanism.html)
