# mattpocock/skills

- 来源：https://github.com/mattpocock/skills
- 抓取日期：2026-10-30 · 版本 `1.3.2`（package.json）· MIT
- 作者：Matt Pocock

## 定位

「给真工程师用的 agent skills」，小而可组合，明确反对 GSD/BMAD/Spec-Kit 那种「拥有流程」的重框架。核心是治 AI 写码的四种失败模式：没听懂需求、太啰嗦、代码跑不通、堆出泥球。

## 目录结构

```
skills/engineering/          20 个（promoted）：ask-matt, code-review, codebase-design,
  diagnosing-bugs, domain-modeling, grill-with-docs, implement, implement-spec,
  improve-codebase-architecture, pr, prototype, research, retro, setup-matt-pocock-skills,
  tdd, to-spec, to-tickets, triage, wayfinder, wizard
skills/productivity/         7 个（promoted）：grill-me, grilling, handoff, teach,
  to-questionnaire, wait-what, writing-for-agents
skills/misc/ 4 · skills/in-progress/ 7 · skills/deprecated/
docs/<bucket>/<skill>.md     与 promoted skill 一一镜像的人读文档
.agents/                     adr/（2 篇）、invocation.md、writing-docs.md、install-block.md
.out-of-scope/               8 篇被拒需求记录
.claude-plugin/（plugin.json + marketplace.json）· agents/openai.yaml（每个 skill 旁一份）
scripts/（link-skills.sh, list-skills.sh, sync-plugin-version.mjs）
```

## 核心机制（带路径）

- `.agents/invocation.md`：定义 **user-invoked vs model-invoked**。user-invoked 设 `disable-model-invocation: true`（Claude）+ `policy.allow_implicit_invocation: false`（Codex 的 `agents/openai.yaml`），只能人敲 `/xxx`；model-invoked 保留 "Use when…" 触发语。规定「skill 间依赖要写 `Call the Skill tool with "grilling"`，不跨目录 `../xxx.md` 引用，不写裸 `/name`」。
- `skills/engineering/ask-matt/SKILL.md`：路由图。主流程 `grill-with-docs → to-spec → to-tickets → implement/implement-spec → tdd → code-review → pr → retro`；on-ramp：`triage`、`diagnosing-bugs`；超大工程走 `wayfinder`（决策票据地图，产出「决策」不是「交付物」）；底层词汇 `domain-modeling`、`codebase-design`（深模块）。
- `skills/engineering/grill-with-docs/SKILL.md` 正文仅一句：「Call the Skill tool twice, for "grilling" and "domain-modeling"」——用组合代替复制。
- `SCOPE.md` + `.out-of-scope/`：issue 必须满足「真实会话里发生过的失败」+「契合哲学」两条才受理；`new-skills.md` 明确不再新增，能由既有 skill 组合的行为不设新 skill。
- `AGENTS.md`：promoted skill 必须有 README 链接、plugin.json 条目、docs 页；`ask-matt` 是「会撒谎的路由器」需随变更同步；全仓禁 em-dash。

## 能做什么

- 一套可复制的「skill 仓库治理规范」（invocation 语义、docs 镜像、out-of-scope、ADR、changeset 发布）。
- 从对齐需求到合并的完整命令集：`/grill-with-docs`（grilling 面试 + 建 GLOSSARY.md/ADR）、`/to-spec`、`/to-tickets`（tracer-bullet + blocking edges）、`/implement-spec`（任务图 + 并行 frontier）、`/tdd`、`/code-review`（Standards+Spec 双轴）、`/retro`（改环境不改代码）。
- `writing-for-agents`：写 AGENTS.md/CLAUDE.md/skill 的方法论。

## 边界与局限

- 偏个人工作流，强假设（issue tracker、GLOSSARY.md、`.scratch/` 本地票据）需先 `/setup-matt-pocock-skills`。
- 技能耦合深，单独抽一个用会缺前置（如 grill-with-docs 依赖 grilling + domain-modeling）。
- 无自动化校验脚本，靠模型遵守措辞精确的 SKILL.md。

## 可复用点

调用权显式分离（user/model-invoked）、router 技能（ask-matt）、「组合代替复制」、docs 与 skill 镜像、out-of-scope 决策留痕。
