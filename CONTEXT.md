# 内容审核领域

本上下文描述图片与 GIF 内容审核中的业务概念，用于统一数据制作、模型训练和推理输出所使用的语言。

## Language

**审核结果**：
一次内容审核最终产生的结果，同时包含二元判定和分类判定；当前阶段不以验收门槛定义它。
_Avoid_: 验收结果、模型原始输出

**二元判定**：
审核结果中表示内容属于 `good` 还是 `risk` 的判定；其映射逻辑归 R5 所有，不属于 VLM 的输出职责。
_Avoid_: 二分类标签、是否违规分类

**业务分类**：
审核结果中由 R5 输出的 10 类最终分类，字段名为 `vlm_advice`；它与业务真值 `human_label` 使用同一套枚举。
_Avoid_: `risk_type`、内容细类

**业务真值**：
经数据制作最终确定的 10 类人工标签，字段名为 `human_label`；它与业务分类使用同一套枚举，但不与 46 类内容细类直接比较。
_Avoid_: `risk_type`、`ref_advice`

**候选标注**：
本地模型或外部标注模型生成、尚未获准用于训练的 `candidate_*` 字段集合。
_Avoid_: 参考标注、训练真值

**外部 API 媒体许可**：
项目图片和 GIF 已确认可发送至数据集制作所用的外部 API；首版不按媒体出域等级拆分生成路线，但该许可不代表 API 已被确定为主生成器，API 与本地模型的职责须经实际 pilot 决定。两条路线都使用现有 `md5` 关联原数据，并保存精确模型快照、请求标识、prompt/taxonomy 版本及原始响应以支持追溯。
_Avoid_: API 自动真值、无需追踪的外部标注

**GIF 最多四帧输入**：
GIF 不超过4帧时使用全部帧，超过4帧时按累计播放时间的首帧、`1/3`、`2/3` 和末帧位置选取4帧；有序帧作为一个 video 帧列表输入。数据制作 API、本地模型、SFT、GRPO、评估和推理必须使用同一规则。
_Avoid_: 四张独立 image、仅首帧 GIF、教师与 actor 使用不同帧、按运行随机抽帧

**动态视觉预算**：
静态图作为单个 image，在完整保留画面和宽高比的前提下由 Qwen3-VL processor 使用动态分辨率，视觉上限为1024 token（`max_pixels=1048576`）；GIF video 每帧上限为256 token（`max_pixels=262144`）。超限媒体只做确定性等比例缩小，不裁剪、拉伸或随机增强。
_Avoid_: 固定缩放到1024×1024、静态图完全不缩放、GIF沿用静态图1024 token/帧

**MD5 分区隔离**：
首版只使用现有 `md5` 识别完全重复媒体，同一 `md5` 的记录不得跨 train/dev；不使用 pHash、视觉向量、模板近重复或 `source` 分组。
_Avoid_: 近重复聚类、来源分组、相同 MD5 跨分区

**训练开发划分**：
参考四字段完成后，以 `md5` 为隔离单位按90%/10%划分 train/dev，并尽量保持10类 `human_label` 与46类 `ref_risk_type` 的比例；随机种子固定并记录，SFT 与 GRPO 共用同一划分。dev 用于训练诊断、checkpoint 选择和调参，不是最终验收集。
_Avoid_: 随机重新划分、SFT/GRPO各用一套分区、把dev称为验收集

**参考标注**：
候选标注通过格式、枚举、CoT 一致性和 R5 映射检查，并由数据负责人确认可用于训练后形成的 `ref_*` 字段集合；人工检查流程不在本上下文中规定。
_Avoid_: 候选标注、模型预测

**主风险域**：
多风险训练样本中由 `human_label` 指定的唯一业务类别；该样本的主内容细类必须属于这个类别对应的小类集合。
_Avoid_: 任意最高风险、R5 推测类别

**内容细类**：
VLM 输出的 46 类中间语义分类，沿用字段名 `risk_type`；每次审核只取一个主内容细类，它不是最终业务分类。当前 `risk_type.md` 是包含非违规和弱风险细类的封闭枚举，变更时必须显式升级版本。
_Avoid_: 最终分类、业务分类、46 类风险

**证据**：
VLM 输出的多值风险信号集合，字段名为 `evidence`；它可以同时描述同一内容中跨风险域的多个真实信号，只能使用登记在版本化清单中的封闭枚举值。数组内部顺序不承载语义，首版不为重复项设置专门规则。除 4 个 `*_other` 内容细类允许空集合外，其他内容细类必须至少有一项证据。
_Avoid_: `risk_type`、业务分类

**内容描述**：
对输入中可观察文字、主体和页面元素的客观概述，字段名为 `description`；它不进行风险归类或规则判断。
_Avoid_: 判定理由、风险结论

**判定理由**：
先确认可观察内容对每项证据的支持，再确认这些证据对主内容细类的支持的简短说明，字段名为 `cot`，类型固定为包含 1～3 项的字符串数组；它必须显式引用输出的 evidence 和 risk_type，不得引入数组之外的证据，并且不作为 R5 输入。它是项目定义的可检查字段，不是模型原生 thinking。
_Avoid_: `description`、R5 规则说明、自由推理长文、`<think>` 内容

**VLM 中间结果**：
VLM 以单一 JSON 对象产生的判定材料，canonical 字段顺序固定为 `description → evidence → risk_type → cot`；其中 R5 只消费 `risk_type` 与 `evidence`，该结果不包含最终业务分类或二元判定。
_Avoid_: 审核结果、最终结果

**R5 规则映射器**：
根据 VLM 产生的内容细类和证据生成 10 类业务分类及置信度的既有黑盒规则；它是业务分类的唯一来源。
_Avoid_: 分类模型、R4

**R5 规则冲突**：
真实的参考内容细类和证据经 R5 得到的业务分类与 `human_label` 不一致的样本状态；冲突解决前，该样本不具备主训练集准入资格。
_Avoid_: 模型错误、普通困难样本

**训练编排框架**：
统一承载 SFT、GRPO、多模态数据模板、LoRA adapter 接续、自定义奖励和分布式训练配置的上层框架；本项目选用 MS-SWIFT 标准 Transformers 后端，不使用 Megatron-SWIFT。
_Avoid_: 裸 PEFT/TRL、Megatron-SWIFT

**八卡原地 Rollout**：
单机8张V100全部作为DeepSpeed ZeRO-3训练rank；GRPO由这8个训练进程共同使用Transformers生成completion，不预留两张专用rollout卡。首版显式设置 `use_vllm=false` 与 `ds3_gather_for_generation=false`，防止生成前把完整policy收集到单卡。
_Avoid_: 6卡训练+2卡rollout、两卡LMDeploy等同GRPO rollout、单卡完整policy聚合

**GRPO Generation Batch**：
一次采样批次跨全部训练进程产生的completion总数，对应 `generation_batch_size`；首版固定为8。配合 `num_generations=4` 时，它表示2个Prompt实例各生成4个completion，而不是8套Prompt模板或每卡8个completion。
_Avoid_: generation_batch_size=16首轮对照、每卡batch=8、多Prompt模板

**真实长度分组**：
SFT与GRPO均启用 `group_by_length`，但分组依据必须是processor展开后的真实编码长度并包含视觉token；SFT length cache包含四字段target，GRPO length cache只包含prompt。保持数据shuffle，禁用packing与padding-free。
_Avoid_: 按文本字符数分组、SFT/GRPO盲用同一长度、group_by_length等同packing

**完整样本长度策略**：
GRPO completion首轮上限为512，最终软/硬边界由合法参考completion的P95/P99加余量确定；prompt与SFT总长度在正式taxonomy渲染后统计。所有阶段对超长样本使用 `truncation_strategy=delete` 并记录，不截断taxonomy、视觉token或四字段JSON。
_Avoid_: 左截断system taxonomy、右截断JSON、依靠截断解决视觉OOM

**V100 Attention 基线**：
训练与Transformers推理首选PyTorch SDPA，当前组合不兼容时才回退eager；V100不使用FlashAttention-2，同时禁用依赖FlashAttention的padding-free。
_Avoid_: FlashAttention-2、未验证kernel即宣称SDPA优化生效

**独立推理对照**：
不参与GRPO的推理基准：一条路线使用Transformers两卡手动均衡/`balanced_low_0`，另一条使用LMDeploy PyTorchEngine TP=2且vision batch为1；用真实样本比较显存、吞吐、OOM、JSON有效率和输出一致性后，决定数据制作、评估或部署后端。
_Avoid_: LMDeploy直接替换MS-SWIFT rollout、独立推理结论直接外推GRPO batch

**ZeRO-3 Offload 回退**：
默认使用纯GPU ZeRO-3；只有定位动态峰值、确认视觉/长度预算并调整bucket后仍OOM，才启用CPU parameter offload。LoRA optimizer状态较小，optimizer offload不是主要显存手段；首版不使用NVMe offload。
_Avoid_: 默认zero3_offload、NVMe无条件兜底、未定位OOM直接QLoRA

**Non-thinking Actor**：
Qwen3-VL-8B-Instruct在SFT、GRPO、评估和部署中关闭模型原生thinking；模型只输出单个四字段JSON。JSON内的 `cot` 保留为简短evidence→risk_type确认，不允许JSON前后出现 `<think>` 或自由推理文本。
_Avoid_: Thinking模型、JSON前缀推理、把cot删掉以关闭thinking

**原生 Hugging Face 备用栈**：
由 `Transformers + TRL + PEFT + Accelerate/DeepSpeed` 直接组合的备用训练实现；只有 MS-SWIFT 出现无法绕过的阻断性问题时才启用，不与主框架并行维护。
_Avoid_: 第二套主框架、默认实现

**SFT 冷启动模型**：
对完整四字段生产 JSON 进行 assistant-only 监督训练后得到的 LoRA 模型；它既是后续 GRPO 的初始策略，也定义 GRPO 应持续遵守的输出契约。
_Avoid_: description-only 模型、核心两字段模型

**CoT 自洽奖励**：
GRPO 中用于检查 `cot` 是否覆盖模型输出的全部 evidence、是否未引入数组外 evidence、是否确认同一 risk_type，以及是否先确认 evidence 再确认 risk_type 的机械一致性分；它不证明视觉事实真实，也不代表模型内部推理忠实。
_Avoid_: CoT 正确性奖励、视觉事实奖励

**Evidence 集合奖励**：
GRPO 中将预测 evidence 与参考 evidence 分别转成集合后计算的 F1 分数，同时惩罚漏报与多报；数组顺序和重复项不影响该语义分，首版也不为重复设置额外约束；非法枚举和不允许的空集合仍由语义门拦截。
_Avoid_: evidence 数组字符串匹配、recall-only 奖励

**GRPO 语义门**：
模型 completion 只有同时满足严格 JSON object、精确四键、字段类型/非空约束以及封闭枚举/空集规则时，才有资格获得 risk_type、evidence、R5 和 CoT 语义奖励并调用 R5；canonical key 顺序错误只扣格式分，不关闭该门，evidence 数组顺序和重复不参与门控。
_Avoid_: 正则抢救字段、非法输出调用 R5

**R5 结果奖励**：
GRPO 中将模型预测的 risk_type/evidence 输入锁定版本 R5，并检查其10类 `vlm_advice` 是否等于 `human_label` 的二元分；它不使用 R5 confidence、二元映射或内部规则路径，基础设施异常也不作为模型负反馈。
_Avoid_: R5 confidence 奖励、二元结果奖励、R5 路径奖励

**软超长惩罚**：
GRPO 中只对超过合法参考输出 token 长度 P95 的 completion 施加线性扣分，并在 `L_max` 达到最大扣分 0.03；它不奖励短输出，也不替代完整四字段格式要求。
_Avoid_: 短输出奖励、硬长度奖励

**重复率监控**：
用于离线评估和训练诊断的 token 重复统计；首版不把它作为 GRPO 奖励或惩罚分量，避免与长度惩罚重复计权。
_Avoid_: 重复惩罚、repetition reward

**批次奖励缩放**：
GRPO 对每个 prompt 的 completion 奖励先做组内中心化，再除以整个批次的奖励标准差；本项目对应 MS-SWIFT 的 `scale_rewards=batch`，用于稳定更新尺度，同时保留不同 prompt 奖励差距的相对大小。
_Avoid_: group reward scaling、逐奖励分量缩放

**GRPO 参考策略**：
GRPO policy 从 SFT adapter 初始化，并将该初始策略的冻结快照作为 KL reference；首版使用 `beta=0.04`，且通过 `kl_in_reward=false` 将 KL 作为独立损失约束，不把它混入业务 reward 后再缩放。
_Avoid_: 持续更新的 reference、无 KL、reward 内 KL

**静态 GRPO 采样**：
首版保留原始 prompt 组并设置 `dynamic_sample=false`；组内 reward 零方差只记录和归因，不自动丢弃后重新 rollout，避免隐式改变训练分布或掩盖系统性失败。
_Avoid_: 动态采样、自动过滤零方差组

**Prompt 实例**：
由 canonical actor 任务指令模板与一个静态image或一个GIF video帧列表组成的单次模型输入；每条媒体对应一个实例，`human_label` 和 `ref_*` 只供训练采样与奖励计算使用，不进入 actor 可见内容。GRPO 对同一实例生成多个 completion，并在该组内比较相对奖励。
_Avoid_: prompt 文案变体、多轮对话、标签可见 actor prompt

**数据集制作 Prompt**：
仅用于生成 `candidate_*` 的离线提示词；它可以读取媒体、`human_label`、完整 taxonomy、详细边界规则和候选生成要求，但不进入 SFT、GRPO、评估或部署，使用 `dataset_prompt_version` 独立追踪。
_Avoid_: actor prompt、生产 prompt

**Actor Prompt**：
Qwen3-VL actor 在 SFT、GRPO、评估和部署阶段共同使用的 canonical 输入模板；它包含紧凑 taxonomy 与四字段 JSON schema，但严禁包含 `human_label`、`candidate_*`、`ref_*`、R5 结果/规则或二元标签，使用 `actor_prompt_version` 追踪。
_Avoid_: 数据集制作 prompt、标签可见 prompt

**Taxonomy Schema**：
数据集制作 prompt 与 actor prompt 共同派生分类信息的版本化来源；制作侧使用完整规则与边界，actor 侧使用完整枚举、简短定义、risk_type-evidence 映射和空集规则的紧凑渲染，使用 `taxonomy_schema_version` 追踪。
_Avoid_: 两套独立分类标准、未版本化枚举

**类别覆盖**：
训练集对 10 类 `human_label` 和 46 类 `risk_type` 的实际样本覆盖；本项目在训练前通过定向数据收集解决不足，首版训练不使用标签加权 sampler 或类别 reward 倍率。首版10,000条有效样本采用 `good/其余9类=65%/35%`：good 6,500条，其中 `good_clean=4,500`、`good_risk_signal=2,000`；后者按 porn相似450、finance相似400、gamble/game相似300、login/客服/二维码/线路相似300、fake/官方/品牌相似300、politic/军警/新闻相似150、跨域多弱信号100采集，最终参考不构成good的样本必须移出该桶。其余9类共3,500条，44个非-good `ref_risk_type` 每类至少60条，另有860条定向补充。首版只固定2,000条 `good_risk_signal` 困难/边界配额；非-good困难正例机会性保留和统计，不设总量或逐类硬指标，不阻塞10k完成。采用两轮收集：第一轮按10类配额收集并生成参考，第二轮只按自动盘点结果补46类和good混淆方向缺口；期望答案只用于外部选材，不注入数据制作Prompt。该二桶描述仅用于采集，不代表固定二元映射；10k是否充分由2.5k/5k/7.5k/10k嵌套学习曲线判断。
_Avoid_: 类别重加权、平衡 sampler、稀有类奖励加成

**生产相关能力保持集**：
独立于训练数据的200～500条诊断样本，用于比较 base、SFT 与 GRPO 在中文 OCR、UI/文档、细节识别、GIF时序及四字段 JSON 稳定性上的变化；它不参与训练，也不是最终验收集。
_Avoid_: 公开训练混合、最终盲测集、通用聊天榜单

**定向能力 Replay**：
仅在能力保持集确认 adapter 加载后发生生产相关能力退化时启用的条件性数据混合；先测试5%再测试10%，公开素材须重新制作为同 actor prompt、同 taxonomy、同四字段 schema 且具备完整 reference/R5 准入的数据，比例同时按 assistant token 与 visual token 统计。公开原生 VQA、caption、聊天或无业务 reward reference 的数据不进入首版 SFT/GRPO。
_Avoid_: 默认公开数据混合、原生公开答案直训、无R5标签的GRPO样本

**GRPO 组大小**：
同一 prompt 实例一次 rollout 生成并参与组内相对比较的 completion 数量，对应 `num_generations`；首版优先使用 `4`，若 smoke test 不能稳定运行则降为 `2`。
_Avoid_: 固定为 2、num_generations=8

**GRPO rollout 温度**：
训练时对同一 prompt 生成多个候选所使用的随机采样强度；本项目设置 `do_sample=true`、`temperature=0.9` 以兼顾组内差异和结构化 JSON 稳定性，评估不复用该训练温度。
_Avoid_: 贪心 rollout、评估温度

**GRPO rollout 截断**：
训练候选 token 的概率截断方式；本项目设置 `top_p=0.95`、`top_k=-1`、`num_beams=1`，保留 nucleus sampling 但不使用固定 top-k 或 beam search，并以 `repetition_penalty=1.0` 明确关闭生成阶段重复惩罚。
_Avoid_: top-k 截断、beam search、生成重复惩罚
