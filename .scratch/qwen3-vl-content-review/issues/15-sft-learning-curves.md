# 15 — 执行四档 SFT 学习曲线并选择 SFT checkpoint

**What to build:** 在已冻结的 2.5k、5k、7.5k、10k 数据清单上执行可比较的正式 SFT，使用统一评估和 TensorBoard 曲线判断数据边际收益，并选出进入 GRPO 的 SFT checkpoint。

**Blocked by:** 09 — 冻结 10,000 条训练数据与评估集合；12 — 完成 8 卡 Qwen3-VL-8B-Instruct SFT smoke test

**Status:** ready-for-agent

- [ ] 四档运行使用相同模型、Prompt、taxonomy、媒体管线、processor、评估配置和可比较的训练策略；任何差异都有显式记录。
- [ ] 每档训练均可恢复、无未处置 OOM/NaN/NCCL 故障，并产出合法 SFT adapter。
- [ ] 每档在固定 dev、能力保持集和历史回归集上使用确定性低温配置评估。
- [ ] TensorBoard 为四档运行使用稳定且互不覆盖的 run 标识，并记录完整 hparams 与版本元数据。
- [ ] TensorBoard 曲线至少可比较 loss、学习率、梯度范数、token/长度分布、吞吐、显存和 dev 四字段合法率。
- [ ] 报告展示 2.5k→5k→7.5k→10k 的边际变化，不把历史指标或观察结果擅自提升为新验收阈值。
- [ ] 选中的 SFT checkpoint 有明确理由、完整版本和恢复验证，可作为 GRPO 唯一起点。

