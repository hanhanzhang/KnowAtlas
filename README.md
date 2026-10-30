# KnowAtlas

个人知识图谱库，系统记录 AI Skill 与各领域知识，帮助我沉淀、连接和快速检索所学内容。

---

## 仓库是什么

KnowAtlas 是用 GitHub 管理的个人知识库，服务于两个读者：

1. **我自己**：长期沉淀技术框架、AI Skill、各领域知识，并能快速检索。
2. **AI Agent**：按照仓库规则，根据 `raw/` 中的原始资源整理 `wiki/`，而不是随意改写。

核心原则：

- `raw/` 保存原始资料，只增不改。
- `wiki/` 保存我或 Agent 消化后的编译知识，可持续修订。
- `AGENTS.md` 是 Agent 协作规则的唯一来源，核心规则是：**根据 raw 资源整理 wiki**。
- `CLAUDE.md` 只负责导入 `AGENTS.md`，避免多工具规则分裂。

---

## 目录结构

```text
KnowAtlas/
├── README.md              # 知识库职责
├── AGENTS.md              # Agent 规则：分类体系、页面类型、raw→wiki 工作流
├── CLAUDE.md              # 只有一行：@AGENTS.md
├── raw/                   # 原始资产层（只增不改）
│   ├── skills/            # 社区高热度 Skill 的原始资料、来源、说明
│   ├── sources/           # 剪藏、论文、外部资料原文
│   └── assets/            # 图片、附件
├── wiki/                  # 编译知识层（可增量修订）
│   ├── index.md           # 总索引：大类地图 + 全部页面
│   └── tech/              # 大类：技术
│       ├── languages/     #   领域：语言与工程
│       ├── agent-harness/ #   领域：Agent 运行时
│       ├── skills/        #   领域：Agent 技能
│       ├── cloud-native/  #   领域：云原生
│       └── data-infra/    #   领域：数据基础设施
└── .claude/
    └── skills/            # Agent 技能定义，每个技能一个 SKILL.md
```

## 分类体系

`wiki/` 按三层组织，深度封顶三层：

```text
wiki/<大类 branch>/<领域 area>/<主题 topic>.md
```

| 大类 | 含义 | 当前状态 |
|---|---|---|
| `tech` | 技术：语言、框架、系统架构 | 有内容 |
| `law` | 法律 | 预留 |
| `finance` | 金融 | 预留 |

每个页面带 YAML front-matter（`title / domain / type / tags / source / updated`），`type` 区分 `summary`（raw 提炼的摘要）与 `deep-dive`（自有深度长文）。`wiki/index.md` 是唯一入口，未登记的页面视为不存在。

## 常用操作

| 我想做什么 | 怎么做 |
|---|---|
| 查某个主题 | 先读 `wiki/index.md`，再点进对应页面 |
| 沉淀新资料 | 原文丢进 `raw/sources/` 或 `raw/skills/`，让 Agent 按 `AGENTS.md` 第 5 节整理 |
| 迁入已写好的长文 | 按 `AGENTS.md` 第 6 节：直接进 `wiki/<大类>/<领域>/`，补 front-matter |
| 新增 Agent 技能 | 在 `.claude/skills/<skill-name>/SKILL.md` 定义，格式见 `AGENTS.md` 第 8 节 |
