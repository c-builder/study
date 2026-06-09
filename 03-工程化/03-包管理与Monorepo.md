# 包管理与 Monorepo

## 学习目标

- 理解 npm/yarn/pnpm 的差异与 pnpm 原理
- 掌握 Monorepo 的概念与 workspace 配置
- 了解 Turborepo、Nx 等 Monorepo 工具
- 能设计多包项目的依赖与发布策略

## 为什么需要

现代前端项目常包含：

- 主应用 + 组件库 + 工具包 + 共享配置
- 多应用共享代码（微前端、多端）

Monorepo 将多个包放在同一仓库，统一版本、依赖、构建，是大型团队的标准实践。

## 核心原理

### 1. 包管理器对比

| 特性 | npm | yarn | pnpm |
|------|-----|------|------|
| node_modules | 扁平/嵌套 | 扁平 | 内容寻址 + 硬链接 |
| 磁盘占用 | 高（重复安装） | 高 | 低（全局 store） |
| 安装速度 | 中 | 快 | 最快 |
| 幽灵依赖 | 有 | 有 | 无（严格） |

**pnpm 原理：**

```mermaid
flowchart LR
    Store[全局 Store] -->|硬链接| ProjectA[项目 A node_modules]
    Store -->|硬链接| ProjectB[项目 B node_modules]
    
    subgraph projectA [项目 A]
        Symlink[.pnpm 符号链接]
        Direct[直接依赖]
    end
```

- 全局 store 存所有包版本
- 项目 node_modules 硬链接到 store，节省磁盘
- 依赖通过 `.pnpm` 严格隔离，无法访问未声明的依赖

### 2. Monorepo 结构

```
my-monorepo/
├── packages/
│   ├── app/           # 主应用
│   ├── ui/            # 组件库
│   ├── utils/         # 工具函数
│   └── config-eslint/ # 共享 ESLint 配置
├── package.json       # 根 package.json
├── pnpm-workspace.yaml
└── turbo.json
```

### 3. pnpm workspace

```yaml
# pnpm-workspace.yaml
packages:
  - 'packages/*'
  - 'apps/*'
```

```json
// 根 package.json
{
  "name": "my-monorepo",
  "private": true,
  "scripts": {
    "build": "turbo run build",
    "dev": "turbo run dev"
  },
  "devDependencies": {
    "turbo": "^2.0.0"
  }
}

// packages/ui/package.json
{
  "name": "@myorg/ui",
  "version": "1.0.0",
  "main": "./dist/index.js"
}

// packages/app/package.json
{
  "name": "app",
  "dependencies": {
    "@myorg/ui": "workspace:*"
  }
}
```

`workspace:*` 表示依赖 workspace 内的包，发布时替换为实际版本。

### 4. Turborepo

```json
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**"]
    },
    "dev": {
      "cache": false,
      "persistent": true
    }
  }
}
```

**特性：**

- **任务编排**：`dependsOn` 定义 build 顺序
- **远程缓存**：CI 共享构建缓存
- **增量构建**：只构建变更的包

```mermaid
flowchart TD
    Utils[utils build] --> UI[ui build]
    Utils --> App[app build]
    UI --> App
```

### 5. 版本与发布

**Changesets：** 管理多包版本与 changelog

```bash
pnpm changeset          # 选择变更的包，写 changelog
pnpm changeset version    # 更新版本号
pnpm changeset publish    # 发布到 npm
```

**策略：**

- 内部包：`workspace:*`，不单独发布
- 对外组件库：独立版本，semver
- 统一 lint/test 在根目录执行

### 6. npm 依赖解析与幽灵依赖

npm/yarn 扁平化 `node_modules` 时，A 的依赖可能被提升到顶层，B 可直接 `require` 未声明的包 → **幽灵依赖**。pnpm 通过 `.pnpm` 严格隔离，只能访问 `package.json` 声明的依赖。

### 7. semver 与 lockfile

| 符号 | 含义 | 示例 |
|------|------|------|
| `^1.2.3` | 兼容 minor/patch | `<2.0.0` |
| `~1.2.3` | 兼容 patch | `<1.3.0` |
| 精确 | 锁定 | `1.2.3` |

**lockfile**（`pnpm-lock.yaml` / `package-lock.json`）锁定完整依赖树，CI 使用 `--frozen-lockfile` 保证可复现。

### 8. peerDependencies

```json
{
  "peerDependencies": {
    "react": ">=18.0.0"
  }
}
```

组件库声明 peer，避免打包多份 React。宿主项目负责安装匹配版本。

### 9. npm scripts 与 npx

```json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build"
  }
}
```

`pnpm exec` 运行本地 bin，无需全局安装。

### 10. Nx / Lerna 对比

| 工具 | 特点 |
|------|------|
| Turborepo | 任务编排 + 远程缓存，轻量 |
| Nx | 依赖图分析、代码生成、插件丰富 |
| Lerna | 传统多包版本发布，现多与 Nx 集成 |

### 11. 私有 registry

Verdaccio、云厂商私有 npm，配合 `.npmrc`：`@myorg:registry=https://npm.example.com/`

---

## 实战：从 0 搭 Monorepo（半天）

```bash
mkdir my-monorepo && cd my-monorepo
pnpm init
echo "packages:\n  - 'packages/*'" > pnpm-workspace.yaml
mkdir -p packages/ui packages/app
pnpm add -D turbo typescript -w
```

**验证：** `pnpm --filter app dev` 能引用 `@repo/ui`；改 ui 后 app HMR 生效。

**踩坑：** 子包未声明依赖却 import → pnpm 报错；正确在 app/package.json 加 `"@repo/ui": "workspace:*"`。

## 常见误区与最佳实践

| 误区 | 正确理解 |
|------|---------|
| "Monorepo = 一个大 package.json" | 多 package，workspace 关联 |
| "pnpm 和 npm 完全一样" | pnpm 严格隔离，需显式声明依赖 |
| "所有代码放一个包" | 按职责拆包，避免循环依赖 |
| "Monorepo 一定比 Multi-repo 好" | 小团队单应用 Multi-repo 更简单 |

**最佳实践：**

- 新项目 Monorepo + pnpm + Turborepo
- 包边界清晰：ui、utils、config 分离
- 根目录统一 ESLint、TypeScript、Prettier
- CI 用 turbo run build --filter=...[changed]

## pnpm 原理深挖

- **content-addressable store**：全局存储包内容，硬链接到 node_modules
- **软链接**：`.pnpm` 虚拟 store 用符号链接组织依赖树
- **严格模式**：只能访问 package.json 声明的依赖，杜绝幻影依赖
- **大规模治理**：`turbo run --filter=...[origin/main]` 只构建变更包

## 面试高频问答（追问链）

> 大厂对一个考点通常追问到能力边界：**概念 → 机制 → 边界 → ⭐ 原理（触底）→ 实战（落地）**。实战层是区分「背过」和「做过」的关键。

### 链一：pnpm 原理

1. **概念**：pnpm 为什么省磁盘？→ 全局 store + 硬链接，同版本只存一份。
2. **机制**：node_modules 结构？→ 扁平 `.pnpm` 存真实包，软链接构建嵌套依赖树。
3. **边界**：幻影依赖是什么？pnpm 怎么治？→ npm 扁平化使未声明依赖可被引用；pnpm 严格隔离只暴露直接依赖。
4. ⭐ **原理（触底）**：硬链接 vs 软链接在 pnpm 里分别用在哪？→ 硬链接把 store 文件链接进 `.pnpm`（共享内容、零拷贝）；软链接把依赖软链到包目录构建依赖关系（保证 require 解析正确）。
5. **实战（落地）**：团队从 npm/yarn 迁到 pnpm 你怎么落地？→ CI 安装慢 + 磁盘占用高 + 幽灵依赖偶发线上报错，落地 `pnpm import` 转 lockfile + 统一 `.npmrc` + 修复被严格模式暴露的未声明依赖；验证 `pnpm install --frozen-lockfile` CI 可复现、安装时长下降、`pnpm why` 单版本；结果磁盘占用降、幽灵依赖类 bug 归零。

### 链二：Monorepo 治理

1. **概念**：Monorepo 优势与成本？→ 代码共享、原子提交、统一工具链；成本在治理与构建放大。
2. **机制**：Turborepo 怎么加速？→ 任务编排 + 内容哈希缓存，命中跳过重复构建。
3. **应用**：Changesets 做什么？→ 多包版本管理、生成 changelog、发版 PR。
4. ⭐ **原理（触底）**：大型 Monorepo 构建变慢怎么治理？→ 增量构建 + 远程缓存（Turbo Remote Cache）+ 受影响包检测（affected）+ CI 任务分片，避免全量构建。
5. **实战（落地）**：你怎么治理一个变慢的大型 Monorepo？→ 全量构建 20 分钟，落地 Turborepo 任务编排 + 远程缓存 + `--filter=...[origin/main]` 只构建受影响包 + CI 分片并行；验证命中缓存的包直接跳过、改一个叶子包只构建其下游、Changesets 管多包发版；结果 PR CI 从 20 分钟降到 3-5 分钟。

## 小结

- pnpm 通过 store + 硬链接省磁盘、防幽灵依赖
- Monorepo 多包同仓，workspace 协议关联
- Turborepo 任务编排 + 缓存加速构建
- Changesets 管理多包版本发布

## 延伸阅读

- [pnpm 文档](https://pnpm.io/)
- [Turborepo 文档](https://turbo.build/repo/docs)
- [Changesets](https://github.com/changesets/changesets)
