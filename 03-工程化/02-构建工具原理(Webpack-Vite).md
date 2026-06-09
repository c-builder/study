# 构建工具原理（Webpack / Vite）

## 学习目标

- 读懂构建流程，能用 bundle 分析器定位体积问题
- 能在 Vite/Webpack 项目中落地代码分割、缓存、提速
- 能排查 HMR 失效、构建慢、Tree Shaking 无效

## 为什么需要

「首屏 JS 2MB」「改一行全量编译 3 分钟」——架构师必须能读构建产物并优化。

> 核心心法：**构建 = 依赖图 + 转换 + 分包 + 输出。** 优化就是减图、减转换、减输出。

---

## 一、心智模型

| 现象 | 先查 |
|------|------|
| 包太大 | analyzer 看谁占体积 |
| 构建慢 | 缓存、thread、减少 loader |
| HMR 全刷新 | 边界模块、循环依赖 |
| 摇树无效 | sideEffects、CJS 依赖 |

---

## 二、体积分析（第一步必做）

```bash
# Vite
npx vite-bundle-visualizer

# Webpack
npx webpack-bundle-analyzer dist/stats.json
```

**常见发现：** 整包 lodash、moment locale、重复 react、未 lazy 的路由。

---

## 三、优化落地清单

### 代码分割

```javascript
// 路由 lazy
const Admin = lazy(() => import('./pages/Admin'));

// Vite manualChunks
build: {
  rollupOptions: {
    output: {
      manualChunks: { vendor: ['react', 'react-dom'] }
    }
  }
}
```

### Tree Shaking

```json
// package.json 库作者
{ "sideEffects": false }
```

```javascript
// 应用侧
import debounce from 'lodash-es/debounce'; // 不要 import _ from 'lodash'
```

### 构建提速

```javascript
// Webpack 5
cache: { type: 'filesystem' }

// Vite 已 esbuild 预构建；大型项目 tuning optimizeDeps.include
```

---

## 四、HMR 排查

1. 改组件是否整页刷新？→ 看控制台 HMR 日志
2. 是否改到 entry 边界外不可 HMR 模块？
3. 是否 `accept` 链断裂（Webpack）

---

## 排查实战

### 案例 A：主包 1.8MB

- **analyzer：** echarts 全量 + moment
- **修复：** echarts 按需 `echarts/core`；dayjs 替 moment
- **验证：** 主包 420KB，Lighthouse TBT 降 40%

### 案例 B：CI 构建 8 分钟

- **原因：** 无 persistent cache，每次全量
- **修复：** Webpack filesystem cache + Turborepo 远程 cache
- **验证：** 二次构建 <2 分钟

---

## 项目落地步骤

1. 接入 bundle analyzer，设预算（script <300KB gzip）
2. 路由全 lazy + vendor split
3. CI 跑 `pnpm build` + Lighthouse CI
4. PR 模板要求附 analyzer 截图（体积变更时）

## 小结

- analyzer 找大头；lazy + manualChunks 拆包
- sideEffects + ESM 依赖才能摇树
- cache 提速；HMR 看边界与日志

## 延伸阅读

- [Vite Build](https://vitejs.dev/guide/build.html)
- [Webpack Code Splitting](https://webpack.js.org/guides/code-splitting/)
