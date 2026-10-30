---
title: "软件开发领域 Skill 全景：九库拆解与八种可复用模式"
domain: tech/skills
type: deep-dive
tags: [agent-skills, skill-design, sdd, tdd, design-system, security-audit, agent-harness]
source: "raw/skills/software-dev-skills/（9 个社区仓库源码分析，抓取于 2026-10-30）"
updated: 2026-10-30
---

# 软件开发领域 Skill 全景：九库拆解与八种可复用模式

> 基于 9 个社区仓库的源码分析（未只读 README，逐目录读取了 `SKILL.md`、`hooks.json`、`plugin.json`、`manifest.json`、`scripts/` 等关键文件）。
> 原始资料在 `raw/skills/software-dev-skills/`，本文是消化后的编译知识。
> 快照：2026-10-30。Star 为当日 GitHub API 返回值。

---

## 目录

- [1. 研究对象与分类](#1-研究对象与分类)
  - [1.1 九库清单](#11-九库清单)
  - [1.2 五种形态](#12-五种形态)
- [2. 逐库拆解](#2-逐库拆解)
  - [2.1 superpowers：方法论 + 跨 harness 插件](#21-superpowers方法论--跨-harness-插件)
  - [2.2 mattpocock/skills：精选工程 skill 集](#22-mattpocockskills精选工程-skill-集)
  - [2.3 spec-kit：命令模板编译到 67 个 harness](#23-spec-kit命令模板编译到-67-个-harness)
  - [2.4 cloudflare/security-audit-skill：单 skill 做到可验证](#24-cloudflaresecurity-audit-skill单-skill-做到可验证)
  - [2.5 ECC：agent harness 全家桶](#25-eccagent-harness-全家桶)
  - [2.6 open-design：技能注册协议 + 设计系统库](#26-open-design技能注册协议--设计系统库)
  - [2.7 hallmark：反 AI 味设计](#27-hallmark反-ai-味设计)
  - [2.8 ai-design-skills：单文件落地页系统](#28-ai-design-skills单文件落地页系统)
  - [2.9 awesome-design-md：设计语料库](#29-awesome-design-md设计语料库)
- [3. 八种可复用模式](#3-八种可复用模式)
  - [3.1 渐进披露](#31-渐进披露)
  - [3.2 门禁分层](#32-门禁分层)
  - [3.3 状态落盘](#33-状态落盘)
  - [3.4 对抗验证](#34-对抗验证)
  - [3.5 工具兜底](#35-工具兜底)
  - [3.6 调用权设计](#36-调用权设计)
  - [3.7 单一真源 + 适配器](#37-单一真源--适配器)
  - [3.8 治理流程](#38-治理流程)
- [4. 共同短板与选型建议](#4-共同短板与选型建议)
- [5. 与既有页面的关联](#5-与既有页面的关联)

---

## 1. 研究对象与分类

### 1.1 九库清单

| 仓库 | Star | 形态 | 一句话 |
|---|---|---|---|
| obra/superpowers | 308.7k | 方法论 + 15 skill + 跨 harness 插件 | 把 SDLC 固化成强制流程 |
| affaan-m/ECC | 292.8k | 全家桶运行时 | 313 skills + 71 agents + hooks + 记忆 |
| mattpocock/skills | 306.7k | 精选 skill 集 | 小而可组合，治四种失败模式 |
| github/spec-kit | 143.9k | CLI + 模板 + 扩展 | 规格驱动开发（SDD） |
| VoltAgent/awesome-design-md | 124.5k | 语料库 | 84 份 DESIGN.md |
| nexu-io/open-design | 104.9k | 产品 + 技能库 | 171 skills + 154 design systems |
| cloudflare/security-audit-skill | 52.9k | 单 skill | 六阶段安全审计 |
| Nutlope/hallmark | 42.2k | 单 skill + 工具链 | 反 AI 味设计 |
| elayadesign/ai-design-skills | 12.3k | 单 skill | 落地页设计系统 |

### 1.2 五种形态

这 9 个仓库「不是同一类东西」，按形态分五类：

1. **方法论 / 流程编排**：superpowers、mattpocock、spec-kit。把软件开发生命周期固化成 skill/命令序列，强调流程、制品与约束。
2. **全家桶运行时**：ECC。大规模 skills + agents + commands + hooks + rules，带持续学习与安全扫描。
3. **垂直领域深 skill**：cloudflare、hallmark、ai-design-skills。单个 skill 做深，配参考文件或脚本。
4. **语料 / 设计系统库**：awesome-design-md、open-design 的 design-systems。无执行逻辑，是供 agent 读取的风格数据。
5. **产品自带技能生态**：open-design。技能库是产品的一部分，附带技能注册协议与 daemon。

形态决定了它们的「可复用价值」完全不同，不能用一个模板去记录，也不能都当作「skill 写法」来学。

---

## 2. 逐库拆解

### 2.1 superpowers：方法论 + 跨 harness 插件

核心是把「头脑风暴→计划→子代理开发→TDD→评审→收尾」固化成 15 个可组合 skill，并用 `hooks/hooks.json` 的 `SessionStart`（matcher `startup|clear|compact`）注入 `using-superpowers/SKILL.md` 全文，让技能自动触发——这是它「自动生效」的机制本质，不是靠模型自觉。

三个值得记的工程细节：

- **进度 ledger 抗 compaction**：`subagent-driven-development` 要求进度落盘 `<repo>/.superpowers/sdd/<plan>/progress.md`，理由是「controller 丢位后重复派发已完成的整个 task 序列是观察到的最贵故障」。
- **计划的可执行性**：`writing-plans` 要求每个 task 有精确文件路径、完整代码、验证步骤，并带 `Interfaces` 块（Consumes/Produces 精确签名），让 implementer 无需主会话上下文。
- **skill 的 TDD 创作法**：`writing-skills` 把 skill 创作映射为 RED-GREEN-REFACTOR——先用 subagent 跑压力场景拿 baseline（RED），再写 skill（GREEN），最后补漏洞。原话「如果你没亲眼看过 agent 在没有 skill 时怎么失败，就不知道这个 skill 教得对不对」。

它还是唯一带 `tests/` 做 skill 行为回归的仓库（对接 superpowers-evals 的 drill eval harness）。

### 2.2 mattpocock/skills：精选工程 skill 集

定位「给真工程师用的 skill」，小而可组合，明确反对 GSD/BMAD/Spec-Kit 那种「拥有流程」的重框架。核心是治四种失败模式：没听懂需求、太啰嗦、代码跑不通、堆出泥球。

两个设计值得单独学习：

- **调用权显式分离**（`.agents/invocation.md`）：user-invoked（`disable-model-invocation: true` + `policy.allow_implicit_invocation: false`）只能人敲 `/xxx`；model-invoked 保留 "Use when…" 触发语给模型自动命中。skill 间依赖要写 `Call the Skill tool with "grilling"`，不跨目录 `../xxx.md` 引用。
- **router 技能**（`ask-matt/SKILL.md`）：一张路由图，主流程 `grill-with-docs → to-spec → to-tickets → implement/implement-spec → tdd → code-review → pr → retro`，两条 on-ramp（`triage`、`diagnosing-bugs`）和一条 `wayfinder`（超出单会话的大工程）。

治理层面（`AGENTS.md`、`SCOPE.md`、`.out-of-scope/`）规定：issue 必须满足「真实会话里发生过的失败」才受理；能由既有 skill 组合的行为不设新 skill；promoted skill 必须有 README 链接、plugin.json 条目、docs 页。这是「skill 仓库如何治理」的现成范本。

### 2.3 spec-kit：命令模板编译到 67 个 harness

核心是 Spec-Driven Development（SDD）：规格是唯一真源，代码是规格的生成产物。`src/specify_cli/integrations/` 有 **67 个适配器**把同一份 canonical 命令模板渲染成各 harness 的 skill/命令（`claude/__init__.py` 的 `registrar_config` 决定落盘目录、格式、参数占位符、扩展名）。

三个值得记的机制：

- **命令模板即 SOP**：`templates/commands/implement.md` 检查 `checklists/` 完成率、按技术栈补 `.gitignore/.dockerignore`、按 Phase 执行、完成后在 tasks.md 打勾，并强制跑 `after_implement` 钩子（optional/mandatory 区分，mandatory 必须真执行）。
- **扩展的声明式 schema**：`extensions/bug/extension.yml` 用 `provides.commands` 注册命令与产物目录 `.specify/bugs/<slug>/`，报告三态 `verified/partial/failed`。
- **真实取舍留痕**：`analyze` 曾用 `context: fork`，因返回 300–500 行报告反撑爆上下文被撤（#3185）。这说明它的模板是「被生产教训打磨过的」，不是一次性生成。

三条独立入口：SDD（constitution→specify→plan→tasks→implement→converge）、缺陷修复（assess→fix→test）、想法评估（intake→research→define→shape→decide，产出 go/needs-clarification/kill）。

### 2.4 cloudflare/security-audit-skill：单 skill 做到可验证

把一个通用编码 agent 变成「安全审计员」。六阶段：侦察（`architecture.md` + `coverage-ledger.json`）→ 覆盖式狩猎 → 候选验证 → 结构化输出（`findings.json` 三态）→ 独立记录复核 → 目标中立报告。

它的价值在于「可验证」三字，落到代码：

- **对抗验证**：验证 finding 的 agent 永远不是发现它的 agent；每个候选给全新 verifier 去证伪。
- **写隔离与产物 promotion**：parent 独占共享文件，每个 agent 只有 `scratch/` 可写，产物 promotion 要 11 步 no-follow / fstat / link count==1 / 字节上限校验。无 OS 级沙箱就不执行目标代码，只记 `needs_validation`。
- **判定规则即约束**：只有确认真实信任边界破坏才 `confirmed`；severity 必须 likelihood × impact；「防御纵深缺口不是漏洞」。
- **schema + 校验器兜底**：`report-schema.json` + 两个零依赖 Node 校验器（含测试），保证输出结构。

### 2.5 ECC：agent harness 全家桶

一次安装获得 71 agents + 313 skills + 98 命令 + hooks + rules + memory + AgentShield。定位「agent harness 操作系统」：`plan → test → implement → review → verify → remember → improve`。

值得记的两个机制（也是其差异化）：

- **instinct 持续学习**（`continuous-learning-v2/SKILL.md`）：把会话观察沉淀为「原子行为 + confidence 0.3–0.9 + domain + scope(project/global) + evidence」，PreToolUse/PostToolUse 观察 → 后台分析 → 聚类 → 进化成 skill/command/agent；v2.1 用 git remote 做项目隔离，跨 2 项目才提升 global。
- **hook 三档开关**：`plugin.json` 的 `userConfig` 暴露 `hooks_enabled` 与 `hook_profile`（minimal/standard/strict），把 hook 当可关的运行时而非硬编码；每条 hook 内联极长 Node bootstrap 解析 plugin 根路径。

但它的体量是双刃剑：313 个 skill 质量不均、存在重复，README 长达 116 KB，多 harness 是「capability-limited adapter」而非等价实现。**适合当作「全家桶天花板」的参考，而非直接照搬**。

### 2.6 open-design：技能注册协议 + 设计系统库

既是设计 agent 工作台，也是「技能注册协议」的参考实现。最值得抄的是 `docs/skills-protocol.md`：

- 在 Claude `SKILL.md` 上加可选 `od:` 命名空间（`mode`、`surface`、`design_system.requires`、`craft.requires`、`critique.policy`），全部可选，**零配置兼容**老 skill，缺失字段有明确默认值。
- 发现与优先级：user-managed 覆盖 bundled，同名 shadow 不删除；每次请求重扫，无 watcher；运行时把激活 skill 真实复制到项目 `.od-skills/`，规避跨设备 symlink 差异。
- `craft/anti-ai-slop.md` 把「AI 味」写成可 lint 的七宗罪（默认 Tailwind indigo、紫蓝信任渐变、emoji 当图标、编造指标…），与 daemon 的 `lint-artifact.ts` 常量同步。
- `design-systems/<slug>/` 统一 `manifest.json + DESIGN.md + tokens.css` 三件套。

### 2.7 hallmark：反 AI 味设计

单 skill（154 行）+ JS 工具链。三层门禁是精髓：**Floor** 16 条永不妥协（token 化颜色字体、禁 `transition: all`、必须有 `:focus-visible`、禁编造指标、首屏 1280×800 完整、CSS 首行必须是 stamp）；**Reflex** 可带理由豁免但必须留痕；**Finish** 只提示不阻塞。

另一个巧思是把「随机性」做成确定性工具：`theme.mjs deal` 发三套主题并写 `.hallmark/log.json` 防连续雷同，`draw` 定页面的 anatomy，`scaffold` 产 `tokens.css`；`sloplint --render` 在 8 个宽度真实浏览器 lint。hero 单独建、单独渲染、冷上下文评审，避免「先写整页再返工」。

### 2.8 ai-design-skills：单文件落地页系统

431 行单文件，分 Part A 策略（intake 问题清单、页面结构 12 段、布局选型表）与 Part B 视觉系统（字体/间距/圆角/背景/动效硬规则）。优先级显式声明：框架默认 < 本文件 < 用户明确要求。设计值集中在文件末尾一节，README 教用户「fork 后只改那一节」。是「单文件自包含 skill」的最小范本。

### 2.9 awesome-design-md：设计语料库

不是 skill，是 84 份从真实网站提炼的 `DESIGN.md`。结构是 YAML front-matter 定义语义 `colors` 与完整 `typography` 角色表，正文写布局/组件/动效语言。分工一句话说清：`AGENTS.md` 管怎么建，`DESIGN.md` 管长什么样。价值在于「给设计 agent 读的纯文本设计系统」的完整形态，以及作为写 DESIGN.md 的模板。

---

## 3. 八种可复用模式

### 3.1 渐进披露

`SKILL.md` 只放入口与流程，长知识进 `references/`，按需加载。几乎全部仓库都用，hallmark 尤其极致（`SKILL.md` 154 行，但 references 里有 50 套主题、21 种 macrostructure、gates、craft 等）。

### 3.2 门禁分层

把规则按「不可违 / 可豁免 / 只提示」分层：hallmark 的 Floor/Reflex/Finish，cloudflare 的 confirmed/needs_validation/rejected。硬规则要少而狠，软规则要留痕。

### 3.3 状态落盘

进度与结论写进磁盘而不是只留内存：superpowers 的 ledger、cloudflare 的 coverage-ledger.json + findings.json、spec-kit 的 `specs/`、hallmark 的 `.hallmark/log.json`。共同理由是「抗 compaction」——模型会忘，文件不会。

### 3.4 对抗验证

发现者 ≠ 验证者，reviewer 必须全新上下文：cloudflare 的「验证 finding 的 agent 永远不是发现它的 agent」，superpowers 的「每个 task 派全新上下文 implementer + 独立 reviewer」。这是防「自己给自己背书」的关键。

### 3.5 工具兜底

能用脚本判定的绝不靠模型自觉：hallmark 的 sloplint（真实浏览器 lint）、cloudflare 的 validate-*.cjs、superpowers 的 tests/、open-design 的 lint-artifact。规则要能被验证，否则是愿望清单。

### 3.6 调用权设计

user-invoked 与 model-invoked 显式分离（mattpocock）：人才能触发的流程（如 `/retro`）不让模型自动乱跑；模型能自主触发的（如 `/code-review`）保留触发语。这解决「skill 何时该自动触发」这个老大难问题。

### 3.7 单一真源 + 适配器

一份模板/技能编译到多个 harness：spec-kit 的 67 个 integration 适配器、ECC 的 `sync-ecc-to-codex.sh`/`build-opencode.js`、superpowers 的 14 套 `.xxx-plugin/`。根目录是唯一真源，适配层从它生成。

### 3.8 治理流程

skill 仓库本身需要治理：mattpocock 的 `.out-of-scope/`、ADR、changeset；spec-kit 的扩展 schema、presets/bundles/workflows 分层；superpowers 的「不轻易接受新 skill 贡献」。没治理的 skill 库会走向 ECC/open-design 式的重复与漂移。

---

## 4. 共同短板与选型建议

**共同短板**：

1. **缺客观评测**：只有 superpowers 有 `tests/` 雏形，其余依赖主观。
2. **自动触发不可靠**：靠 description 措辞命中，mattpocock 用 user-invoked 规避，superpowers 用 hook 注入规避，都说明「让模型自觉」不靠谱。
3. **仓库越大漂移越重**：ECC、open-design 有大量重复技能。
4. **营销叙事强于代码事实**：ECC（README 116 KB）、open-design（产品 + 技能强绑定）需谨慎分辨。

**选型建议**：

- 想学「怎么写好一个 skill / 一个 skill 库」：mattpocock（治理）→ superpowers（skill 的 TDD 创作法）→ hallmark（单 skill + 工具兜底的极致）。
- 想给团队装一套流程：spec-kit（重、制品化）或 superpowers（重、纪律化），二选一；mattpocock 适合个人且愿意先做 setup。
- 想学「可验证的垂直 skill」：cloudflare/security-audit-skill 是天花板。
- 想要现成设计语料/系统：awesome-design-md + open-design 的 design-systems。

---

## 5. 与既有页面的关联

- **agent-harness 系列**（`tech/agent-harness/codex-harness.md`、`deepseek-harness.md`）讲的是「运行时怎么跑」，本文讲的是「给运行时装什么能力（skill）」。skill 的自动触发（superpowers 的 SessionStart hook、ECC 的 PreToolUse hook）正是 harness 的 hook 机制在上层的应用。
- 后续整理单一 skill 的深挖时（如 `claude-skills` 写作规范、`security-audit` 的六阶段），建议在 `tech/skills/` 下另立页面，并在此处「交叉主题」登记。
