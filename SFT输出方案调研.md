# SFT 输出方案调研：Qwen3-VL 风险审核应监督什么

> 调研日期：2026-08-05  
> 状态：主 SFT 输出方案已确认；长度硬上限保留到正式数据统计后确定  
> 范围：只研究 SFT 的监督输出与序列化；不决定 GRPO 奖励权重、训练超参数或最终验收口径。

## 1. 结论摘要

本项目的主 SFT 目标，建议采用**与生产推理完全相同的单个完整 JSON**，监督四个已确认字段：

```text
description -> evidence -> risk_type -> cot
```

也就是：

```json
{"description":"页面展示六合彩号码、生肖图和投注引导文字。","evidence":["online_lottery_activity"],"risk_type":"gamble_lottery","cot":["画面中的报码、生肖和投注引导支持 evidence=online_lottery_activity。","该 evidence 对应网络彩票投注场景，因此 risk_type=gamble_lottery。"]}
```

推荐同时满足以下条件：

- 只对 assistant 的完整 JSON（包括括号、键名、标点、字段值与结束标记）计算 token-level SFT loss；system、user、视觉 token 不计算 loss。
- 训练 target 使用通过既有准入流程的 `ref_*` 值，但输出键名使用生产字段名，不输出 `ref_` 前缀。
- `human_label`、`candidate_*`、R5 输出、二元结果和样本元数据均不进入 assistant target，也不作为生产式模型输入。
- 数据集制作 prompt 与 actor prompt 分离：前者可读取 `human_label`、完整 taxonomy 和详细边界以生成候选；后者只含紧凑 taxonomy 与生产 schema，并在 SFT、GRPO、评估、部署中保持一致。
- 第一版不增加辅助分类头、不混入同一 prompt 的多种 target、不做字段 loss 加权。
- 保留 `cot`，因为它已属于生产契约；但把它定义为**可核验的短决策记录**，而不是“模型真实内部思维”的保证。
- 使用 Qwen3-VL-8B-Instruct 的 non-thinking 路径；SFT target 不含 `<think>` 或 JSON 外推理文本。
- `cot` 放在最后：本项目已经把它定义为对 `evidence + risk_type` 的确认，而非分类前的自由推理。这样可避免噪声 rationale 成为 `risk_type` 的自回归前置条件。
- evidence 数组内部顺序不承载业务语义；最终需求选择首版不强制排序或去重，也不为重复设置单独奖励、预处理或专项监控。

这不是声称“完整 JSON 在所有任务上绝对最优”。它是在本项目的生产契约、R5 接口和 SFT→GRPO 链路下风险最低的正式 actor；本文第 12 节的短输出组只作诊断，不再承担主方案选型职责。

## 2. 已知项目约束

本调研以下列已确认前提为边界：

- 基座：`Qwen3-VL-8B-Instruct`，输入为单个静态 image 或单个 GIF video 帧列表；静态图/GIF 每帧视觉预算分别为1024/256 token。
- 数据约 1 万条，原始标签只有 10 类 `human_label`；数据制作后才产生经确认的 `ref_description/ref_evidence/ref_cot/ref_risk_type`。
- VLM 生产输出是严格 JSON：`description/evidence/risk_type/cot`。
- `risk_type` 是 46 类封闭单值枚举；`evidence` 是封闭多值数组，内部顺序不承载业务语义，首版不专门处理重复项。
- R5 只消费 `risk_type + evidence`，输出 10 类 `vlm_advice` 与置信度；VLM 不负责二元映射。
- `cot` 为 1～3 个短步骤，用于确认全部 evidence，并由 evidence 确认最终 risk_type。
- 只有参考 `risk_type/evidence` 经 R5 得到的 advice 与 `human_label` 一致，样本才进入主训练集。
- 训练链必须先 SFT 冷启动，再从 SFT adapter 继续 GRPO。
- 主框架是 MS-SWIFT 标准 Transformers 后端。

因此，本问题不是一般的“分类任务是否需要生成解释”，而是：在生产端已经要求生成四字段 JSON、GRPO 又需要从可解析策略开始的条件下，SFT 是否应该提前学习完整契约。

## 3. 资料能直接证明什么

为避免把通用论文结论误当作本项目实验证据，本文区分三类陈述：

- **官方事实**：框架、模型或标准的明确行为。
- **论文证据**：原始论文在其任务和实验设置中的观察。
- **项目推断**：依据本项目契约和上述事实作出的工程判断，仍需本项目实验验证。

### 3.1 官方事实：SFT 学的是有顺序的 assistant token 序列

TRL 的 SFT 文档给出的目标是逐 token 负对数似然：每个目标 token 都根据之前的 token 预测；日志中的 loss 是非掩码 token 上的平均交叉熵。也就是说，JSON 虽然在业务上是一个对象，模型实际学习的是一条有顺序的 token 序列。[TRL SFT：loss 与 label shifting](https://huggingface.co/docs/trl/en/sft_trainer#looking-deeper-into-the-sft-method)

Transformers 也明确说明，causal LM 只观察左侧 token 并预测下一个 token。[Transformers：Causal language modeling](https://huggingface.co/docs/transformers/main/tasks/language_modeling)

JSON 标准本身则把 object 定义为无序的 name/value 集合，array 是有序序列。因此，“字段顺序影响模型生成”来自自回归模型，而不是 JSON 语义要求；不存在 RFC 或 Qwen 官方给出的最优字段顺序。[RFC 8259，第 4、5 节](https://www.rfc-editor.org/rfc/rfc8259)

### 3.2 官方事实：训练和推理必须使用同一聊天模板语义

Transformers 文档指出，chat messages 最终会被转换成包含角色控制 token 的单一 token 序列，错误控制 token 会显著损害表现；Qwen3-VL 官方模型卡也通过 `AutoProcessor.apply_chat_template` 处理图像/文本消息。[Transformers：Chat templates](https://huggingface.co/docs/transformers/chat_templating)、[Qwen3-VL-8B-Instruct 模型卡](https://huggingface.co/Qwen/Qwen3-VL-8B-Instruct)

MS-SWIFT 的标准多模态 SFT 格式同样把媒体放在 `images/videos`，把监督响应放在 assistant message；其文档明确支持 Qwen2/2.5/3-VL 的视频帧列表。[MS-SWIFT：Custom dataset / Multimodal](https://github.com/modelscope/ms-swift/blob/main/docs/source_en/Customization/Custom-dataset.md#multimodal)

### 3.3 官方事实：可以且应该只训练 assistant 响应

MS-SWIFT 的 SFT 默认 `loss_scale=default`：assistant response 权重为 1，system、user 和 multimodal token 不参与 loss。[MS-SWIFT：loss_scale](https://github.com/modelscope/ms-swift/blob/main/docs/source_en/Instruction/Command-line-parameters.md#template-arguments)

原生 TRL 备用实现也提供 `assistant_only_loss=True`，但它依赖 chat template 正确返回 assistant mask；当前 TRL 对已知 Qwen3 模板会自动补丁。不能只看配置名，仍应解码 `labels` 验证掩码。[TRL SFT：Train on assistant messages only](https://huggingface.co/docs/trl/en/sft_trainer#train-on-assistant-messages-only)

### 3.4 论文证据：高质量冷启动可改善 RL 起点，但不证明某个 JSON 字段组合最优

DeepSeek-R1 对比了直接 RL 与先用少量冷启动数据再 RL 的路线。论文报告，直接 RL 的 R1-Zero 存在可读性差和语言混杂，而 R1 使用数千条冷启动数据建立可读输出模式后再做 RL。[DeepSeek-R1，第 2.3.1 节](https://arxiv.org/abs/2501.12948)

这是“先让策略学会目标响应形态，再做 RL”的正面证据，但任务是数学/推理，并不能直接证明本项目必须选择四字段 JSON。

### 3.5 官方事实 + 项目推断：无差异奖励会让 GRPO 缺少有效相对信号

TRL 的 GRPO 文档给出组相对 advantage：同一 prompt 的多条 completion 以组内 reward 均值和标准差归一化；其 `frac_reward_zero_std` 专门记录同组 reward 无差异的比例。[TRL GRPO：Computing the advantage](https://huggingface.co/docs/trl/en/grpo_trainer#computing-the-advantage)、[DeepSeekMath 原始论文](https://arxiv.org/abs/2402.03300)

据此可作项目推断：如果 SFT 没学过生产 JSON，GRPO 初始 completion 大量缺字段或不可解析，格式/R5 reward 可能在组内一起失败，产生零方差组。GRPO 仍可能通过探索学会格式，但在 8×V100、约 1 万样本的预算下，把确定性的语法和字段契约留给 RL，性价比很差。

### 3.6 论文证据：rationale 可能有帮助，但不等于忠实解释

CoT 和 STaR 在数学、常识推理等任务中显示了“监督或生成中间 rationale”可能优于只预测最终答案；Multimodal-CoT 也在 ScienceQA/A-OKVQA 上采用“先 rationale、后 answer”的两阶段设计。[Chain-of-Thought Prompting](https://arxiv.org/abs/2201.11903)、[STaR](https://research.google/pubs/star-self-taught-reasoner-bootstrapping-reasoning-with-reasoning/)、[Multimodal-CoT](https://arxiv.org/abs/2302.00923)

但 Turpin 等人的 NeurIPS 2023 论文表明，模型生成的 CoT 可能是对既有答案的合理化，不能自动视为真实决策原因。[Language Models Don't Always Say What They Think](https://proceedings.neurips.cc/paper_files/paper/2023/hash/ed3fea9033a80fea1376299fa7863f4a-Abstract.html)

对本项目的含义是：`cot` 可以作为**受约束、可机械检查的证据—类别说明**，但不应作为“模型内部推理绝对忠实”的审计证明。尤其候选 CoT 由教师模型参考 `human_label` 生成时，存在事后合理化风险，必须依赖已有字段一致性与样本准入检查。

## 4. 候选 SFT 输出比较

下表的等级是项目推断，不是官方基准结果。

| 方案 | 生产契约一致性 | R5 核心字段冷启动 | GRPO 格式冷启动 | 标签噪声暴露 | 主要问题 | 结论 |
|---|---|---|---|---|---|---|
| A. 完整 JSON：四字段 | 高 | 高 | 高 | 四字段都会进入 loss | 长文本可能支配 token loss；CoT 质量必须受控 | **主选** |
| B. 仅 `description` | 低 | 无 | 低 | 只暴露描述噪声 | 没教 evidence、risk_type、JSON 契约；R5 无法消费 | 淘汰 |
| C. 仅 `evidence+risk_type` | 中低 | 高 | 中低 | 较低 | description/cot 只能留给 GRPO 或后处理；训练/推理不一致 | 仅作诊断基线 |
| D. 完整 JSON，但不含 `cot` | 中 | 高 | 中 | 较低 | 与已确认生产契约不一致；GRPO 需从零补 cot | 仅作 CoT 消融 |
| E. 分阶段：先核心字段、再完整 JSON | 最终可高 | 高 | 高 | 后阶段仍暴露 | 增加阶段和超参；第一阶段 target 与生产不同，收益未知 | 非首轮默认 |
| F. 多任务：同图同时训练核心与完整输出 | 可高 | 高 | 高 | 重复样本放大噪声 | 必须用不同 prompt 区分任务，否则同 prompt 多 target；容易引入模式混淆 | 条件性备用 |
| G. 生成 JSON + 辅助分类头 | 需额外部署契约 | 高 | 高 | 两套标签路径 | 改模型结构、checkpoint 和推理；与“单 JSON 输出”目标不一致 | 不采用 |

### 4.1 为什么不选“只教 description”

`description` 是自由文本且不是 R5 输入。只监督它，相当于把以下内容全部留给 GRPO 探索：JSON 语法、四个键、46 类枚举、evidence 数组、空 evidence 特例、CoT 结构和 risk_type/evidence 的一致性。这样做没有利用现有 `ref_evidence/ref_risk_type/ref_cot` 标签，也无法建立 R5 可消费的策略起点。

### 4.2 为什么“只教 risk_type+evidence”不作为主方案

这是很有价值的**诊断基线**：它能测出在较短 target 下，模型对 R5 核心字段能达到的上限，并帮助识别 description/cot 是否稀释核心任务。

但如果生产仍要求四字段，它不适合作为最终 SFT actor：GRPO 的初始策略不满足生产 prompt 的响应契约；description 和 cot 会变成没有监督冷启动的生成任务。除非需求将生产输出缩减为两字段，否则训练/推理契约不应人为分裂。

### 4.3 为什么首轮不做分阶段或多任务

两者在理论上都可行，但当前没有项目数据证明其收益：

- “核心字段 SFT → 完整 JSON SFT”最终仍需完整 target，增加了阶段、学习率/步数和遗忘风险。
- 把同一图像复制成核心输出和完整输出会改变样本权重。如果 prompt 相同而 target 不同，会直接制造多目标歧义；如果 prompt 不同，模型还需学习一个生产中不用的任务模式。
- 约 1 万样本、LoRA、冻结视觉侧的设置下，先建立简单清晰的单任务基线，更容易解释 SFT 与后续 GRPO 的增益。

只有当完整 JSON 的 R5 核心指标显著低于核心两字段基线，且差异可复现时，才值得引入多任务/课程学习。

## 5. SFT 输出如何影响后续 GRPO

本项目不能只按“SFT validation loss 最低”选择 target。SFT checkpoint 同时决定 GRPO 的初始采样分布、可获得的 reward 密度、completion 长度、多样性，以及启用 KL 时的 reference 锚点。

### 5.1 初始 policy、reference 与目标契约必须一致

MS-SWIFT 允许 GRPO 同时用 SFT adapter 初始化可训练 policy，并用 `ref_adapters` 指定 reference adapter；其文档给出的 LoRA SFT→GRPO 接续方式是同时设置 `--adapters sft_ckpt --ref_adapters sft_ckpt`。`beta>0` 时，KL 项限制 policy 偏离 reference；`beta=0` 则不加载 reference。[MS-SWIFT：RLHF/GRPO 参数](https://github.com/modelscope/ms-swift/blob/main/docs/source_en/Instruction/Command-line-parameters.md#rlhf-arguments)

因此，如果 SFT 只学习 description，而 GRPO prompt 要求完整 JSON，会出现两个方向同时冲突：

- online reward 要求 policy 从 description-only 跳到四字段 JSON；
- 非零 KL 又要求它靠近 description-only 的 SFT reference。

KL 并非数学上禁止策略改变，但会让“补齐 schema”与“保持 reference”相互拉扯。相反，完整 JSON SFT 使 policy、reference、GRPO rollout 和生产 parser 从一开始共享同一契约，KL 主要约束质量退化，而不是阻止 schema 迁移。

这是项目推断。准确 KL 系数仍属于后续 GRPO 参数讨论；本文只建议初始阶段保留 `beta>0`，不在此确定具体数值。MS-SWIFT 当前文档的 GRPO 默认值是 0.04，而原生 TRL 当前默认 0.0，说明不存在跨框架统一默认最优值。[MS-SWIFT 参数文档](https://github.com/modelscope/ms-swift/blob/main/docs/source_en/Instruction/Command-line-parameters.md#grpo-arguments)、[TRL GRPOConfig](https://huggingface.co/docs/trl/en/grpo_trainer#trl.GRPOConfig)

### 5.2 完整合法 JSON 提高首轮可解析率与 reward 密度

GRPO 对每个 prompt 生成多条 completion，再调用 reward function。若 completion 缺少 `risk_type/evidence` 或 JSON 无法解析，R5 reward 无法形成任务区分，多个 completion 容易一起只得到格式失败分。[MS-SWIFT：GRPO 算法与自定义 reward](https://github.com/modelscope/ms-swift/blob/main/docs/source_en/Instruction/GRPO/GetStarted/GRPO.md)

完整 JSON SFT 的价值不仅是“格式更好看”，而是让首轮 rollout 更可能同时获得：

- JSON parse/字段完整 reward；
- 封闭枚举合法 reward；
- evidence-risk_type 一致性 reward；
- R5 advice 与 `human_label` 的任务 reward。

这样 reward 能从“全部不可用”变成多个可分解信号，组内更容易出现可比较的样本。实际提升幅度必须用第 12.5 节的 SFT actor 多采样检查测量，不能由文献直接给出。

### 5.3 completion 越长，V100 上的 rollout 与训练成本越高

MS-SWIFT 的 `num_generations` 是每个 prompt 生成的样本数，GRPO batch 以 completion 为单位；每条 completion 都要生成、计算 reward，并为 policy/old/reference 计算 token log-prob。框架也单独记录 completion 平均/最大长度和截断比例。[MS-SWIFT：GRPO 参数与 logged metrics](https://github.com/modelscope/ms-swift/blob/main/docs/source_en/Instruction/GRPO/GetStarted/GRPO.md)

据此可作直接工程推断：当 `description+cot` 每条多出 ΔT 个 token、每个 prompt 采样 G 条时，每轮至少新增约 `G×ΔT` 个生成和打分 token；policy 更新和启用 KL 时的 reference log-prob 计算也随 completion token 增加。实际显存/时间不是严格线性，因为还有视觉编码、padding、批处理和 kernel 效率，但在 8×16GB V100 上，长自由文本会被 `num_generations` 放大，而不是只在 SFT 中多付一次成本。

因此推荐完整 JSON **不等于推荐长 JSON**：

- description 只保留完成视觉事实核验所需内容；
- cot 只保留 1～3 个短确认步骤；
- `max_completion_length` 要根据真实 token 分布设定，并监控 `completions/clipped_ratio`；
- 先压缩套话，再考虑减少核心字段或降低 `num_generations`。

### 5.4 SFT 过度模板化会降低 GRPO 组内多样性

GRPO 的相对 advantage 依赖同 prompt 多条 completion 的 reward 差异。MS-SWIFT/TRL 都记录 `frac_reward_zero_std`；它表示组内 completion reward 相同、几乎没有有效相对信号。[MS-SWIFT：GRPO logged metrics](https://github.com/modelscope/ms-swift/blob/main/docs/source_en/Instruction/GRPO/GetStarted/GRPO.md#logged-metrics)、[TRL：`frac_reward_zero_std`](https://huggingface.co/docs/trl/en/grpo_trainer#logged-metrics)

若 SFT 过拟合到完全固定的长 description/cot 套话，或者 GRPO 采样温度过低，同组输出可能高度相同：全部正确时没有可比较增益，全部错误时也没有改进方向。完整 JSON SFT 的目标应是“稳定 schema + 基本正确”，不是把每种类别背成唯一文案。

控制方式：

- schema 和枚举必须 canonical，但自然语言只要求短、事实一致，不强迫唯一措辞；
- 避免重复数据和类别模板大量复制；
- 进入 GRPO 前测同 prompt 多采样的 unique completion、unique `(evidence,risk_type)`、reward std；
- SFT 训练轮数按开发集核心指标和生成多样性早停，不以训练 loss 持续下降为唯一目标。

### 5.5 R5 只看核心字段时，description/cot 可能在 GRPO 中退化

标准 GRPO 用 sequence reward 得到组相对 advantage，并把同一 completion 的 advantage 用到其 token 级 policy objective。若 reward 只由 R5 的 `risk_type/evidence` 计算，description/cot token 没有独立正确性信号；它们可能随能提高 R5 reward 的更新一起漂移、缩短、套话化或产生与核心字段不一致的内容。[DeepSeekMath 原始 GRPO 目标](https://arxiv.org/abs/2402.03300)、[MS-SWIFT：GRPO objective](https://github.com/modelscope/ms-swift/blob/main/docs/source_en/Instruction/GRPO/GetStarted/GRPO.md#algorithm-overview)

为让最终 actor 保住四字段契约，后续 reward 设计建议至少分解为：

- JSON/四键/类型/枚举的 format reward；
- evidence 与 description 的可观察一致性 reward；
- cot 对全部 evidence 和 risk_type 的覆盖/一致性 reward；
- R5 advice 任务 reward；
- 必要的长度或重复惩罚，但不能鼓励删掉必需字段。

同时初始阶段建议使用非零 KL 锚定完整 JSON SFT reference。KL 只能限制整体漂移，不能替代字段级 reward；分解 reward 也不能证明自由文本视觉事实正确，二者需要配合。MS-SWIFT 支持多个 `reward_funcs`、`reward_weights` 和自定义插件，但具体函数与权重留到奖励设计阶段确认。[MS-SWIFT：GRPO arguments](https://github.com/modelscope/ms-swift/blob/main/docs/source_en/Instruction/Command-line-parameters.md#grpo-arguments)

### 5.6 字段顺序同时影响因果依赖、截断和 credit assignment

字段顺序没有官方最优解，但对 causal LM 有三个项目影响：

1. **因果依赖**：后字段可条件于前字段，前字段不能使用后字段。把 noisy cot 放在 risk_type 前，会让 risk_type 依赖 cot；放后则 cot 只能确认结果。
2. **截断**：越靠后的字段越可能在 `max_completion_length` 下被截断。把 R5 核心字段放在长 cot 前，可避免 `risk_type` 本身晚于 cot；但严格 JSON 一旦尾部被截断仍不可解析，所以根本控制仍是限长和截断惩罚。
3. **sequence-level credit**：R5 reward 是 completion 级信号，无法仅把优劣归因给 risk_type/evidence token。把核心字段相邻、自由文本受限，并提供字段一致性 reward，可减少无关 token 对更新的干扰，但不能彻底解决 credit assignment。

这三点共同支持 `description,evidence,risk_type,cot` 作为首选，而不是只按“人类阅读时先推理后答案”的习惯排序。

### 5.7 SFT 标签噪声会成为 GRPO 的起点，也可能被 KL 锚定

若 `ref_description/ref_cot` 含虚构事实，或 `ref_evidence/ref_risk_type` 虽通过 R5 映射却视觉上错误，完整 JSON SFT 会直接模仿这些错误。GRPO 又从该 policy 开始；启用非零 KL 后，reference 还会持续偏好这些模式。

因此“完整 JSON + KL”不会自动修复标签噪声，反而提高了 SFT 标签质量的重要性。应在进入 SFT 前完成字段一致性、视觉事实和 R5 准入；在 GRPO reward 中避免把 `human_label` 单一 outcome 当成 description/cot 正确性的证明。

### 5.8 evidence 的集合语义与首版边界

同一 evidence 集合的不同排列在自回归 token 序列上不同，但本项目已明确其业务语义等价。因此 SFT 数据保留已确认参考数组的顺序，reward 将预测与参考分别转成集合后计算 F1，format 和语义门均不检查 evidence 排列。重复项预计概率较低，首版不增加去重规则、预处理或专项奖励；如实际 rollout 暴露问题，再单独修订。

该结论是结合 JSON array 有序语义、causal LM 和项目 R5 集合语义作出的工程推断。

## 6. 推荐的 canonical 生产 JSON

### 6.1 结构与类型

```json
{
  "description": "string",
  "evidence": ["closed_enum", "..."],
  "risk_type": "closed_enum",
  "cot": ["step 1", "step 2"]
}
```

训练时应序列化为单行、无 Markdown 代码围栏、无前后解释：

```json
{"description":"页面展示六合彩号码、生肖图和投注引导文字。","evidence":["online_lottery_activity"],"risk_type":"gamble_lottery","cot":["画面中的报码、生肖和投注引导支持 evidence=online_lottery_activity。","该 evidence 对应网络彩票投注场景，因此 risk_type=gamble_lottery。"]}
```

### 6.2 推荐字段顺序及其性质

推荐固定为：

1. `description`
2. `evidence`
3. `risk_type`
4. `cot`

必须明确：这是**项目工程推断，不是官方最优顺序**。

推断理由：

- `description` 先写客观可见事实，生成时还未看到类别 token，减少描述直接围绕既定类别反向编造的条件。
- `evidence` 紧接可见事实，将自由文本观察收敛到封闭枚举。
- `risk_type` 紧接 evidence，使 R5 的两个核心输入相邻；risk_type 不依赖可能有噪声的 cot 文本。
- `cot` 已被定义为“确认 evidence+risk_type”，放最后与该语义一致。它是对前面结果的受约束核验记录，不是假定会改善前面已经生成的 risk_type。

当前常见的 `description,evidence,cot,risk_type` 顺序也应保留为消融项：它允许 risk_type 条件于 cot，但会让教师生成的 rationale 成为分类的前置条件，并可能把最终 risk_type 退化成复制 cot 中已出现的类别。究竟哪种更好不能靠文献代替本项目实验。

### 6.3 canonical serialization 规则

- JSON 顶层必须且只能有四个键；键名唯一。
- 固定键顺序，不使用缩进，不附加 Markdown 或自然语言前后缀。
- UTF-8 直接保留中文；使用标准 JSON 转义处理引号、反斜杠和控制字符。
- evidence 数组内部顺序不作为 canonical 条件；首版不强制排序或去重。
- `risk_type` 使用一个 46 类封闭枚举值。
- `cot` 为 1～3 个非空字符串；每个 evidence 必须被明确支持，最终 risk_type 必须被明确确认。
- 只有既定四个 `*_other` 类允许 `evidence=[]`；其 cot 必须解释风险域依据和回退原因。
- 不用 `null`、缺键或空字符串表达“标签未知”。四字段任一参考值未完成的样本不进入完整 JSON 主 SFT。

canonical serialization 只固定顶层 key 顺序、JSON 紧凑格式和字段类型；不把 evidence 数组排列纳入生产契约。

## 7. `cot` 是否进入 SFT

### 7.1 建议：进入，但按“短决策记录”治理

如果生产必须返回 `cot`，主 SFT 就应监督它。否则存在两个不理想选择：

- 让 GRPO 从零学会 cot 的格式和内容；
- 推理阶段由另一个组件补 cot，改变已确认的 VLM 单 JSON 契约。

但它不应是开放式长推理。推荐继续坚持已确认边界：

- 1～3 个短步骤；
- 先逐项说明观察如何支持 `evidence`；
- 再说明这些 evidence 如何支持 `risk_type`；
- 不引入 evidence 数组外的新证据；
- 可说明排除相邻类别，但只能保留一个最终 risk_type；
- 不把 cot 当作 R5 输入，也不把它当成模型内部机制的忠实证明。

### 7.2 为什么推荐放在 risk_type 后

这里的 cot 是“确认”而非“先思考再分类”。放在 `risk_type` 后有两个好处：

- noisy cot 不能成为 risk_type 的左侧条件；
- cot 可以直接核对已经生成的 evidence 与 risk_type。

代价是 cot 不可能通过同一条自回归序列反过来改善已经生成的 risk_type。如果后续目标改变为“让 rationale 参与决策”，应把 `description,evidence,cot,risk_type` 作为另一方案测试，但那是语义变更，不只是 JSON 排版变化。

### 7.3 标签噪声控制

本项目的 cot 候选由教师模型参考 `human_label` 生成，天然可能出现“先知道答案、再合理化”的现象。R5 一致只说明 `risk_type/evidence` 能映射到 10 类标签，不能证明 cot 中每个视觉事实真实存在。

因此主 SFT 只能使用已经转为 `ref_cot` 的样本，并在静态校验中至少保证：

- cot 明确覆盖 evidence 数组中的每项；
- cot 提到的 evidence 集合与数组完全一致；
- cot 的最终类别与 `risk_type` 完全一致；
- cot 不添加 description 中没有依据的具体视觉事实；
- cot 长度受限，避免套话支配训练 token。

人工检查的具体方式和覆盖率仍按项目既有决定，不在本文新增要求。

## 8. assistant-only loss 与训练样本组织

### 8.1 推荐样本语义

概念上的 MS-SWIFT 单条数据为：

```json
{
  "messages": [
    {
      "role": "system",
      "content": "<固定的风险审核系统指令>"
    },
    {
      "role": "user",
      "content": "<image><固定的生产审核指令>"
    },
    {
      "role": "assistant",
      "content": "{\"description\":\"页面展示六合彩号码、生肖图和投注引导文字。\",\"evidence\":[\"online_lottery_activity\"],\"risk_type\":\"gamble_lottery\",\"cot\":[\"画面中的报码、生肖和投注引导支持 evidence=online_lottery_activity。\",\"该 evidence 对应网络彩票投注场景，因此 risk_type=gamble_lottery。\"]}"
    }
  ],
  "images": ["/absolute/path/example.jpg"]
}
```

GIF 按已经确认的抽样策略作为一个 `videos`/video帧列表进入，不能表示成四张独立 image；训练 target 结构保持不变。MS-SWIFT 官方格式支持 Qwen3-VL 的视频或帧列表。

### 8.2 loss mask

主方案：

- system：`labels=-100`
- user：`labels=-100`
- 图像/GIF 视觉占位与视觉 token：`labels=-100`
- assistant JSON 的每个 token：参与 loss
- assistant 回合结束 token：参与 loss，帮助学习正确终止
- padding：`labels=-100`

MS-SWIFT 单轮 SFT 可使用默认 `loss_scale=default`。不要仅凭日志假定掩码正确，应抽样调用 template encoder 并解码 `labels`；MS-SWIFT 官方自定义数据文档也给出了 `template.encode` 后检查 `input_ids/labels` 的方式。[MS-SWIFT：测试最终数据格式](https://github.com/modelscope/ms-swift/blob/main/docs/source_en/Customization/Custom-dataset.md#multimodal)

### 8.3 VLM 截断风险

TRL 官方提醒，VLM 的普通文本截断可能删掉 image tokens；在原生 TRL 备用实现中建议 `max_length=None`，除非已证明整个数据集不会截断视觉 token。[TRL：Training Vision Language Models](https://huggingface.co/docs/trl/en/sft_trainer#training-vision-language-models)

MS-SWIFT 主实现也必须检查编码后视觉 token 和 assistant 尾部 JSON 是否同时完整；不能用“保留开头”或“保留结尾”的粗暴截断解决。SFT 使用 `truncation_strategy=delete`，按“完整actor prompt + 视觉token + 四字段target”的真实编码长度生成独立length cache，并在正式taxonomy/参考数据完成后确定安全 `max_length`。

## 9. token 长度支配与字段加权

### 9.1 风险

标准 SFT 对每个非掩码 token 计算交叉熵并取平均。`risk_type` 通常只有少量 token，`evidence` 较短，而 `description+cot` 可能占 assistant target 的大部分 token。因此它们会贡献更多监督位置，整体 loss 下降并不等价于 46 类分类或 evidence 指标变好。[TRL SFT loss 定义](https://huggingface.co/docs/trl/en/sft_trainer#computing-the-loss)

这不是“长字段一定损害分类”的证明，而是必须单独监控字段级指标的原因。

### 9.2 第一版建议：不做字段权重，先控长度

第一版不建议自定义字段 loss 权重，理由是：

- 单个 JSON response 内做字段级加权需要可靠地把字符字段映射回 tokenizer spans，增加转义、中文 tokenization 和模板边界错误风险。
- MS-SWIFT 虽支持 response 级 `loss_scale`，但本项目四字段在同一个生产 response 中；把字段拆成多个 assistant message 会破坏生产序列。[MS-SWIFT：自定义 response loss_scale](https://github.com/modelscope/ms-swift/blob/main/docs/source_en/Customization/Custom-dataset.md#standard-dataset-format)
- 没有本项目基线时直接设权重，会同时改变优化目标和可解释性。

先采用：

- description 限制为客观、短句，不写分类判断和规则解释；
- cot 保持 1～3 个短步骤，不复述整段 description；
- 数据准备阶段统计每个字段的 tokenizer token 数 `P50/P90/P95/max`，而不是用字符数猜 token 长度；
- 训练/验证分别报告 JSON、description、evidence、risk_type、cot 指标，不用总 loss 替代核心任务指标。

如果完整 JSON 的核心指标稳定落后于核心两字段基线，再测试字段加权或多任务；不要在第一版同时引入。后续 GRPO 的字段重要性优先通过分解的 format/consistency/R5 reward 表达，而不是先把 SFT 改成难以验证的自定义 token 权重。

## 10. 哪些字段不进入 SFT prompt 或 target

下列字段即使存在于离线数据表，也不得出现在生产式 SFT 的 system/user prompt 或 assistant 文本中：

- `human_label`：10 类准入真值，不是 VLM 输出。
- `candidate_description/candidate_evidence/candidate_cot/candidate_risk_type`：未经最终确认的中间产物。
- `ref_description/ref_evidence/ref_cot/ref_risk_type` 这些**字段名**：它们是离线存储名；target 只取其值并映射为生产键名。
- `vlm_advice`、`vlm_confidence`：R5 输出。
- 二元 good/risk 结果：由 R5 规则决定。
- R5 内部规则路径、映射表或调试 trace。
- 样本 ID、媒体路径、来源、审核人、候选模型信息等元数据。

四个 `ref_*` 只允许在离线数据管道中读取，用于构造 assistant 的四键 canonical JSON；构造完成后，actor 可见序列中既没有 `ref_*` 字段名，也没有 `candidate_*`、`human_label` 或 R5 输出。数据集制作 prompt 虽可读取 `human_label` 和候选中间结果，但不得被复用为 actor prompt；否则训练时模型能看到而生产时看不到，形成标签泄漏和输入分布偏移。

## 11. 是否需要辅助头、分阶段或额外多任务

### 11.1 辅助分类头：不推荐

项目最终只部署生成式 JSON，并已冻结 `lm_head` 之外的既定模块策略。新增 46 类或 evidence 多标签分类头会：

- 改变模型结构与 checkpoint；
- 产生“生成结果”和“分类头结果”冲突时的新仲裁问题；
- 需要额外训练、保存、部署和评测路径；
- 不能直接解决 JSON 生成契约。

除非将来明确把 R5 输入改为隐藏向量分类结果，否则不应为首轮增加辅助头。

### 11.2 分阶段/多任务：保留为触发式备用

建议只在以下证据同时出现时启用：

1. 核心两字段 SFT 在同一开发集上的 `risk_type/evidence/R5` 指标显著高于完整 JSON；
2. 差异在至少两个随机种子或重复短跑中稳定；
3. 错误分析显示主要原因确实是长字段/CoT 干扰，而不是标签质量、类别不平衡或视觉信息不足。

可测试的最低复杂度备用是：先用完整 JSON 建立主 actor，再在同一 checkpoint 上对少量、带独立 task prompt 的核心字段样本做混合训练。不要让同一个 prompt 同时对应两种 schema。

## 12. 最小验证与消融清单

以下是决定“是否正式确认完整 JSON SFT”的最小实验，不是最终验收。

### 12.1 数据静态检查

- 100% target 可被标准 JSON parser 解析。
- 顶层键集合精确等于四字段，且序列化顺序一致。
- `risk_type/evidence` 全部属于版本锁定的封闭枚举。
- evidence 全部属于封闭枚举；空数组只出现在允许的四个 `*_other`，数组顺序和重复不作为首版静态门槛。
- cot 数组长度为 1～3；逐项覆盖 evidence，并确认完全相同的 risk_type。
- description 不出现类别判定或 R5 规则结论。
- 所有样本的参考 `risk_type/evidence` 均通过既有 R5→`human_label` 准入。
- 统计四字段各自 token 长度及整条 completion 长度的 `P50/P90/P95/P99/max`，按静态图/GIF、10 类 human_label、46 类 risk_type 分组检查长尾。

### 12.2 编码与 loss mask 检查

- 随机抽样静态图、长文本图、GIF 各若干条，解码 `input_ids` 和非 `-100` labels。
- 非 `-100` labels 应精确等于完整 assistant JSON 加回合结束 token。
- system/user/视觉/padding token 必须全部被 mask。
- 编码后 image/video token 未被截断，JSON 末尾、右花括号和结束 token 未被截断。
- 训练序列化、离线评测 parser、生产 parser 使用同一 schema 版本和同一 canonical serializer。

### 12.3 小样本过拟合检查

先选一个覆盖静态图/GIF、空 evidence 特例和多 evidence 的小集合，验证 LoRA 能过拟合到：

- JSON parse 率接近 100%；
- 四键完整且顺序稳定；
- risk_type/evidence exact match 明显收敛；
- cot 不引用数组外 evidence。

若小集合都学不会，优先排查模板、labels mask、转义、视觉 token 和 max length，而不是直接扩大训练。

### 12.4 最小三组 SFT 消融

在同一数据划分、相同步数/有效 batch、相同 LoRA 配置下短跑：

| 组 | target | 用途 |
|---|---|---|
| A（推荐主线） | `description,evidence,risk_type,cot` | 完整生产契约基线 |
| B（核心上限诊断） | `evidence,risk_type` | 测量长字段对 R5 核心任务的影响 |
| C（CoT 消融） | `description,evidence,risk_type` | 测量 cot 监督的收益/噪声与 token 成本 |

三组都必须单独使用明确 schema 的 prompt，不能让同一 prompt 对应不同 target。B/C 仅用于实验比较，不是生产候选 actor。

至少报告：

- JSON 可解析率、四键完整率、封闭枚举合法率；
- `risk_type` accuracy、macro-F1、按类召回；
- evidence exact-set accuracy、micro/macro precision/recall/F1；
- 参考/预测字段经 R5 后的 10 类指标与二元指标；
- description 的客观事实错误抽检结果；
- cot-evidence 覆盖率、cot-risk_type 一致率、数组外 evidence 幻觉率；
- completion token 长度、截断率和生成延迟。

### 12.5 GRPO 就绪检查

用 SFT actor 对一组固定开发样本做多次采样，观察：

- 各 completion 是否能被 parser 和 R5 消费；
- 格式 reward、枚举 reward、R5 reward 是否存在足够组内差异；
- `frac_reward_zero_std` 是否因“全部格式失败”而偏高；
- 失败主要来自分类错误，还是基础 JSON/字段缺失。

只有当主要错误已经从“格式不可用”转为“任务判断可优化”时，SFT 才完成了 GRPO 冷启动职责。数值阈值应基于实际 reward 分布确定，本文不虚构一个文献并未支持的统一百分比。

## 13. 主要风险与应对

| 风险 | 影响 | 首选控制 |
|---|---|---|
| description/cot token 占比过高 | 总 loss 好看但核心分类不足 | 限长、字段级指标、核心两字段对照 |
| 教师 cot 事后合理化 | 学到流畅但不真实的解释 | cot 最后、只保留 ref、机械一致性检查、CoT 消融 |
| description 泄漏类别判断 | 描述失去客观性并放大标签偏差 | description 固定为观察事实，字段规则静态检查 |
| 多种等价 JSON 序列 | 增加格式熵和 exact-match 错误 | 固定顶层 key 顺序；evidence 按集合语义评估 |
| 视觉或 JSON 尾部被截断 | 训练错误或永远缺尾字段 | 编码后检查；先控像素/帧/文本长度 |
| 只训练短核心 target | GRPO actor 不满足生产 schema | 核心 target 只作诊断，不作为最终 actor |
| 同 prompt 多种 schema | 生成模式不稳定 | 每个 schema 使用独立 task prompt；主线只用生产 schema |
| R5 一致被误当成视觉正确 | 将错误证据合理化 | 区分映射一致性与视觉事实正确性；沿用既有 ref 准入 |

## 14. 已确认方案

建议正式确认以下 SFT 方案：

1. 主 SFT 对每条完整生产 JSON `{description,evidence,risk_type,cot}` 做 assistant-only loss。
2. 固定 canonical 字段顺序为 `description -> evidence -> risk_type -> cot`；这是项目方案，不冒充官方最优顺序。
3. cot 继续保留 1～3 个短步骤，但定位为“确认 evidence+risk_type 的可核验决策记录”，并放在 risk_type 后。
4. evidence 采用集合语义；首版不强制内部顺序或去重，set-F1 忽略排列和重复。
5. target 只取已准入的四个 `ref_*` 值；不包含 human_label、candidate、R5 输出和二元结果。
6. 第一版使用普通 token-level loss，不做字段加权、不增辅助头、不做多任务/课程学习。
7. A是唯一正式actor；B/C仅作诊断短跑，不作为生产候选。
8. GRPO 从完整 JSON SFT adapter 初始化，并以同一 SFT 初始策略冻结快照作为 reference；`beta=0.04`、`kl_in_reward=false`。
9. GRPO reward 使用已确认的渐进格式分、严格语义门和 `risk_type/evidence/R5/cot` 分解信号，不使用单一 R5 outcome。
10. GRPO completion 首轮硬上限为512；最终 `L_soft/L_max` 由合法参考JSON的P95/P99加余量确定。SFT与GRPO都禁止静默截断。
11. 关闭原生thinking；JSON中的 `cot` 仍为1～3个简短的evidence→risk_type确认步骤。

## 15. 一手来源索引

- [Qwen3-VL-8B-Instruct 官方模型卡](https://huggingface.co/Qwen/Qwen3-VL-8B-Instruct)
- [Qwen3-VL 官方仓库](https://github.com/QwenLM/Qwen3-VL)
- [Transformers：Chat templates](https://huggingface.co/docs/transformers/chat_templating)
- [Transformers：Multimodal chat templates](https://huggingface.co/docs/transformers/chat_templating_multimodal)
- [Transformers：Causal language modeling](https://huggingface.co/docs/transformers/main/tasks/language_modeling)
- [TRL：SFT Trainer](https://huggingface.co/docs/trl/en/sft_trainer)
- [TRL：GRPO Trainer](https://huggingface.co/docs/trl/en/grpo_trainer)
- [MS-SWIFT：Custom dataset](https://github.com/modelscope/ms-swift/blob/main/docs/source_en/Customization/Custom-dataset.md)
- [MS-SWIFT：Command-line parameters](https://github.com/modelscope/ms-swift/blob/main/docs/source_en/Instruction/Command-line-parameters.md)
- [RFC 8259：JSON 标准](https://www.rfc-editor.org/rfc/rfc8259)
- [DeepSeekMath：GRPO 原始论文](https://arxiv.org/abs/2402.03300)
- [DeepSeek-R1：cold-start 与多阶段训练](https://arxiv.org/abs/2501.12948)
- [Chain-of-Thought Prompting 原始论文](https://arxiv.org/abs/2201.11903)
- [STaR 原始论文页面](https://research.google/pubs/star-self-taught-reasoner-bootstrapping-reasoning-with-reasoning/)
- [Multimodal Chain-of-Thought 原始论文](https://arxiv.org/abs/2302.00923)
- [Turpin et al.：CoT 忠实性限制，NeurIPS 2023](https://proceedings.neurips.cc/paper_files/paper/2023/hash/ed3fea9033a80fea1376299fa7863f4a-Abstract.html)
