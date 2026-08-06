# SFT 直接监督完整生产 JSON

正式 SFT 对完整四字段生产 JSON 进行 assistant-only token loss 训练，字段顺序固定为 `description → evidence → risk_type → cot`；GRPO 从该 SFT adapter 启动并继续生成相同契约的 JSON。若 SFT 只学习 `description` 或 `evidence+risk_type`，GRPO 就必须同时探索缺失字段、JSON 结构和任务判断；启用 KL reference 时，这种 schema 迁移还会受到来自短输出 SFT reference 的反向约束。完整 JSON 冷启动使初始策略、reference、reward parser 与生产输出保持一致，让 GRPO 主要优化正确性和一致性。仅核心字段或去掉 `cot` 的方案不作为正式 actor。Qwen原生thinking保持关闭，SFT target及后续生成不得在JSON前后包含 `<think>` 或自由推理文本；JSON内的 `cot` 仍是1～3个简短的evidence→risk_type确认步骤。GRPO completion首轮上限为512，正式长度边界根据参考数据P95/P99确定，所有阶段禁止静默截断。调研依据见 `SFT输出方案调研.md`。
