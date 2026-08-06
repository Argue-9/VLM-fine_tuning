# 13 — 完成 GRPO 最小可执行 tracer

**What to build:** 使用轻量兼容测试模型，从已验证 SFT adapter 贯通冻结 reference、同 Prompt 多 completion、分解奖励、优化更新、checkpoint 恢复和 TensorBoard 奖励监控，为正式 8 卡 GRPO 排除逻辑问题。

**Blocked by:** 10 — 贯通 GRPO 奖励计算链路；11 — 完成 SFT 最小可执行 tracer

**Status:** ready-for-agent

- [ ] GRPO 从 SFT adapter 初始化，并以其初始状态的冻结快照作为 reference。
- [ ] 最小运行使用同一个 actor Prompt 生成多 completion，完成奖励计算和至少两个 optimizer update。
- [ ] 有效 dropout 为 0，并验证 `beta=0.04`、`kl_in_reward=false`、`scale_rewards=batch`、`dynamic_sample=false`。
- [ ] 训练能够保存、恢复 GRPO adapter，并继续生成相同四字段契约。
- [ ] TensorBoard 记录总奖励、`r_fmt`、`r_type`、`r_ev`、`r_R5`、`r_cot`、长度惩罚、语义门通过率、KL、completion 长度和组内奖励方差。
- [ ] TensorBoard 单独记录零方差组比例、R5 调用量、R5 故障量、非法 JSON 和各类语义门失败原因。
- [ ] R5 故障注入会中止或跳过批次并留下监控事件，不会写入模型零奖励。
- [ ] event 文件可加载，训练 step 与奖励统计能够对应到同一次运行配置。

