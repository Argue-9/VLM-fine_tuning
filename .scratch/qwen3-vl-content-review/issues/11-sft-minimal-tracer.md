# 11 — 完成 SFT 最小可执行 tracer

**What to build:** 使用轻量兼容测试模型和少量合法多模态样本贯通 SFT 数据、processor、assistant-only loss、LoRA、checkpoint 恢复和推理回归，为 8 卡正式 smoke test 提前消除实现错误。轻量模型只用于测试训练链路，不作为正式 actor。

**Blocked by:** 02 — 贯通统一静态图片与 GIF 媒体管线；03 — 锁定 taxonomy、四字段 Schema、canonical actor Prompt 与数据制作 Prompt；06 — 贯通原始样本到正式参考答案

**Status:** ready-for-agent

- [ ] 一次最小训练运行至少完成两个 optimizer update，并能从保存的 adapter 恢复推理。
- [ ] loss mask 只覆盖完整四字段 assistant JSON，不覆盖 system、user 或视觉 token。
- [ ] 可训练参数检查只包含七类目标语言线性层中的 LoRA 参数，视觉塔、merger/projector、embedding、`lm_head` 和底座冻结。
- [ ] LoRA 配置使用 rank 32、alpha 64、bias none、SFT dropout 0.05。
- [ ] SFT 使用包含视觉 token 和目标 token 的真实 target-inclusive 长度；超长样本被删除并计数，不静默截断。
- [ ] TensorBoard 至少记录 train loss、learning rate、gradient norm、optimizer step、有效 token 数、样本长度、吞吐和超长删除计数。
- [ ] TensorBoard event 可被标准读取器加载，标量 step 单调且运行元数据包含模型、数据、Prompt、taxonomy 和随机种子版本。
- [ ] 恢复后的 adapter 能生成可被严格解析并继续进入 R5 的四字段 JSON。

