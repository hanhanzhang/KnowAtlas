# AGENTS.md

核心工作流：**读取 `raw/` 中的原始资源，整理成编译知识写入 `wiki/domains/`，并更新 `wiki/index.md`。**

## 1. 启动顺序

按顺序读取：

1. `README.md`：了解仓库目的和目录职责。
2. `AGENTS.md`：了解规则。
3. `wiki/index.md`：了解已有知识入口，避免重复建页。

不要跳过启动顺序直接修改文件。

## 2. 目录职责

| 路径 | 职责 | 写入规则 |
|---|---|---|
| `raw/skills/` | 收集社区高热度 Skill 的原始资料、链接、说明 | 只增不改；保留来源、标题、日期 |
| `raw/sources/` | 剪藏、论文、外部资料原文 | 只增不改 |
| `raw/assets/` | 图片、附件 | 只增不改；控制体积 |
| `wiki/domains/` | 编译后的领域知识，包括技术框架、领域 Skill 等 | 可增量修订 |
| `wiki/index.md` | 总索引，登记所有领域知识页面 | 新增或重命名页面后更新 |
| `.claude/skills/` | Agent 可执行技能定义 | 每个技能一个 `SKILL.md` |

核心原则：**原始资料进 `raw/`，编译知识进 `wiki/`，Agent 可执行技能进 `.claude/skills/`。**

注意区分两类 Skill：

- `raw/skills/`：**资料**，记录社区高热度 Skill 是什么、来源在哪、原始说明。
- `.claude/skills/`：**能力**，定义 Agent 如何执行某个技能，含 `name`、`description` 和操作步骤。

不要把 `raw/skills/` 的原文直接复制成 `.claude/skills/` 下的技能；前者是素材，后者是规则。

## 3. 核心工作流：raw → wiki

当用户要求整理资料时，按以下步骤执行：

1. 从 `raw/skills/` 或 `raw/sources/` 读取目标原始资料。
2. 判断它属于哪个领域、哪个技术框架或哪类 Skill。
3. 在 `wiki/domains/` 下找到对应主题页；不存在则新建，命名用英文小写加连字符，如 `react-server-components.md`、`claude-skills.md`。
4. 将原始资料消化后写入主题页，而不是原文堆砌。至少包含：
   - 一句话概述
   - 关键要点
   - 适用场景与边界
   - 来源链接（指向 `raw/` 或外部原文）
5. 在 `wiki/index.md` 中登记或更新该页面的条目。
6. 向用户说明：读了哪些 raw 文件、写了哪个 wiki 页面、更新了索引的哪一行。

## 4. 技能路由

根据用户任务读取对应文件：

| 用户要做什么 | 读取/执行 |
|---|---|
| 整理 `raw/` 新资料 | 先读本文件第 3 节工作流；必要时读 `.claude/skills/ingest/SKILL.md` |
| 查询某个技术框架 | 先查 `wiki/index.md`，再读 `wiki/domains/` 下对应页面 |
| 查询某个领域 Skill | 先查 `wiki/index.md`，再读 `wiki/domains/` 下对应页面；原始资料在 `raw/skills/` |
| 新增 Agent 技能 | 在 `.claude/skills/<skill-name>/SKILL.md` 定义 |
| 维护索引 | `wiki/index.md` |

若目标文件不存在，先告知用户，不要凭空创建大量内容。

## 5. SKILL.md 约定

Agent 可执行技能放在：

```text
.claude/skills/<skill-name>/SKILL.md
