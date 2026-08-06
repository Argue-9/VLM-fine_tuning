# 分离数据集制作 Prompt 与 Actor Prompt

本项目维护两套独立版本的提示词。数据集制作 prompt 可读取媒体、`human_label`、完整 taxonomy、详细边界规则和候选生成要求，仅用于产生待确认的 `candidate_*`；训练/推理 actor prompt 则只包含紧凑 taxonomy 与生产四字段 JSON schema，严禁出现 `human_label`、`candidate_*`、`ref_*`、R5/advice/confidence、二元标签或 R5 内部规则，并由 SFT、GRPO、评估和部署共同使用。两套 prompt 从同一版本化 taxonomy 派生，但分别记录 `dataset_prompt_version`、`actor_prompt_version` 与 `taxonomy_schema_version`。该分离避免把数据制作阶段的标签信息泄漏给 actor，也避免训练与生产输入分布漂移。
