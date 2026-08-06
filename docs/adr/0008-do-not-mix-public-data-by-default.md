# 首版不默认混入公开原生训练数据

本项目首版 SFT 以100%业务四字段数据建立基线，公开原生 VQA、caption、通用聊天或纯文本指令数据不直接进入正式 actor；缺少完整 `risk_type/evidence/human_label/R5` reference 的公开原生样本也不进入 GRPO。项目另建200～500条不参与训练的生产相关能力保持集，先比较 base、SFT 与 GRPO 是否在中文 OCR、UI/文档、细节识别、GIF时序或四字段 JSON 稳定性上发生实质退化。只有确认退化后，才对许可合格的公开素材重新执行项目数据制作流程，并以相同 actor prompt、taxonomy 和四字段 schema 进行5%与10%的定向 replay 消融；混合比例同时按 assistant token 与 visual token 统计。该决策避免未经验证的异构输出协议损害生产 JSON 和业务目标，同时保留在实测退化后恢复必要能力的路径。
