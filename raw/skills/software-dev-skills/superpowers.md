# obra/superpowers

- 来源：https://github.com/obra/superpowers
- 抓取日期：2026-10-30 · 版本 `v7.0.1`（`.claude-plugin/plugin.json`）· MIT
- 作者：Jesse Vincent / Prime Radiant

## 定位

给编码 agent 装一套「可执行的软件工程纪律」。以可组合的 skill 为底，靠 SessionStart hook 注入 bootstrap 让技能自动触发。定位是方法论 + 跨 harness 插件，不是语料。

## 目录结构

```
skills/                      15 个 skill，每个一个 SKILL.md
  brainstorming / writing-plans / executing-plans /
  subagent-driven-development / test-driven-development /
  systematic-debugging / requesting-code-review / receiving-code-review /
  using-git-worktrees / finishing-a-development-branch /
  verification-before-completion / dispatching-parallel-agents /
  writing-skills / diagnosing-superpowers / using-superpowers
hooks/                       session-start（bash）、hooks.json、hooks-cursor.json、run-hook.cmd
tests/                       按 harness 分类的 skill 行为回归（含 eval harness 对接）
.claude-plugin/ .codex-plugin/ .cursor-plugin/ .gemini/ .kimi-plugin/
.devin-plugin/ .hermes-plugin/ .muse-plugin/ .antigravity-plugin/ .opencode/ .pi/
docs/ scripts/ index.js（OpenCode v2 目录式插件入口）
```

## 核心机制（带路径）

- `hooks/hooks.json`：注册 `SessionStart`（matcher `startup|clear|compact`），命令执行 `hooks/run-hook.cmd session-start`。
- `hooks/session-start`：bash 脚本，把 `skills/using-superpowers/SKILL.md` 全文做 JSON 转义后注入 `additionalContext`。注释里写清不同 harness 字段名差异（Cursor 用 `additional_context`、Claude 用 `hookSpecificOutput.additionalContext`、Copilot/Muse 用顶层 `additionalContext`），并处理 Windows EPIPE、PATH 损坏等边界。
- `skills/using-superpowers/SKILL.md`：`<EXTREMELY-IMPORTANT>` + 12 条「合理化借口对照表」（"This is just a simple question" → "Questions are tasks"）强制先查 skill；给 Claude/Codex/Pi/Antigravity/Hermes/Muse 各留 `references/*-tools.md` 适配。
- `skills/subagent-driven-development/SKILL.md`：每个 task 派「全新上下文」implementer → 生成 review package → 派 task reviewer 做「spec 符合 + 代码质量」双审 → fix loop → 写 ledger。进度落盘 `<repo>/.superpowers/sdd/<plan>/progress.md`，理由原文：「conversation memory 不存活过 compaction，controller 丢位后重复派发已完成的整个 task 序列是观察到的最贵故障」。
- `skills/writing-plans/SKILL.md`：计划要求每个 task 有精确文件路径、完整代码、验证步骤；含 `Interfaces` 块（Consumes/Produces 精确签名），让 implementer 无需主会话上下文即可实现。
- `skills/writing-skills/SKILL.md`：把 skill 创作映射为 TDD（压力场景 subagent = test case，SKILL.md = production code，RED/GREEN/REFACTOR）。原文：「如果你没亲眼看过 agent 在没有 skill 时怎么失败，就不知道这个 skill 教得对不对」。

## 能做什么

- 给任意编码 agent 装一套完整 SDLC 纪律（头脑风暴→计划→子代理开发→TDD→评审→收尾）。
- 跨 14+ harness 安装（Claude/Codex/Cursor/Gemini/Kimi/Devin/Hermes/Muse/Antigravity/OpenCode/Pi/Qwen/Copilot/Grok）。
- `writing-skills` 是「怎么写好一个 skill」的方法论，含 TDD 验证。
- 自带 `tests/` 做 skill 行为回归（用 superpowers-evals 的 drill eval harness）。

## 边界与局限

- 强流程，token 成本高；对简单任务也要求先 brainstorming，易过度流程化。
- 自动触发强依赖 harness 的 hook / subagent 能力；`SessionStart` 注入在各 harness 表现不一。
- 官方声明不轻易接受新 skill 贡献，更新必须跨所有 harness 兼容。

## 可复用点

SessionStart 注入 bootstrap、skill 间依赖显式化、进度 ledger 抗 compaction、subagent 全新上下文 + 双审、skill 的 TDD 创作法。
