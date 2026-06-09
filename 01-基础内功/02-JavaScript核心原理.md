# JavaScript 核心原理

## 学习目标

- 读懂执行上下文、原型链、闭包、this，并能在调试器中验证
- 能用 Memory 面板排查内存泄漏
- 掌握类型转换陷阱，避免线上 bug
- 能在 Code Review 中识别常见 JS 反模式

## 为什么需要

JS 是前端运行时。不懂原理，你会：

- 看不懂 `this` 为何是 undefined
- 改 state 不更新（React）或响应式失效（Vue）
- 排查不了「页面越用越卡」（内存泄漏）
- 面试和线上 bug 反复踩原型链、闭包、异步

> 核心心法：**原理不是用来背的，是用来解释 DevTools 里看到的现象的。**

---

## 一、心智模型：现象 → 原理 → 工具

| 现象 | 先怀疑 | 验证工具 |
|------|--------|---------|
| `this` 不对 | 调用方式 / 箭头函数 | Console 打断点看 this |
| 循环 setTimeout 输出全相同 | var + 闭包 | 跑 demo + 改 let |
| 属性找不到 | 原型链 | `console.log(obj.__proto__)` |
| 页面越用越卡 | 泄漏：listener/timer/DOM | Memory → Heap snapshot |
| `[] + {}` 怪结果 | 类型转换 | 查 ToPrimitive 规则 |

---

## 二、执行上下文与作用域

### 读懂

每次函数调用创建一个**执行上下文**：变量环境、词法环境、this、外层引用（作用域链）。

### 定位（调试）

1. Sources 面板 → 打断点
2. Scope 侧边栏看 **Local / Closure / Global**
3. 闭包变量在 Closure 下可见

### 变量提升与 TDZ

```javascript
console.log(a); // undefined — var 提升
var a = 1;
console.log(b); // ReferenceError — TDZ
let b = 2;
```

**项目实践：** 全项目 eslint `no-var`，避免提升陷阱。

---

## 三、原型链：读懂 + 验证 + 应用

```mermaid
flowchart LR
    obj[实例] --> proto[Constructor.prototype]
    proto --> ObjectProto[Object.prototype]
    ObjectProto --> null[null]
```

```javascript
function User(name) { this.name = name; }
User.prototype.sayHi = function() { return this.name; };
const u = new User('Alice');

// 验证查找链
console.log(u.sayHi === User.prototype.sayHi); // true
console.log(Object.getPrototypeOf(u) === User.prototype); // true
```

**new 四步（面试 + 手写）：** 建对象 → 连原型 → 绑 this 执行 → 返回对象

**项目应用：** 读 Vue/React 源码时看到 `instanceof`、`extends`，本质都是原型链。

---

## 四、闭包：读懂 + 排查 + 应用

**定义：** 函数 + 其引用的词法环境；外层执行完，被引用的变量仍存活。

```javascript
function createCounter() {
  let count = 0;
  return {
    inc: () => ++count,
    get: () => count
  };
}
```

**经典 bug（必会）：**

```javascript
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100); // 3,3,3
}
for (let j = 0; j < 3; j++) {
  setTimeout(() => console.log(j), 100); // 0,1,2
}
```

**React Hooks 闭包陷阱：**

```javascript
useEffect(() => {
  const id = setInterval(() => {
    setCount(count + 1); // 永远 +1，count 是创建 effect 时的快照
  }, 1000);
  return () => clearInterval(id);
}, []); // 修复：用 setCount(c => c + 1) 或 count 加入 deps
```

---

## 五、this：读懂 + 定位表

| 调用方式 | this |
|---------|------|
| 普通函数 | 非严格：global；严格：undefined |
| obj.method() | obj |
| call/apply/bind | 指定对象 |
| new | 新对象 |
| 箭头函数 | 外层 this |

**项目规则：** 对象方法用普通函数；回调若需组件 this 用箭头或 bind；React 类组件注意 bind。

---

## 六、类型转换（线上 bug 高发区）

```javascript
0 == false    // true
'' == false   // true
[] == false   // true
null == undefined // true

// 项目规范：始终 ===
Number('')    // 0
Number('abc') // NaN
```

**验证：** ESLint `eqeqeq: always`

---

## 七、内存泄漏：定位 + 修复（实战重点）

### Memory 面板步骤

1. DevTools → **Memory** → **Heap snapshot**
2. 操作前拍快照 A
3. 执行可疑操作（打开关闭 Modal 10 次）
4. 强制 GC（垃圾桶图标）→ 快照 B
5. Comparison 视图 → 看 **Detached DOM tree**、**Closure** 增长

### 常见泄漏与修复

```javascript
// 1. 未清理 listener
useEffect(() => {
  const onScroll = () => {};
  window.addEventListener('scroll', onScroll);
  return () => window.removeEventListener('scroll', onScroll); // 必须
}, []);

// 2. 定时器
useEffect(() => {
  const t = setInterval(tick, 1000);
  return () => clearInterval(t);
}, []);

// 3. 全局数组持有 DOM
const cache = [];
// 移除 DOM 时 cache.push 要对应 delete
```

### 案例：SPA 路由切换后内存涨

1. **现象：** 切换 20 次路由，Memory 涨 50MB
2. **Heap：** Detached `<div>` 来自未卸载的 chart 实例
3. **修复：** 组件 unmount 时 `chart.dispose()`
4. **验证：** 20 次切换后 Heap 稳定

---

## 八、项目落地步骤

1. **规范：** eslint no-var、eqeqeq、no-unused-vars
2. **Review 清单：** effect 有 cleanup？listener/timer 清理？
3. **上线前：** 关键页 Memory snapshot 对比
4. **培训：** 闭包 + this + 原型链各一个 demo

---

## 排查实战案例

### 案例 A：按钮点击 count 不变

- **原因：** 闭包旧 state，functional update 未用
- **修复：** `setCount(c => c + 1)`
- **验证：** 连续点击正确递增

### 案例 B：`instanceof` 异常

- **原因：** 跨 iframe 或多 realm
- **修复：** `Array.isArray()` / `Object.prototype.toString.call()`

### 案例 C：第三方库 + 全局污染

- **原因：** 库写 `window.xxx` 未清理
- **修复：** 卸载时 delete 或 iframe 隔离

---

## 常见误区与最佳实践

| 误区 | 正确理解 |
|------|---------|
| 闭包=泄漏 | 特性；未清理才泄漏 |
| 箭头函数万能 | 无 arguments、不能 new、this  lexical |
| 深拷贝 JSON 够用 | 丢 Date/函数/undefined |

**可执行清单：**

- [ ] 全项目 === 
- [ ] 每个 useEffect 检查 cleanup
- [ ] 长列表/Modal 测 Memory
- [ ] 手写 new/bind 理解一次

## 小结

- Scope 面板验证闭包；Heap snapshot 查泄漏
- 原型链解释继承；this 看调用方式
- 闭包陷阱在循环和 Hooks 中最常见
- 类型转换用 === 和显式转换规避

## 延伸阅读

- [You Don't Know JS](https://github.com/getify/You-Dont-Know-JS)
- [MDN: Memory Management](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Memory_management)
- [Chrome DevTools Memory](https://developer.chrome.com/docs/devtools/memory-problems)
