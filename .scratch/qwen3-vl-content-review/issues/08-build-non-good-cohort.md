# 08 — 完成 3,500 条非 good 数据纵向生产

**What to build:** 使用已验收的数据生产链路，把业务风险媒体完整转化为 3,500 条合格非 good 参考样本，同时满足九个业务类别总量和 44 个非 good `risk_type` 的最低覆盖。

**Blocked by:** 06 — 贯通原始样本到正式参考答案

**Status:** ready-for-agent

- [ ] gamble、porn、finance、politic、other_fraud、login、fake、vpn、game 分别达到 750、700、470、460、360、320、260、90、90 条。
- [ ] 44 个非 good `risk_type` 各不少于 60 条，剩余样本按错误风险与业务优先级分配并留有记录。
- [ ] 每条样本均通过四字段、CoT、枚举和 R5 与 `human_label` 一致性检查。
- [ ] 精确重复 MD5 不会被重复计入 3,500 条有效配额。
- [ ] 多风险画面由 `human_label` 锚定唯一主域，但所有真实可见 evidence 仍被保留。
- [ ] 隔离样本、生成失败、各业务域和各小类缺口单独报告，不通过修改真实证据强行补齐。
- [ ] 产出可供后续冻结步骤读取的非 good cohort 清单和分布报告。

