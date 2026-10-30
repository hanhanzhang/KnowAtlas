# Nutlope/hallmark

- 来源：https://github.com/Nutlope/hallmark
- 抓取日期：2026-10-30 · 版本 `2.0.0`（SKILL.md frontmatter）· MIT
- 作者：Nutlope / Together AI

## 定位

「反 AI 味」设计 skill：让编码助手生成的 UI 看起来像人设计的，而不是模型默认输出。一张硬底线（Floor）+ 发牌式方向（50 套主题）+ 真实浏览器 lint（sloplint）。

## 目录结构

```
skills/hallmark/
  SKILL.md                154 行：Floor + Flow（pre-flight→ask once→shape→direction→load→build→sweep）
  agents/                 评审子代理提示
  references/             themes/（50 套，每个 ≤8KB）、gates.md、anatomy.md、
                          macrostructures.md（21 种形状）、imagery.md、craft.md、
                          layout.md、genres/、direction.md、patterns.md、flow.md、...
  scripts/                theme.mjs（deal/draw/scaffold/photos/log）、sloplint.mjs、
                          sloplint-render.mjs、capture.mjs、refcheck.mjs、imagegen.mjs、
                          install.mjs、install-hook.mjs、lint-hook.mjs、photos.json（755 张图）
eval/  site/  docs/（50 套主题截图）
```

## 核心机制（带路径）

- `SKILL.md` 三层门禁：**Floor** 16 条永不妥协（颜色/字体必须 token 化、禁 `transition: all`、`overflow-x: clip`、必须有 `:focus-visible`、禁编造指标/引言/logo、首屏 1280×800 必须完整、CSS 首行必须是 stamp）；**Reflex** 可带理由豁免但必须留痕；**Finish** 只提示不阻塞。
- 流程（`SKILL.md` § Flow）：pre-flight 读项目（`design.md` 锁定的系统最高优先）→ 一次性问四件事（素材/受众/用途/语气，跳过项为 opt-out 并推断留痕）→ shape（材料优先 + 定 job/signature）→ `theme.mjs deal` 发三套主题（带 `.hallmark/log.json` 防连续雷同，支持 `--reroll`）→ 只加载命中的参考文件 → hero 单独建、单独渲染、冷上下文评审 → `sloplint --render` 在 8 个宽度真实浏览器检查 → 收尾材料清单。
- `theme.mjs`：`deal / draw / scaffold / photos / log` 把「随机性 + 去重 + 确定性脚手架」做成工具；`scaffold` 产出 `tokens.css` + `base.css` + stamp 骨架。
- `sloplint.mjs`：静态 + `--render`（Puppeteer）两级；`--before/--kept` 在 redesign 时记录并校验链接/表单/标题没丢；`--stage hero` 支持分阶段。

## 能做什么

- 生成「不像 AI」的网页/落地页；`/hallmark redesign <path>` 保内容保接线重构；`/hallmark audit <path>` 只报告；`custom` 模式脱离主题自由设计（仍受 Floor/gates/sweep 约束）。
- 被 Claude/Codex/Cursor/OpenCode 直接调用（`npx skills add nutlope/hallmark`）。

## 边界与局限

- 强绑定网页/营销页；渲染级检查依赖 Node 18+ 与 Chrome/Puppeteer。
- 50 套主题有明确审美偏好；custom 模式依赖模型设计能力。

## 可复用点

硬底线（不可违）+ 可豁免 Reflex + 只提示 Finish 的三层门禁、随机性做成确定性工具（theme deal/log 去重）、真实浏览器 lint 作为验证兜底、stamp 决策留痕。
