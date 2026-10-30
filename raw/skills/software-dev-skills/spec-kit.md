# github/spec-kit

- 来源：https://github.com/github/spec-kit（中文 README：README.zh-CN.md）
- 抓取日期：2026-10-30 · MIT
- 作者：GitHub

## 定位

开源工具套件，为 AI 编码助手提供「结构化流程 + 可复用模板 + 有据可查的制品」。核心是 Spec-Driven Development（SDD）：先明确做什么/为什么，再决定怎么做，规格是唯一真源，代码是规格的生成产物。三条独立入口：SDD、缺陷修复（bug）、想法评估（assess）。

## 目录结构

```
src/specify_cli/            Python CLI（`specify`）
  integrations/             67 个 harness 适配器（claude, codex, copilot, cursor_agent,
                            gemini, opencode, devin, droid, amp, cline, ...）
  extensions/ presets/ bundles/ workflows/ events/ mcp_server/
templates/                  spec/plan/tasks/checklist/constitution 模板 + commands/ 10 个命令模板
extensions/                 bug, assess, git, github, selftest, template, agent-context
presets/                    lean, scaffold, constitution-sync, self-test
bundles/                    assess, bugfix
workflows/                  speckit, assess, bugfix
spec-driven.md              SDD 方法论长文（The Power Inversion / 九条宪法）
```

## 核心机制（带路径）

- `src/specify_cli/integrations/`：67 个适配器把同一份 canonical 命令模板渲染成各 harness 的 skill/命令。`claude/__init__.py` 的 `registrar_config = {dir: ".claude/skills", format: "markdown", args: "$ARGUMENTS", extension: "/SKILL.md"}`；维护 `CANONICAL_TO_NATIVE` 事件名映射。真实取舍记录在注释：`analyze` 曾用 `context: fork`，因返回 300–500 行报告反撑爆上下文被撤（#3185）。
- `templates/commands/implement.md`：SOP 级实现命令——检查 `checklists/` 完成率、按技术栈补 `.gitignore/.dockerignore`、按 Phase 执行、完成后在 tasks.md 打勾，结尾强制跑 `.specify/extensions.yml` 的 `after_implement` 钩子（optional / mandatory 区分，mandatory 必须真执行）。
- `templates/commands/specify.md`：spec 命令，含 `handoffs`（label/agent/prompt）与 `scripts:`（sh/ps/py 三种模板解析），Pre-Execution 检查 `extensions.yml` 的 `hooks.before_specify`。
- `extensions/bug/extension.yml`：声明式 schema（`provides.commands` 带 name/file/description），注册 `speckit.bug.assess/fix/test` 与产物目录 `.specify/bugs/<slug>/`，报告结论三态 `verified/partial/failed`。
- `spec-driven.md`：SDD 哲学——「Power Inversion」（代码服从规格）、九条宪法（The Nine Articles）、Phase -1 Pre-Implementation Gates、File Creation Order。

## 能做什么

- `specify init <proj> --integration <agent>` 一条命令把「宪法→规格→计划→任务→实现→收敛」装进任意助手，聊天里逐条 `/speckit-constitution /speckit-specify /speckit-plan /speckit-tasks /speckit-implement /speckit-converge`。
- 扩展/预设/工作流/捆绑包四层定制；bug 修复（assess→fix→test）与想法评估（intake→research→define→shape→decide，产出 go/needs-clarification/kill）两条独立流程。
- 制品落盘 `.specify/`，可版本化、可演进。

## 边界与局限

- 制品多、流程重；存量项目落地成本高（现有项目有专门指南）。
- 命令模板塞满技术栈 if-else，维护面大；`context: fork` 等取舍说明其上下文成本敏感。
- 依赖 Python 3.11+ / uv，CLI 与助手技能是两套机制（CLI 装一次、项目 init 一次）。

## 可复用点

单一真源命令模板 + 多 harness 适配器、扩展的声明式 schema（extension.yml）、前置/后置钩子、制品目录约定（`.specify/`）。
