# 跨仓库对比

- 来源：本目录 9 个仓库的原始资料
- 抓取日期：2026-10-30

## 五种形态

| 形态 | 代表 | 特征 |
|---|---|---|
| 方法论 / 流程编排 | superpowers、mattpocock、spec-kit | 把 SDLC 固化成 skill/命令序列，强调流程与制品 |
| 全家桶运行时 | ECC | 大规模 skills+agents+commands+hooks+rules，带记忆与安全 |
| 垂直领域深 skill | cloudflare、hallmark、ai-design-skills | 单个 skill 做深，配参考文件或脚本 |
| 语料 / 设计系统库 | awesome-design-md、open-design 的 design-systems | 无执行逻辑，供 agent 读取的风格数据 |
| 产品自带技能生态 | open-design | 技能库是产品的一部分，附带注册协议与 daemon |

## 8 个可复用模式

1. **渐进披露**：`SKILL.md` 只放入口与流程，长知识进 `references/`，按需加载（几乎全部）。
2. **门禁分层**：Floor 不可违 / Reflex 可豁免但留痕 / Finish 只提示（hallmark、cloudflare 的 confirmed/needs_validation/rejected）。
3. **状态落盘**：ledger、findings.json、`specs/`、`.hallmark/log.json` 抗 compaction（superpowers、cloudflare、spec-kit）。
4. **对抗验证**：发现者 ≠ 验证者，reviewer 必须全新上下文（cloudflare、superpowers）。
5. **工具兜底**：能用脚本判定的绝不靠模型自觉（sloplint、validate-*.cjs、theme.mjs、lint-artifact）。
6. **调用权设计**：user-invoked 与 model-invoked 显式分离（mattpocock）。
7. **单一真源 + 适配器**：一份模板/技能编译到 67/14/10+ harness（spec-kit、ECC、superpowers）。
8. **治理流程**：`.out-of-scope/`、ADR、changeset、扩展 schema（mattpocock、spec-kit）。

## 共同短板

- 效果缺客观评测：只有 superpowers 有 `tests/` 雏形（eval harness），其余依赖主观。
- 自动触发靠 description 措辞，可靠性无法保证（mattpocock 用 user-invoked 规避，superpowers 用 hook 注入规避）。
- 仓库越大，重复与漂移越严重（ECC、open-design 有大量重复技能）。
- 营销叙事强于代码事实的倾向：ECC（README 116 KB）、open-design（产品 + 技能强绑定）需谨慎分辨。
