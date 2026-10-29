# 知识索引

> KnowAtlas 的唯一知识入口。`wiki/` 下每个页面都必须在此登记，未登记的页面视为不存在。
> 维护规则见 `AGENTS.md`。

---

## 大类地图

| 大类 | 状态 | 说明 |
|---|---|---|
| [`tech/`](tech/) | ✅ 有内容 | 技术：语言与工程、Agent 运行时、云原生、数据基础设施 |
| `law/` | 预留 | 法律：尚未录入 |
| `finance/` | 预留 | 金融：尚未录入 |

大类是封闭词表，新增需先在本表登记，详细规则见 `AGENTS.md` §3。

---

## tech（技术）

### 语言与工程 · `tech/languages/`

| 页面 | 一句话 | 章节重点 | 类型 |
|---|---|---|---|
| [typescript.md](tech/languages/typescript.md) | 从 TS 类型系统到 Node 运行时、包管理与 Electron 的通用工程参考，不绑定具体项目 | 类型系统 / 事件循环 / pnpm / monorepo / Electron | deep-dive |
| [rust.md](tech/languages/rust.md) | Rust 语言内核 + 工程实践：从所有权限位到 Cargo、测试、CI 与发布 | 所有权与生命周期 / trait 与泛型 / 闭包与迭代器 / 异步 / Cargo 工程化 / 测试体系 / API 设计 / 性能 | deep-dive |
| [go.md](tech/languages/go.md) | Go 语言内核 + 工程实践 + runtime 深挖：从切片/接口/泛型到 GMP、分配器、GC 与诊断 | 切片与 map / 接口与泛型 / goroutine 与 channel / 模块与测试 / GMP 调度 / 内存分配 / GC / pprof | deep-dive |

### Agent Harness · `tech/agent-harness/`

| 页面 | 一句话 | 章节重点 | 类型 |
|---|---|---|---|
| [codex-harness.md](tech/agent-harness/codex-harness.md) | 先讲 agent harness 通用原理，再逐层落到 Codex 的控制循环与实现 | agent loop / 统一协议 / steering / 工具系统 / 上下文管理 / multi-agent | deep-dive |
| [deepseek-harness.md](tech/agent-harness/deepseek-harness.md) | 基于 Cordis 的插件化 Agent 运行时完整拆解 | 插件模型 / profile 组合 / turn 生命周期 / 安全模型 / 设计模式 | deep-dive |

### 云原生 · `tech/cloud-native/`

| 页面 | 一句话 | 章节重点 | 类型 |
|---|---|---|---|
| [kubernetes.md](tech/cloud-native/kubernetes.md) | 以 v1.37.0 源码为基准核对的 K8s 核心概念与架构 | 组件通信 / RBAC / 调度 / 网络 / 资源隔离 / Pod 生命周期 | deep-dive |

### 数据基础设施 · `tech/data-infra/`

| 页面 | 一句话 | 章节重点 | 类型 |
|---|---|---|---|
| [fluss.md](tech/data-infra/fluss.md) | 面向实时分析与 AI 的流式存储，定位湖仓架构的实时数据层 | 数据模型 / Coordinator + TabletServer / Kafka 兼容 / 湖集成 / tiering | deep-dive |

---

## 交叉主题

- **语言 ↔ 实现**：`tech/languages/rust.md` 讲语言规则与工程做法，`tech/agent-harness/codex-harness.md` 是这些规则在大型 Rust 工程里的实现案例，建议对照阅读（语言页示例已全部自包含，不再依赖具体项目）。
- **Rust ↔ Go**：`tech/languages/rust.md` 与 `tech/languages/go.md` 结构对齐（语言内核 / 工程实践），可横向对照所有权 vs GC、trait vs 接口、`Result` vs `error`、async runtime vs GMP 调度；Go 页额外深挖 runtime。
- **同类系统对比**：`codex-harness` 与 `deepseek-harness` 同为 agent harness。Codex 是 Rust 单体 + 统一协议，DeepSeek 是 TypeScript 全插件化 + 日志即 API，可从「控制循环 / steering / 工具审批」三条线横向比较。
- **部署视角**：`tech/data-infra/fluss.md` §13 部署与运维 ↔ `tech/cloud-native/kubernetes.md` 第八章节点管理。

---

## 待建页面（backlog）

- `tech/agent-harness/index.md`：两个 harness 的对比导读（loop / steer / 工具审批 / 多 agent），升格为领域 MOC 的触发条件是本领域超过 5 篇。
- `tech/agent-harness/mcp.md`：MCP 协议与工具生态。
- `tech/skills/claude-skills.md`：待 `raw/skills/` 资料入库后整理。
- `law/`、`finance/`：待原始资料入库后按 `AGENTS.md` §3 建目录。

---

## 登记规范（自查用）

1. 新增或重命名 wiki 页面后，必须同步更新本文件对应表格。
2. 链接必须可解析，且指向 `wiki/` 内的相对路径。
3. 每行至少给出「一句话」，不写「详见原文」这类空描述。
4. 页面 front-matter 的 `domain` 必须与所在目录一致（如 `tech/languages`）。
