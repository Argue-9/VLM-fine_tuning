# 06 — 贯通原始样本到正式参考答案

**What to build:** 建立可重复执行的数据生产纵向链路，使包含媒体、MD5、来源和 `human_label` 的原始记录生成 `candidate_*`，经严格检查后成为 `ref_*`，或以明确原因进入 `r5_conflict` 和其他隔离状态。

**Blocked by:** 05 — 完成数据标注模型同集试点

**Status:** ready-for-agent

- [ ] 原始记录缺少媒体、MD5、来源或合法 10 类 `human_label` 时被拒绝并记录原因。
- [ ] candidate 保留四字段原始生成结果、解析状态、生成后端和数据制作版本，不会直接覆盖参考答案。
- [ ] 晋升检查覆盖严格 JSON、四字段类型、46 类枚举、evidence 枚举、空集合规则和 CoT 机械自洽性。
- [ ] 只有参考 `risk_type/evidence` 经锁定 R5 得到的 `vlm_advice == human_label` 时，样本才可进入正式参考集合。
- [ ] R5 不一致样本进入 `r5_conflict`，且其真实 evidence 不会为了命中 R5 被静默删改。
- [ ] 每条 `ref_*` 可追溯媒体、MD5、来源、`human_label`、候选响应、三个 Prompt/Schema 版本、R5 和媒体管线版本。
- [ ] 对同一冻结输入重新运行验证不会产生不同的晋升或隔离状态。
- [ ] 试点数据能够从原始记录完整走到 ref 或 conflict，并生成数量与失败原因报告。

