# WebAssembly 与高性能计算

## 学习目标

- 理解 WASM 编译执行原理与 JS 互操作
- 掌握 WASM 适用场景与架构选型
- 了解 Emscripten 编译流程与性能优化

## 为什么需要

FFmpeg.wasm、图像处理、加密、游戏引擎等场景，大厂面试常问：**什么时候用 WASM 而不是 JS？边界在哪？**

## 一、WASM 原理

```
C/C++/Rust 源码 → 编译器 → .wasm 二进制 → 浏览器 WASM 引擎 → 机器码执行
```

| 特性 | JS | WASM |
|------|-----|------|
| 执行 | 解释 + JIT | 预编译接近原生速度 |
| 启动 | 快 | 需下载+编译模块，冷启动慢 |
| DOM 访问 | 直接 | 必须通过 JS 胶水代码 |
| 适用 | 业务逻辑、UI | 计算密集、已有 C/Rust 库 |

```mermaid
flowchart LR
    JS[JavaScript] <-->|胶水层| WASM[WASM Module]
    WASM --> CPU[接近原生速度执行]
    JS --> DOM[DOM / API]
```

## 二、JS 与 WASM 互操作

```javascript
// 加载 WASM 模块
const response = await fetch('module.wasm');
const { instance } = await WebAssembly.instantiateStreaming(response, imports);

// 调用导出函数
const result = instance.exports.add(1, 2);

// imports：WASM 调用 JS（访问 DOM 必须通过 imports）
const imports = {
  env: {
    consoleLog: (ptr, len) => {
      const bytes = new Uint8Array(instance.exports.memory.buffer, ptr, len);
      console.log(new TextDecoder().decode(bytes));
    },
  },
};
```

**内存共享**：WASM 线性内存（`WebAssembly.Memory`），JS 通过 `TypedArray` 读写。

```javascript
const memory = new WebAssembly.Memory({ initial: 256 }); // 16MB pages
const view = new Float64Array(memory.buffer);
view[0] = 3.14;
```

## 三、适用场景

| 场景 | 为什么 WASM |
|------|------------|
| 音视频编解码 | FFmpeg.wasm，C 库直接复用 |
| 图像处理 | OpenCV、滤镜、压缩 |
| 加密/哈希 | 性能敏感，已有成熟 C 库 |
| 游戏/物理引擎 | Unity WebGL 导出 |
| 科学计算 | 矩阵运算、仿真 |

**不适合**：
- 简单业务逻辑（WASM 冷启动开销不值）
- 频繁 DOM 操作（胶水层开销大）
- 小包体敏感场景（.wasm 文件体积）

## 四、Emscripten 编译流程

```bash
# C 代码编译为 WASM + JS 胶水
emcc hello.c -o hello.js -s WASM=1 -s EXPORTED_FUNCTIONS='["_add"]'
```

产出：
- `hello.wasm`：二进制模块
- `hello.js`：加载器 + 内存管理 + JS 互操作

**Rust 路径**：`wasm-pack` 更适合现代项目，类型安全的互操作。

## 五、性能优化

- **流式编译**：`instantiateStreaming` 边下载边编译
- **缓存**：Service Worker 缓存 .wasm，二次访问快
- **SIMD / 多线程**：`WebAssembly SIMD`、SharedArrayBuffer + Atomics（需 COOP/COEP 头）
- **减少 JS↔WASM 边界调用**：批量传数据，减少跨边界次数

## 六、与 Web Worker 配合

```javascript
// 主线程
const worker = new Worker('wasm-worker.js');
worker.postMessage({ imageData, width, height });
worker.onmessage = (e) => updateCanvas(e.data);

// wasm-worker.js：在 Worker 中加载 WASM，不阻塞主线程
```

## 七、WASM Component Model 与 wasm-gc

**Component Model**（2024+）：把 WASM 模块封装为可组合组件，接口用 WIT 描述，跨语言互操作更标准。

| 演进 | 说明 |
|------|------|
| 传统 WASM | 线性内存 + 手动胶水 |
| wasm-gc | 语言级对象/字符串，减少手动内存管理 |
| Component Model | 模块化组合，类似「WASM 的 npm」 |

架构师关注点：工具链仍在演进，生产落地评估团队 Rust/C++ 能力与浏览器支持矩阵。

## 八、COOP/COEP 跨域隔离落地

多线程 WASM（SharedArrayBuffer）需要：

```http
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

**Nginx 示例**：

```nginx
add_header Cross-Origin-Opener-Policy same-origin;
add_header Cross-Origin-Embedder-Policy require-corp;
```

**验证**：`crossOriginIsolated === true`；否则 `SharedArrayBuffer` 不可用。

第三方资源需 `Cross-Origin-Resource-Policy: cross-origin` 或改同源托管。

## 手写实现：最小 WASM 加载

```javascript
async function loadWasm(url, imports = {}) {
  const response = await fetch(url);
  if (!WebAssembly.instantiateStreaming) {
    const bytes = await response.arrayBuffer();
    const { instance } = await WebAssembly.instantiate(bytes, imports);
    return instance;
  }
  const { instance } = await WebAssembly.instantiateStreaming(response, imports);
  return instance;
}

// 使用
const instance = await loadWasm('/add.wasm');
console.log(instance.exports.add(2, 3)); // 5
```

## 排查实战

### 案例 A：SharedArrayBuffer is not defined

| 步骤 | 动作 |
|------|------|
| 读懂 | 多线程 WASM 初始化失败 |
| 定位 | `crossOriginIsolated` 为 false |
| 解决 | 配 COOP/COEP 响应头，第三方资源加 CORP |
| 验证 | 控制台 `crossOriginIsolated === true` |

### 案例 B：.wasm 404 或 MIME 错误

| 步骤 | 动作 |
|------|------|
| 读懂 | `instantiateStreaming` 失败 |
| 定位 | 服务器返回 `application/octet-stream` 或路径错误 |
| 解决 | Nginx `types { application/wasm wasm; }` |
| 验证 | Network 看 Content-Type 为 `application/wasm` |

### 案例 C：比 JS 还慢

| 步骤 | 动作 |
|------|------|
| 读懂 | 小数据量 WASM 反而慢 |
| 定位 | 冷启动 + 频繁跨边界调用 |
| 解决 | 批量传数据到线性内存，减少调用次数 |
| 验证 | Benchmark 数据量超过阈值后 WASM 领先 |

## 面试高频问答（追问链）

> 大厂对一个考点通常追问到能力边界：**概念 → 机制 → 边界 → ⭐ 原理（触底）→ 实战（落地）**。实战层是区分「背过」和「做过」的关键。

### 链一：WASM 原理与取舍

1. **概念**：WASM 比 JS 快多少？→ 计算密集 2-10 倍，但胶水层/冷启动有开销需实测。
2. **机制**：为什么快？→ 预编译机器码、无需 JIT 预热、类型固定利于优化。
3. **边界**：能直接操作 DOM 吗？→ 不能，必须通过 JS imports 胶水层。
4. **应用**：什么时候不该用？→ 简单逻辑、频繁 DOM、小包体、团队无 C/Rust 能力。
5. ⭐ **原理（触底）**：JS 和 WASM 之间传大数据为什么是瓶颈？怎么优化？→ 跨边界基本类型可直传，复杂数据需在共享线性内存(ArrayBuffer)序列化拷贝；优化是减少调用次数、用共享内存批量传、避免频繁字符串编解码。
6. **实战（落地）**：JS↔WASM 传大图像你怎么优化？→ 场景：前端图片滤镜 4K 图处理卡 UI 3s+；步骤：滤镜算法 Rust 编译 WASM 放 Worker、共享 ArrayBuffer 批量传像素、减少每像素跨边界调用、instantiateStreaming 边下边编；验证：对比纯 JS 处理耗时、主线程 Long Task 消失；结果：4K 滤镜 3.2s→0.4s，主线程零阻塞。

### 链二：多线程与安全

1. **概念**：WASM 怎么用多线程？→ Web Worker + SharedArrayBuffer + Atomics。
2. **机制**：SharedArrayBuffer 为什么需要 COOP/COEP？→ 防 Spectre 侧信道，要求跨域隔离才启用。
3. ⭐ **原理（触底）**：一个典型落地场景(如音视频/图像处理)你怎么设计 JS+WASM 架构？→ 重计算放 WASM(在 Worker 中)、UI/DOM 留 JS 主线程、共享内存传帧数据、用消息控制，兼顾性能与不卡 UI。
4. **实战（落地）**：音视频/图像处理 JS+WASM 架构你怎么落地？→ 场景：浏览器端视频缩略图提取；步骤：FFmpeg.wasm 放 Worker 解码帧 → 共享内存传帧数据到主线程 Canvas 预览、主线程只管 UI/进度条；验证：4K 视频抽帧主线程 INP 不受影响；结果：无需服务端转码即可完成客户端预览，首帧<2s。

## 常见误区

| 误区 | 正确理解 |
|------|---------|
| "WASM 一定比 JS 快" | 小任务冷启动+胶水开销可能更慢，需 Benchmark |
| "WASM 能直接操作 DOM" | 必须经 JS imports，DOM 密集不适合 |
| "忽略 COOP/COEP" | 多线程 WASM 依赖跨域隔离 |
| "每次调用传大对象" | 用共享线性内存批量传，减少跨边界 |

## 小结

- WASM = 近原生速度 + 复用 C/Rust 生态，通过 JS 胶水访问 Web API
- 适用计算密集；不适用 DOM 密集和简单逻辑
- 优化：流式编译、Worker、减少跨边界调用、SIMD
- 多线程需 COOP/COEP；Component Model 是下一代互操作方向
- Rust + wasm-pack 是近年主流工具链

## 延伸阅读

- [WebAssembly MDN](https://developer.mozilla.org/zh-CN/docs/WebAssembly)
- [Emscripten 文档](https://emscripten.org/docs/getting_started/index.html)
- [wasm-pack](https://rustwasm.github.io/wasm-pack/)
