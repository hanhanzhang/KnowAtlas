# elayadesign/ai-design-skills

- 来源：https://github.com/elayadesign/ai-design-skills
- 抓取日期：2026-10-30 · MIT
- 作者：elayadesign

## 定位

AI 设计 skill 集合（目前仅 1 个：`landing-page-design`）。目标是「可自我独立安装、可 fork 定制」的设计 skill，而非框架。README 明确「每个 skill 是自包含文件夹，装你需要的即可」。

## 目录结构

```
skills/landing-page-design/SKILL.md   431 行，单文件
```

## 核心机制（带路径）

- `SKILL.md` 分两半：**Part A 策略与结构**（A1 intake 问题清单——目的/受众/素材/约束；A2 页面结构 12 段——首屏/中段/底部；A3 布局选型 4 类表），**Part B 视觉系统**（字体/间距/圆角/背景/hero 布局/图标/动效的硬规则）。
- 优先级明确：框架默认 < 本文件 < 用户明确要求（「When a rule here conflicts with a framework default, this file wins. When the user's explicit prompt conflicts with a rule, the user wins.」）。
- 设计值（fonts/colors/spacing/motion）集中在文件末尾一节；README 教用户「fork 后只改那一节，其余自动跟随」。
- intake 采用「一次批量问缺的字段，不逐个问；答不上就做合理假设、一行说明后继续」，避免阻塞。

## 能做什么

- 从零做高转化落地页；也用于编辑/改文案/评审。
- 多 harness 零配置：Cursor/Windsurf/Cline 只需把 SKILL.md 贴进规则文件；Claude.ai 上传到 Project。

## 边界与局限

- 单点场景（落地页/营销站/Web UI），无验证工具，全靠模型遵守措辞。
- 「规则赢过框架默认」在复杂项目里会与框架约定冲突。
- 作者另有一仓库 `redesign-skill` 处理「升级既有站点」（本仓库 README 链接过去）。

## 可复用点

单文件自包含、策略/视觉两段式结构、优先级显式声明、设计值集中在可 fork 段落、批量 intake + 假设即走。
