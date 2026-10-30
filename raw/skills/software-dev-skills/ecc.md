# affaan-m/ECC

- 来源：https://github.com/affaan-m/ECC
- 抓取日期：2026-10-30 · 版本 `2.2.3`（.claude-plugin/plugin.json）· MIT
- 作者：Affaan Mustafa

## 定位

「Agent harness 操作系统」：一次安装获得 71 agents + 313 skills + 98 命令 + hooks + rules + memory + AgentShield。宣称 `plan → test → implement → review → verify → remember → improve`，把重复经验沉淀为可复用能力。

## 目录结构

```
skills/          313 个（tdd-workflow, security-review, coding-standards,
                 python-testing, django-*, laravel-*, springboot-*, continuous-learning-v2, ...）
agents/          71 个（code-reviewer, planner, tdd-guide, build-error-resolver,
                 go-reviewer, java-build-resolver, ... 每种带 tools/model/描述）
commands/        98 个 slash 命令 shim（迁移中）
rules/           22 个语言/框架目录（common + typescript/python/golang/rust/swift/php/...）
hooks/           hooks.json + codex-hooks.json + qoder-hooks.json + hermes/ + memory-persistence/
manifests/       7 个（context-packs, context-profiles, install-components, install-profiles, ...）
scripts/         60+ 个（install/repair/doctor/sync-ecc-to-codex.sh/build-opencode.js/
                 gemini-adapt-agents.js/harness-audit.js/session-inspect.js/...）
.claude-plugin/ .codex/ .opencode/ .cursor/ .gemini/ .zed/ .kimi/ .pi/ ...  多 harness 适配
contexts/        dev.md research.md review.md（上下文 profile）
```

## 核心机制（带路径）

- `hooks/hooks.json`：每条 hook 内联极长 Node bootstrap（解析 `CLAUDE_PLUGIN_ROOT`、WSL 路径、plugin cache 多候选）后转发 `scripts/hooks/*.js`；用 `run-with-flags.js` 做 minimal/standard/strict 三档。`plugin.json` 的 `userConfig` 暴露 `hooks_enabled` 与 `hook_profile`，把 hook 当可开关的运行时。
- `skills/continuous-learning-v2/SKILL.md`：**instinct** = 原子行为 + `confidence 0.3–0.9` + `domain` + `scope(project/global)` + evidence。PreToolUse/PostToolUse 观察（100% 可靠，替代 v1 的 Stop hook）→ 后台 Haiku 分析 → 聚类 → 进化成 skill/command/agent。v2.1 用 git remote 做项目隔离，跨 2 项目才提升 global。
- `agents/*.md`：每个带「Prompt Defense Baseline」（防角色篡改/注入/泄密）+ 置信度过滤（>80% 才报；HIGH/CRITICAL 必须给行号 + 失败场景）。
- 多 harness：`sync-ecc-to-codex.sh`、`build-opencode.js`、`gemini-adapt-agents.js` 等从根目录**生成**适配，根目录是唯一真源。
- AgentShield：扫描 prompt/hook/MCP 配置/权限/密钥/agent 文件的安全扫描。

## 能做什么

- 一次安装获得规划/评审/构建修复/安全/架构/多语言专家代理。
- hooks 做质量与安全拦截（如 gateguard-fact-force、doc-file-warning、suggest-compact）。
- instincts 把重复经验沉淀；rules 按语言/项目选择性加载；contexts 提供 dev/research/review 三套上下文 profile。
- 支持 Claude/Codex/OpenCode/Cursor/Gemini/Zed/Copilot/Qwen/Kimi 等（能力受限适配，非等价）。

## 边界与局限

- 体量巨大、上下文与认知负担高；313 个 skill 质量不均，存在重复。
- 营销叙事强（README 116 KB）；多 harness 是「capability-limited adapter」，功能不等价。
- hook 链复杂（每条内联 bootstrap），排障成本高；instinct 学习依赖 hook 与后台模型。

## 可复用点

hook 三档开关（userConfig）、instinct 置信度 + 项目隔离的持续学习、Prompt Defense Baseline、根目录真源 + 生成适配器、多语言规则目录。
