# 技术方案文档（RFC / ADR）

## 学习目标

- 能写一份可评审、可执行的 RFC
- 能写 ADR 记录决策并归档
- 能组织评审会议并产出 action items

## 为什么需要

架构师产出主要是**文档**。好文档 = 减少重复讨论 + 新人可接手 + 决策可追溯。

> 核心心法：**RFC 说服人做；ADR 记录为什么做了。**

---

## 一、RFC 完整范例（节选：引入 Module Federation）

```markdown
# RFC-007: 主应用与子应用 Module Federation 集成

## 元信息
- 作者：张三 | 状态：Accepted | 日期：2025-06-01
- 评审：@前端负责人 @运维

## 摘要
主应用 webpack5 通过 MF 加载 3 个子应用，实现独立部署，解决 monolith 发布耦合。

## 背景
- 现状：单仓 40 万行，发布需全量回归 4h
- 数据：Q2 因发布延迟需求延期 3 次
- 不做代价：继续阻塞，Q3 预计 5 团队并行

## 目标 / 非目标
**目标：** 子应用独立 CI；共享 react 单例；路由 /app-a/* 加载
**非目标：** 不改子应用技术栈；不做 SSR 统一

## 方案
### 架构图
（Mermaid：Host -- remoteEntry --> SubA/SubB）

### 详细设计
- Host `ModuleFederationPlugin` shared react singleton
- Sub 暴露 `./App`，publicPath 自动
- 路由：React Router 嵌套 + lazy import('subA/App')

### 备选
| 方案 | 优 | 劣 |
|------|----|----|
| qiankun | 框架无关 | 运行时沙箱复杂 |
| MF | 构建集成、共享依赖 | 绑 webpack |

## 迁移计划
- W1：POC 1 个子应用
- W2-4：迁移 3 个子应用
- W5：下线 monolith 对应模块

## 风险
| 风险 | 缓解 |
|------|------|
| 双 react | shared singleton + pnpm why 检查 |
| 版本不兼容 | 约定 shared 版本范围 |

## 成功指标
- 子应用发布无需主应用发版
- 主包体积不增 >10%

## 开放问题
- [ ] 子应用 CI 模板谁维护？
```

---

## 二、ADR 范例

```markdown
# ADR-007: 采用 Module Federation 集成子应用

## 状态
Accepted（ superseded by ADR-012 若未来迁移）

## 决策
Host/Sub 使用 Webpack 5 Module Federation，react/react-dom shared singleton。

## 理由
RFC-007 评审通过；POC 验证加载 <500ms；团队 webpack 熟。

## 后果
+ 独立部署达成
- 需统一 webpack 5 升级
- 运维多 3 条 CDN 路径

## 备选
qiankun：沙箱好但 bundle 重复
```

---

## 三、评审会议流程（60 分钟）

| 时间 | 内容 |
|------|------|
| 0-5 | 作者讲摘要+推荐方案 |
| 5-25 | 评审人提问（提前 24h 读 RFC） |
| 25-45 | 讨论备选与风险 |
| 45-55 | 投票 Accepted / Needs revision |
| 55-60 | 记录 action items + 负责人 |

**评审清单：**

- [ ] 问题是否量化？
- [ ] 非目标是否明确？
- [ ] 有无更简单方案？
- [ ] 迁移与回滚是否可执行？
- [ ] 成功指标是否可测？

---

## 四、文档驱动与知识库

```
docs/
├── rfcs/007-module-federation.md
├── adr/007-mf-adoption.md
└── runbooks/deploy-subapp.md
```

- PR 链接 RFC/ADR 编号
- Accepted RFC 不可改，修订用新 RFC
- ADR 只增不改，supersede 用新 ADR

---

## 实战作业

1. 选你当前项目一个真实痛点
2. 按 RFC 模板写 2 页
3. 找同事做 30 分钟 review
4. 产出 ADR 1 页

---

## 小结

- RFC = 提案说服；ADR = 决策存档
- 好 RFC 有背景数据、备选、风险、指标
- 评审要限时、有 checklist、有 action items

## 延伸阅读

- [ADR GitHub](https://adr.github.io/)
- Google [Design Docs](https://google.github.io/eng-practices/review/)
