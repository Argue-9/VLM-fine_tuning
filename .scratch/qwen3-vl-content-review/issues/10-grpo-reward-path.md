# 10 — 贯通 GRPO 奖励计算链路

**What to build:** 对合法和非法 completion 端到端计算已确认的渐进格式、严格语义门、risk type、evidence、R5、CoT 和长度奖励，并把外部 R5 故障与模型得分严格分离。

**Blocked by:** 03 — 锁定 taxonomy、四字段 Schema、canonical actor Prompt 与数据制作 Prompt；04 — 接通 R5 决策边界与最终结果；06 — 贯通原始样本到正式参考答案

**Status:** ready-for-agent

- [ ] `r_fmt` 对 J/K/T/E/C 的五级累计公式通过精确数值 fixture。
- [ ] `g=JKTE`；`g=0` 时不计算语义分且不调用 R5，字段顺序错误只影响格式分。
- [ ] 总奖励严格使用 format/type/evidence/R5/cot 的 0.20/0.25/0.35/0.15/0.05 权重。
- [ ] `r_type` exact-match 与 `r_ev` 双空、单空、部分交集、完全匹配 set-F1 均通过边界测试。
- [ ] `r_cot` 的 coverage、no-extra、type-mention、order-ok 四个二值项及空 evidence 规则通过测试。
- [ ] 软超长惩罚最大扣分为 0.03，边界值可由合法参考数据的 P95/P99 注入。
- [ ] 奖励中不存在 description 在线语义分、重复惩罚、类别倍率、R5 置信度、二元结果或 R5 内部路径分。
- [ ] R5 超时、不可用和版本错误触发批次跳过或中止及报警，不产生模型零分。

