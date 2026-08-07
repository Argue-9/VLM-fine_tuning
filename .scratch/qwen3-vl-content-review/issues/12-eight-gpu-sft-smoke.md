# 12 — 完成 8 卡 Qwen3-VL-8B-Instruct SFT smoke test

**What to build:** 在单机 8×V100-SXM2-16GB 上运行正式 Qwen3-VL-8B-Instruct FP16 LoRA SFT 最小训练，证明 ZeRO-3、真实视觉 token、checkpoint 和 TensorBoard 监控能够在目标硬件上稳定工作。

**Blocked by:** 11 — 完成 SFT 最小可执行 tracer

**Status:** ready-for-agent

- [ ] 启动 8 个活跃训练 rank，每张 GPU 一个进程，并有证据表明 ZeRO-3 对模型状态进行分片。
- [ ] 使用每卡 batch 1、梯度累积 2、有效 batch 16、真实 target-inclusive `group_by_length`。
- [ ] 使用 FP16、SDPA 或记录后的 eager 回退，不使用 BF16、FlashAttention-2、packing 或 padding-free。
- [ ] 至少完成两个 optimizer update，无 OOM、NaN/Inf、NCCL hang 或静默截断。
- [ ] 记录每张 GPU 的峰值显存、step 时间、吞吐和活跃 rank 状态，默认不启用 CPU/NVMe offload。
- [ ] TensorBoard 主事件只由约定 rank 写入，避免多 rank 重复 step 或事件文件冲突。
- [ ] TensorBoard 包含 loss、learning rate、gradient norm、token/长度统计、吞吐及各 rank 峰值显存汇总。
- [ ] 保存的正式 SFT adapter 可以恢复，并在相同版本媒体、Prompt 和 processor 下生成合法四字段结果。

