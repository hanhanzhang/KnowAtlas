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
├── AGENTS.md              # Agent规则：根据raw资源整理wiki
├── CLAUDE.md              # 只有一行：@AGENTS.md
├── raw/                   # 原始资产层（只增不改）
│   ├── skills             # 收集社区高热度skill 
│   ├── sources/           # 剪藏、论文、外部资料原文
│   └── assets/            # 图片、附件
├── wiki/                  # 编译知识层（可增量修订）
│   ├── index.md           # 总索引
│   └── domains/           # 领域知识（技术框架、领域skill等）
└── .claude/
    └── skills/            # Agent 技能定义，每个技能一个 SKILL.md
