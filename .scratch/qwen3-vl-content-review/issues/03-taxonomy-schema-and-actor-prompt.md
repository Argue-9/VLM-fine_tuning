# 03 — 锁定 taxonomy、四字段 Schema、canonical actor Prompt 与数据制作 Prompt

**What to build:** 让端到端链路使用锁定版本的 46 类 taxonomy、evidence 枚举、四字段输出契约、唯一 canonical actor Prompt，以及独立版本化的数据制作 Prompt；并对字段结构、空集合规则、短 CoT、标签泄漏与两套 Prompt 的独立性进行确定性校验。数据制作 Prompt 必须在本切片内产出并版本锁定，作为后续试点（05）与数据生产（06）的冻结输入。

**Blocked by:** 01 — 建立可执行的端到端契约骨架

**Status:** ready-for-agent

- [ ] `risk_type` 枚举完整覆盖已确认的 46 个值，并能验证其所属的 10 个业务域。
- [ ] evidence 只能使用锁定枚举；仅四个已确认的 `*_other` 允许空 evidence，其余 42 类必须非空。
- [ ] actor completion 必须是单个严格 JSON object，恰好按 `description`、`evidence`、`risk_type`、`cot` 顺序包含四个字段。
- [ ] `description`、`evidence`、`risk_type`、`cot` 的类型与非空规则被严格验证；JSON 外文本、Markdown 和原生 thinking 被拒绝。
- [ ] `cot` 只能包含 1～3 个短步骤，并可检查 evidence 覆盖、无额外 evidence、risk_type 确认及先 evidence 后 risk_type 的顺序。
- [ ] 每个 `risk_type` 至少有一个合法 fixture，四个空 evidence 例外及典型非法组合均有测试。
- [ ] actor Prompt 不包含 `human_label`、candidate/ref、R5 输出、R5 内部规则、二元标签或 10 类答案。
- [ ] SFT、GRPO、评估和部署能够通过同一个版本标识加载完全相同的 canonical actor Prompt 与 taxonomy。
- [ ] 数据制作 Prompt 与 actor Prompt 是两套独立正文，分别记录 `dataset_prompt_version` 与 `actor_prompt_version`。
- [ ] 数据制作 Prompt 可见媒体、`human_label`、完整 taxonomy、详细边界说明与候选生成规则，用于生成 `candidate_*`；严禁作为 actor 训练或部署输入，且不出现在 SFT/GRPO 训练样本或部署链路中。
- [ ] 数据制作 Prompt 在本切片内产出并版本锁定，其正文、渲染变量与防泄漏检查可追溯，作为 05 试点与 06 数据生产的冻结输入。
