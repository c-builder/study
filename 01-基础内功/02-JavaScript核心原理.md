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

## 手写实现（面试必考）

### new / call / bind / instanceof

```javascript
function myNew(Ctor, ...args) {
  const obj = Object.create(Ctor.prototype);
  const ret = Ctor.apply(obj, args);
  return ret instanceof Object ? ret : obj;
}

Function.prototype.myBind = function (ctx, ...preset) {
  const fn = this;
  return function (...args) {
    if (new.target) return new fn(...preset, ...args);
    return fn.apply(ctx, [...preset, ...args]);
  };
};

function myInstanceof(obj, Ctor) {
  let p = Object.getPrototypeOf(obj);
  while (p) {
    if (p === Ctor.prototype) return true;
    p = Object.getPrototypeOf(p);
  }
  return false;
}
```

### 寄生组合继承

```javascript
function inherit(Child, Parent) {
  Child.prototype = Object.create(Parent.prototype);
  Child.prototype.constructor = Child;
  Object.setPrototypeOf(Child, Parent);
}
```

### this 四条判定规则

1. new → 指向新对象
2. call/apply/bind → 指向指定对象
3. 对象方法 → 指向调用对象
4. 默认 → 严格模式 undefined，非严格 globalThis

## 面试高频问答（追问链）

> 大厂对一个考点通常追问到能力边界：**概念 → 机制 → 边界 → ⭐ 原理（触底）→ 实战（落地）**。实战层是区分「背过」和「做过」的关键。

### 链一：原型与原型链

1. **概念**：原型链是什么？→ 对象 `__proto__` 指向构造函数 `prototype`，逐层向上，终点 `Object.prototype.__proto__ === null`。
2. **机制**：属性查找过程？→ 自身 → 原型 → 原型的原型，找不到返回 undefined。
3. **边界**：`new` 做了什么？→ 创建对象、链接原型、绑 this 执行、返回（构造函数返回对象则用它）。
4. **应用**：怎么实现继承？→ 寄生组合继承（`Object.create(Parent.prototype)` + 修正 constructor）。
5. ⭐ **原理（触底）**：`class` 的本质？为什么说是语法糖但不完全等价？→ 底层仍是原型；但 class 不可提升、内部严格模式、方法不可枚举、必须 new 调用。
6. **实战（落地）**：`instanceof` 跨 iframe 失效怎么在项目里处理？→ 微前端/嵌入页统一用 `Array.isArray`/toString 判型；SDK 对外暴露类型守卫函数，避免跨 realm instanceof；单测覆盖 iframe 场景。

### 链二：闭包

1. **概念**：闭包是什么？→ 函数 + 其引用的词法环境。
2. **机制**：为什么能访问外部变量？→ 函数 `[[Environment]]` 持有定义时的作用域链。
3. **边界**：闭包一定内存泄漏吗？→ 否，未释放的引用才泄漏。
4. **应用**：闭包用在哪？→ 模块私有状态、柯里化、防抖节流、React Hooks。
5. ⭐ **原理（触底）**：循环里 `var` + setTimeout 全打印同值，为什么？怎么修？→ var 函数作用域共享同一变量；改 let（块级，每轮新绑定）或 IIFE 传参。
6. **实战（落地）**：React 里 stale closure 怎么产生、怎么解？→ 定时器/回调捕获旧 state；deps 补全、函数式 setState、useRef 存最新值；React DevTools Profiler 复现后改 deps 验证不再复现。

### 链三：this 与类型判断

1. **概念**：this 由什么决定？→ 调用方式，不是定义位置。
2. **机制**：四条规则？→ new > 显式 call/apply/bind > 方法调用 > 默认（严格 undefined）。
3. **边界**：箭头函数的 this？→ 无 own this，词法绑定外层，不能 new、无 arguments。
4. ⭐ **原理（触底）**：`Object.prototype.toString.call` 为什么最准？→ 读取内部 `[[Class]]`/Symbol.toStringTag，能区分 Array/Date/null 等 typeof 无法区分的类型。
5. **实战（落地）**：工具库里怎么做可靠类型判断？→ 封装 `getType(val)` 用 toString + 自定义 tag；表单/接口层统一调用；单测覆盖 null/Date/跨 iframe Array，避免散落 typeof 误判。

## 小结

- Scope 面板验证闭包；Heap snapshot 查泄漏
- 原型链解释继承；this 看调用方式
- 闭包陷阱在循环和 Hooks 中最常见
- 类型转换用 === 和显式转换规避

## 延伸阅读

- [You Don't Know JS](https://github.com/getify/You-Dont-Know-JS)
- [MDN: Memory Management](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Memory_management)
- [Chrome DevTools Memory](https://developer.chrome.com/docs/devtools/memory-problems)
