# 17 — 选择 V100 推理后端并贯通生产链路

**What to build:** 使用最终 actor 在相同 V100、媒体、Prompt 和样本条件下比较两卡 Transformers 与 LMDeploy PyTorchEngine TP=2，选择后端并接入生产端到端链路。

**Blocked by:** 02 — 贯通统一静态图片与 GIF 媒体管线；03 — 锁定 taxonomy、四字段 Schema、canonical actor Prompt 与数据制作 Prompt；04 — 接通 R5 决策边界与最终结果；16 — 执行四档 GRPO 学习曲线并选择最终 actor

**Status:** ready-for-agent

- [ ] 两个候选后端都从视觉 batch 1 开始，并使用相同最终 adapter、processor、媒体结果、Prompt、taxonomy 和严格 parser。
- [ ] Transformers 方案测试非对称/手动连续模型放置，避免让 GPU0 独自承担不受控的视觉塔峰值。
- [ ] LMDeploy 方案使用 PyTorchEngine TP=2；当前稳定版 vLLM 不被加入 V100 正式候选。
- [ ] 同集报告比较输出一致性、峰值显存、首 token 延迟、端到端延迟、吞吐、稳定性和运维复杂度。
- [ ] 选定后端在目标 V100 上对静态图片与 GIF 连续回归无 OOM，并生成合法四字段 JSON。
- [ ] 生产包装只把 `risk_type/evidence` 交给 R5，并返回由 R5 决定的二元结果与 10 类 `vlm_advice`。
- [ ] 最终后端选择、失败回退和版本锁定有明确记录，独立推理实现不被误接入 GRPO rollout。

