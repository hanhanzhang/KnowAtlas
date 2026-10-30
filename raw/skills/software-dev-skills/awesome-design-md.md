# VoltAgent/awesome-design-md

- 来源：https://github.com/VoltAgent/awesome-design-md
- 抓取日期：2026-10-30 · MIT
- 作者：VoltAgent

## 定位

不是 skill，是**设计语料库**：84 份从真实网站提炼的 `DESIGN.md`，复制进项目后让 AI agent 生成视觉一致的 UI。基于 Google Stitch 提出的 `DESIGN.md` 概念（纯文本设计系统）。

## 目录结构

```
design-md/<site>/{DESIGN.md, README.md}   84 个站点（apple, airbnb, linear.app, claude,
                                          ferrari, bugatti, coinbase, figma, framer, ...）
```

## 核心机制（带路径）

- `DESIGN.md` 结构（以 `design-md/apple/DESIGN.md` 为例）：YAML front-matter 定义 `colors`（语义命名 + hex，如 `primary: "#0066cc"`、`canvas`、`ink`、`surface-tile-*`）与 `typography`（每个角色 fontFamily/size/weight/lineHeight/letterSpacing 完整数值），正文用自然语言写布局、组件、动效与「一条签名交互色」等设计语言。
- 分工声明：`AGENTS.md` 管「怎么建」，`DESIGN.md` 管「长什么样」，二者并列放项目根，任何编码 agent / Google Stitch 直接读懂，无 Figma/JSON schema/工具依赖。
- `README.md` 按行业分组（AI & LLM、金融、汽车、消费…）提供 `getdesign.md/<site>/design-md` 的展示链接。

## 能做什么

- 作为风格参考库 / few-shot 语料直接复制进项目；也作为写 DESIGN.md 的模板。
- 展示 `DESIGN.md` 这种「给设计 agent 读的纯文本设计系统」的完整形态。

## 边界与局限

- 无执行逻辑、无校验，是风格快照，不保证与站点现状一致。
- 直接套用有品牌混淆与版权风险；仓库自身是 awesome 列表 + 展示站引流（getdesign.md）。

## 可复用点

DESIGN.md 的语义 token + 类型角色表格式、AGENTS.md/DESIGN.md 分工、纯 markdown 零工具依赖。
