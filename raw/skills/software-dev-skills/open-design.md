# nexu-io/open-design

- 来源：https://github.com/nexu-io/open-design
- 抓取日期：2026-10-30 · Apache-2.0（skills 目录，部分子目录另有 LICENSE）
- 作者：nexu-io

## 定位

「第一个协作式 AI 设计 agent 工作台」：本地优先、开源，团队与 AI agent 一起做动效、原型、网站、App、仪表盘、PPT、图片、视频。既是产品（daemon + apps），也是技能库（171 skills）+ 设计系统库（154 个）+ 渲染模板库（116 个）。

## 目录结构

```
skills/              171 个功能技能（SKILL.md + assets/ + references/）
design-systems/      154 个设计系统包（manifest.json + DESIGN.md + tokens.css；apple 有 17 种语言 DESIGN.md）
design-templates/    116 个渲染模板（PPT/deck/文档/图片/视频/音频）
craft/               13 篇通用工艺参考（typography, color, anti-ai-slop, animation-discipline, ...）
plugins/             registry + spec + 官方/社区插件
apps/ packages/      daemon、web、SDK 等产品代码
docs/skills-protocol.md  技能协议规范
specs/current/skills-and-design-templates.md  技能与模板分类规则
```

## 核心机制（带路径）

- `docs/skills-protocol.md`：在 Claude `SKILL.md` 基础上加可选 `od:` 命名空间——`mode`（utility/prototype/deck/template/design-system/image/video/audio）、`surface`、`scenario`、`design_system.requires`、`craft.requires`（如 `[typography, color, anti-ai-slop]`）、`critique.policy`。全部可选，**零配置兼容**老 skill；缺失字段有明确默认值（mode 从描述推断回退 prototype，design_system.requires 默认 true）。
- 发现与优先级：user-managed 覆盖 bundled，同名 shadow 不删除；每次请求重扫（无 watcher/SIGHUP/重启）；运行时把激活 skill 真实复制到项目 `.od-skills/`，规避跨设备 symlink/rename 差异。
- `craft/anti-ai-slop.md`：把「AI 味」写成可 lint 的七宗罪（默认 Tailwind indigo、紫蓝信任渐变、emoji 当图标、编造指标、lorem ipsum…），与 `apps/daemon/src/lint-artifact.ts` 的 `AI_DEFAULT_INDIGO` 常量同步。
- `design-systems/<slug>/` 统一 `manifest.json + DESIGN.md + tokens.css`，manifest 用固定 schema（schemaVersion/id/category/source/files）；旧版只有 DESIGN.md 的目录仍兼容（fallback）。

## 能做什么

- 设计 agent 工作台（协同、反馈、迭代）+ 一套可复用的「技能注册协议」参考实现。
- 171 个 skill 覆盖 PPT、海报、视频、Figma、GSAP、3D、社交卡片、FAL/Replicate/Venice 生成等。
- 154 个设计系统包可直接注入 agent prompt 做品牌一致 UI。

## 边界与局限

- 仓库约 3.6 GB，产品与技能强绑定；`od:` 扩展只对自家 daemon 有意义。
- 大量 skill 质量参差；design-systems 与 awesome-design-md 高度同源（都是 DESIGN.md 思路）。
- skills 与 design-templates 有明确分类规则（功能 vs 渲染形状），但边界需读 spec 才清楚。

## 可复用点

`od:` 可选命名空间做零配置兼容、skill 注册协议（发现/优先级/运行时 staging）、anti-ai-slop 可 lint 化、design-system 包三件套格式。
