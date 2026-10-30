# cloudflare/security-audit-skill

- 来源：https://github.com/cloudflare/security-audit-skill
- 抓取日期：2026-10-30 · MIT
- 作者：Cloudflare

## 定位

把通用编码 agent 变成「安全审计员」的单 skill。六阶段编排：侦察→覆盖式狩猎→候选验证→结构化输出→独立记录复核→目标中立报告。是 Cloudflare「Build your own vulnerability harness」博客所描述漏洞发现 harness 的单仓起点。

## 目录结构

```
skills/security-audit/
  SKILL.md                    192 行：模式/术语/通用执行安全/写隔离/核心原则
  RECONNAISSANCE.md           阶段 1 侦察
  HUNTING.md                  阶段 2 覆盖式狩猎
  ATTACK-CLASSES.md           核心/通配/显性攻击类
  MEMORY-SAFETY-AND-BINARY.md 内存安全/二进制/内核
  AI-AND-LLM.md               prompt 注入/agent/输出处理
  WEB-PROTOCOL-AND-AUTH.md    HTTP 帧/缓存/认证协议
  CLIENT-SIDE.md              DOM 注入/消息信任/UI 欺骗/原型污染
  SUPPLY-CHAIN-AND-RELEASE.md 依赖/CI/发布/签名/更新/插件
  CLOUD-AND-DEPLOYMENT.md     IAM/IaC/容器/无服务器
  PROTOCOLS-RPC-AND-MESSAGING.md
  RESOURCE-EXHAUSTION-AND-AVAILABILITY.md
  DATA-ISOLATION-AND-LIFECYCLE.md
  DESKTOP-MOBILE-AND-LOCAL-IPC.md
  VALIDATION-AND-REPORTING.md 阶段 3–6
  report-schema.json          findings.json 的 JSON Schema
  validate-findings.cjs / validate-coverage-ledger.cjs  零依赖校验器（含 .test.cjs）
```

## 核心机制（带路径）

- `SKILL.md` 分 **guidance mode / full audit mode**：默认只答问题；只有明确「审计/渗透测试/要报告」才跑六阶段并落盘。
- 六阶段：侦察写 `architecture.md` + `coverage-ledger.json` → 按 ledger 单元派隔离 hunter → 每个候选给**全新 verifier 去证伪** → 写 `findings.json`（confirmed / needs_validation / rejected 三态，各有严格定义）→ 新 agent 复核来源 → 生成 `REPORT.md` / `FINDINGS-DETAIL.md` / `NEEDS-VALIDATION.md`。
- 写隔离（`SKILL.md` § Write isolation）：parent 独占共享文件；每个 agent 只有 `scratch/` 可写；产物 promotion 有 11 步 no-follow / fstat / link count==1 / 字节上限校验。无 OS 级沙箱就不执行目标代码，只记 `needs_validation`。
- 判定规则：只有「确认了真实信任边界破坏」才 confirmed；severity 必须 likelihood × impact；「防御纵深缺口不是漏洞」；「finding 的验证者永远不能是发现者」。
- 多次运行增量：用 prior ledger/findings 补缺口，作者称单次运行约找到重复运行总量的一半。

## 能做什么

- 审计任意仓库/API/服务/CLI/库/daemon，产出带来源证据、安全复现、优先级、最小修复的报告。
- 覆盖账本（coverage-ledger.json）可复现、可增量；报告目标中立（不针对具体漏洞扫描器）。
- 校验器零依赖（Node），JSON Schema 约束输出结构。

## 边界与局限

- 要求强沙箱 + 并行子代理 + 强模型；无沙箱时验证退化为 needs_validation。
- 领域攻击类文件是 prompt 模板而非工具，效果依赖模型安全知识。
- 面向「有真实信任边界」的目标，对边界模糊的目标产出有限。

## 可复用点

guidance/full 双模式、覆盖账本、对抗验证（发现者≠验证者）、三态 verdict、写隔离与产物 promotion 安全、schema + 校验器兜底。
