# 16 — 执行四档 GRPO 学习曲线并选择最终 actor

**What to build:** 从对应的正式 SFT 状态继续四档 GRPO，对比格式、核心字段、R5 结果、奖励稳定性和能力保持，选择最终生产 actor，而不引入未确认的数值验收线。

**Blocked by:** 10 — 贯通 GRPO 奖励计算链路；14 — 完成 8 卡 Qwen3-VL-8B GRPO smoke test；15 — 执行四档 SFT 学习曲线并选择 SFT checkpoint

**Status:** ready-for-agent

- [ ] 四档 GRPO 使用冻结的数据清单、相同 actor Prompt、taxonomy、媒体管线、奖励版本和确定性评估配置。
- [ ] 每档从可追溯的 SFT adapter 启动，保存初始 reference 标识和最终 GRPO adapter。
- [ ] TensorBoard run 独立且可横向比较，记录总奖励、五个正向分量、长度惩罚、KL、语义门通过率、零方差组、completion 长度、吞吐和显存。
- [ ] 训练监控证明没有把 R5 故障写成零奖励，也没有隐藏 description、重复、类别倍率或置信度奖励。
- [ ] 每档最终 adapter 在固定 dev、能力保持集和历史回归集上完成确定性评估。
- [ ] 报告同时比较 `risk_type`、evidence、R5 advice、四字段合法率、长度、重复率和能力保持表现。
- [ ] 选择最终 actor 的依据和取舍被记录，但不从历史数据推导新的 precision、recall 或 F1 硬门槛。
- [ ] 最终 adapter 能恢复并完成静态图片、GIF、严格 JSON 和 R5 端到端回归。

