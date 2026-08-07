# 05 — 完成数据标注模型同集试点

**What to build:** 在同一批包含静态图片、GIF、good、风险和边界场景的业务样本上运行 `qwen3.7-plus` 与本地候选方案，经过完全相同的媒体、Prompt、Schema 和 R5 检查后，选择全量 candidate 生成方式。

**Blocked by:** 02 — 贯通统一静态图片与 GIF 媒体管线；03 — 锁定 taxonomy、四字段 Schema、canonical actor Prompt 与数据制作 Prompt；04 — 接通 R5 决策边界与最终结果

**Status:** ready-for-agent

- [ ] 试点样本清单固定，并同时覆盖静态图、GIF、good、主要风险域、弱信号和空 evidence 例外。
- [ ] `qwen3.7-plus` 与至少一个可运行的本地候选使用同一数据制作 Prompt、媒体结果和 taxonomy 版本。
- [ ] 每个候选结果保留原始响应、解析状态、四字段 candidate、失败原因和完整版本元数据。
- [ ] 报告比较严格 JSON 合格率、字段/枚举合格率、R5 一致率、延迟、吞吐、峰值显存和失败恢复能力。
- [ ] 本地候选测试明确记录模型、精度、设备分布和推理后端，不把训练 actor Prompt 当作数据制作 Prompt。
- [ ] 形成有证据的候选生成选型记录；全量生成不得在选型记录完成前启动。
- [ ] 试点的任何候选结果都不会绕过晋升流程直接成为 `ref_*`。

