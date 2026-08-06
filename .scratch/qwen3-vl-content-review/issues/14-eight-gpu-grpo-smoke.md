# 14 — 完成 8 卡 Qwen3-VL-8B GRPO smoke test

**What to build:** 在单机 8×V100 上用全部 ZeRO-3 rank 原地执行 Transformers rollout 和 GRPO 更新，证明正式模型、奖励、reference、generation batch 与 TensorBoard 监控在目标硬件上稳定运行。

**Blocked by:** 12 — 完成 8 卡 Qwen3-VL-8B SFT smoke test；13 — 完成 GRPO 最小可执行 tracer

**Status:** ready-for-agent

- [ ] 8 张 GPU 都作为训练 rank，明确使用 `use_vllm=false` 和 `ds3_gather_for_generation=false`，没有 6+2 卡拆分。
- [ ] 使用每卡 batch 1、梯度累积 1、全机 `generation_batch_size=8`。
- [ ] 默认批次实际形成 2 个 Prompt×每个 4 completions，采样参数与已确认规格一致。
- [ ] 至少完成两个 optimizer update，无 OOM、NaN/Inf、NCCL hang、R5 静默失败或输入/输出静默截断。
- [ ] 如果 G=4 失败，保留峰值和错误证据后改为 4 个 Prompt×每个 2 completions，并重新通过全部 smoke 条件。
- [ ] TensorBoard 由约定 rank 写主事件，包含总奖励及全部分量、KL、语义门、零方差组、completion 长度、吞吐、step 时间和各 rank 峰值显存汇总。
- [ ] TensorBoard 明确区分模型低奖励和 R5 基础设施故障，且 event 文件可正常加载。
- [ ] 保存的 GRPO smoke adapter 能恢复并完成静态图和 GIF 四字段生成及 R5 调用。

