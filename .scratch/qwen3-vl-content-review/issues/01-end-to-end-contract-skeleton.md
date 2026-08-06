# 01 — 建立可执行的端到端契约骨架

**What to build:** 建立第一条可运行的内容审核纵向链路：接收静态图片，经可替换 actor 得到四字段结果，严格解析后调用可替换 R5，最终返回 R5 所决定的二元结果与 10 类 `vlm_advice`。该切片使用确定性测试替身验证系统边界，不把测试替身误作正式模型或 R5。

**Blocked by:** None — can start immediately.

**Status:** ready-for-agent

- [ ] 一条合法静态图片 fixture 能从统一入口走到最终二元结果与 `vlm_advice`，且中间四字段结果可观察。
- [ ] actor 与 R5 都通过明确接口注入，测试替身可以在后续 ticket 中逐个替换而不改变调用方契约。
- [ ] 严格解析失败、actor 失败和 R5 失败具有不同的可判定错误结果，不被吞掉或混为模型分类。
- [ ] 成功结果只采用 R5 返回的二元结果与 `vlm_advice`，不存在 VLM 侧固定映射。
- [ ] 每次运行记录 actor、R5、taxonomy、Prompt、媒体管线和 processor 的版本占位元数据。
- [ ] 自动化端到端测试覆盖成功链路与至少一种 actor 非法输出。

