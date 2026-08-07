# 04 — 接通 R5 决策边界与最终结果

**What to build:** 用锁定版本的真实 R5 适配器替换端到端骨架中的 R5 测试替身，使任何合法 VLM 中间结果只通过 `risk_type + evidence` 得到唯一的二元结果和 10 类 `vlm_advice`。

**Blocked by:** 01 — 建立可执行的端到端契约骨架；03 — 锁定 taxonomy、四字段 Schema、canonical actor Prompt 与数据制作 Prompt

**Status:** ready-for-agent

- [ ] R5 入参只包含 `risk_type` 和 `evidence`，不会收到 `description`、`cot` 或 actor Prompt 内容。
- [ ] R5 成功响应能够通过既有业务包装契约暴露二元结果与 10 类 `vlm_advice`，不新造二元字段语义。
- [ ] `vlm_confidence` 可以保留为内部结果，但不是端到端成功响应的强制验收字段。
- [ ] R5 版本随每次调用和训练样本记录，版本不一致被识别为基础设施错误。
- [ ] 超时、不可用、非法响应和版本不一致不会被转换为 good、risk 或任一业务分类。
- [ ] 固定 fixture 验证相同 `risk_type/evidence` 产生可复现 R5 结果。
- [ ] 端到端测试证明最终业务结果只有 R5 一个来源。

