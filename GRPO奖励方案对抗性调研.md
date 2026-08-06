# GRPO 奖励方案对抗性调研

> 调研日期：2026-08-05  
> 状态：首版奖励方案已确认；数据相关长度统计保留到正式参考集完成后计算  
> 范围：Qwen3-VL-8B-Instruct、完整四字段 JSON、SFT→GRPO、MS-SWIFT Transformers rollout、FP16 LoRA、8×V100。本文不决定最终验收口径。

## 1. 执行结论

不存在脱离本项目数据就能证明的“全局最优权重”。本文所称最优，是指在当前输出契约、R5 黑盒、标签条件和 V100 预算下，经过奖励投机审查后最稳妥的**首版可执行配置**。

推荐采用：

- 严格 JSON parser，不从非法文本中用正则抢救 `risk_type/evidence`。
- 渐进的格式分 `r_fmt`，但只有完整 schema、类型和枚举合法时才开启语义奖励。
- `risk_type` exact-match、`evidence` set-F1、R5 advice outcome 和低权重 CoT 自洽奖励。
- 不在线奖励 description 的自由文本语义；依靠完整 JSON SFT reference、非零 KL 和离线检查保持。
- 不把二元结果或 `vlm_confidence` 纳入奖励。
- 正向权重固定为：格式 0.20、risk_type 0.25、evidence 0.35、R5 0.15、cot 0.05。
- 另加最多 0.03 的软超长惩罚；不设置重复惩罚，重复率仅用于离线评估和训练监控。
- MS-SWIFT 首版使用 `scale_rewards=batch`、`kl_in_reward=false`、`beta=0.04`、`dynamic_sample=false`。
- 类别不平衡通过训练前的数据收集解决；首版使用常规随机采样，不设置标签加权 sampler 或类别相关 reward 倍率。

总奖励为：

\[
R = 0.20r_{fmt} + g(0.25r_{type}+0.35r_{ev}+0.15r_{R5}+0.05r_{cot})
    -0.03p_{len}
\]

其中 `g∈{0,1}` 是语义奖励门；正向权重之和为 1，理论值域约为 `[-0.03, 1.00]`。

## 2. 为什么不能只用 R5 命中奖励

主训练集已经要求参考 `risk_type/evidence` 经 R5 后与 `human_label` 一致。因此下面三个信号天然相关：

1. 预测命中 `ref_risk_type`；
2. 预测命中 `ref_evidence`；
3. 预测字段经 R5 后命中 `human_label`。

若三者都给高权重，就会把同一正确性重复奖励。更危险的是，R5 outcome 粒度只有 10 类，模型可能找到“字段不真实、但规则仍输出正确 advice”的捷径。

因此本文让 reference evidence 获得最高权重、reference risk_type 次之，R5 只以 0.15 权重作为端到端校验：

- reference 字段约束模型说出真实、完整的中间语义；
- R5 reward 允许语义近似但能产生正确业务结果的 completion 得到少量信用；
- R5 reward 不足以压过 reference 字段，降低规则投机收益。

这是一项项目设计推断。正式短跑必须记录各奖励分量的相关矩阵；若 `r_R5` 与其他分量几乎完全相关，其权重还可进一步降低。

## 3. MS-SWIFT 实际如何聚合奖励

当前 MS-SWIFT 源码先计算每个 completion 的加权总分：

\[
r_i=\sum_j w_j r_{i,j}
\]

再对同一 prompt 的 `G` 个 completion 减去组均值。`scale_rewards` 决定是否以及用什么标准差归一化：[MS-SWIFT `compute_advantages`](https://github.com/modelscope/ms-swift/blob/main/swift/rl_core/advantage.py)

| 配置 | 源码语义 | 本项目判断 |
|---|---|---|
| `group` | 除以每个 prompt 组内 reward 标准差 | 0.01 与 1.0 的差异都可能被放成近似同一更新幅度，不利于离散+连续混合奖励 |
| `batch` | 仍减组均值，但除以全批 reward 标准差 | **首选**；保留不同 prompt 的奖励差距量级，同时比 `none` 更稳定 |
| `none` | 只减组均值，不除标准差 | 第一备选；保留绝对尺度，但对权重和学习率更敏感 |
| `gdpo` | 每个奖励函数先做组内标准化，再加权并全局标准化 | 暂不选；risk/evidence/R5 高度相关，逐分量标准化可能再次放大重复信号 |

`batch` 不是经本项目实验证明的永久最优值。建议只把 `batch` 与 `none` 做一次短跑对照，不同时扩展到四种模式。

MS-SWIFT 支持多个 `reward_funcs`、`reward_weights`、自定义插件以及 reward 分量日志：[GRPO 参数](https://github.com/modelscope/ms-swift/blob/main/docs/source_en/Instruction/Command-line-parameters.md#grpo-arguments)、[自定义 reward](https://github.com/modelscope/ms-swift/blob/main/docs/source_en/Instruction/GRPO/DeveloperGuide/reward_function.md)。

## 4. 输入、参考值和严格解析

reward plugin 可读取：

- 模型 completion；
- `ref_risk_type`；
- `ref_evidence`；
- `human_label`；
- response token IDs；
- 当前 schema/R5 版本。

其中 `ref_*` 和 `human_label` 只进入 reward 代码，不能出现在训练/推理 actor prompt。数据集制作另用独立 prompt，可读取 `human_label` 生成候选，但该 prompt 不进入 SFT、GRPO、评估或部署。

严格解析必须满足：

- completion 除首尾空白外只能是一个 JSON object；
- 不接受 Markdown 代码围栏、自然语言前后缀或正则抽取结果；
- 检测重复 JSON key，不能让普通 `json.loads` 静默覆盖；
- 顶层键集合必须精确等于四个生产字段；
- `description` 是非空字符串；
- `evidence` 是字符串数组；
- `risk_type` 是字符串；
- `cot` 是包含 1～3 个非空字符串的数组；
- risk_type/evidence 均在版本锁定的封闭枚举内；
- 只有既定四个 `*_other` risk_type 允许空 evidence；数组内部顺序不承载语义，首版不检查重复项。

字段顺序已经确认为 `description → evidence → risk_type → cot`。reward plugin 必须从版本化 schema 读取这一 canonical key order，训练 serializer、离线评测 parser 与生产 parser 共用同一版本。

## 5. 格式分与语义门

定义布尔量：

- `J`：可被严格解析为单个 JSON object；
- `K`：顶层 key 集合精确正确；
- `T`：四字段类型、非空和 `cot[1..3]` 正确；
- `E`：risk/evidence 枚举及空集规则正确；
- `C`：四个 key 使用 canonical 顺序；evidence 数组顺序不参与该条件。

所有布尔量在公式中取 0 或 1，并按依赖逐级计算：

\[
r_{fmt}=0.25J+0.20JK+0.20JKT+0.25JKTE+0.10JKTEC
\]

\[
g=JKTE
\]

性质：

- 完全无法解析时 `r_fmt=0`，所有语义奖励关闭；
- 能解析但缺键、类型错误或枚举非法时可得到部分格式信号，但不能调用 R5；
- 语义合法、只是 canonical 顺序错误时 `g=1`，不会抹掉正确语义，只损失最后 0.10 的格式子分；
- 完全合法且 canonical 时 `r_fmt=1`。

不建议从非法 JSON 中用正则提取核心字段。那会奖励“对 parser 不可用、但对 reward 正则可用”的双重协议，模型很容易学会投机。

如果 SFT actor 多次采样后仍以不可解析 JSON 为主，应返回 SFT 修复，而不是继续细化非法文本奖励。GRPO 不应负责从零学习基础 schema。

## 6. `risk_type` 奖励

\[
r_{type}=\mathbb{1}[\hat t=t^*]
\]

- `\hat t`：预测 risk_type；
- `t*`：`ref_risk_type`；
- 非法枚举已经被 `g=0` 拦截。

不设置类别相似度或同一大类部分分。R5 的 10 类 outcome 已经提供业务层面的宽松信号；再给 46 类人为相似度会制造第二套隐藏映射。

## 7. `evidence` 集合奖励

令预测集合为 `P=set(pred_evidence)`，参考集合为 `S=set(ref_evidence)`，按集合计算：

\[
r_{ev}=\begin{cases}
1,& |P|=|S|=0\\
0,& \text{仅一个集合为空}\\
\frac{2|P\cap S|}{|P|+|S|},& \text{其他情况}
\end{cases}
\]

这就是 set-F1：

- precision 会惩罚 evidence 堆砌；
- recall 会惩罚为了触发 R5 而隐藏真实 evidence；
- 部分命中提供比 exact-match 更稠密的 GRPO 信号；
- 四个允许空 evidence 的 `*_other` 样本有明确边界。

第一版不使用 F2。虽然业务强调召回，但 F2 会降低多报 evidence 的代价，容易形成 evidence stuffing；类别召回优先应通过训练前的数据收集和后续误差切片处理。

## 8. R5 outcome 奖励

当且仅当 `g=1` 时调用 R5：

\[
r_{R5}=\mathbb{1}[R5(\hat t,P).advice=human\_label]
\]

约束：

- 不把 `vlm_confidence` 代入奖励。除非先证明其校准性和单调意义，否则模型可能通过堆叠规则信号抬高 confidence。
- 不加入二元 good/risk 奖励；二元映射归 R5 所有，而且会与 10 类 advice 再次重复计权。
- 对完全相同的 `(risk_type, evidence array, R5版本)` 可缓存 R5 结果；首版不为缓存另加 evidence 归一化规则。
- R5 服务或代码发生基础设施异常时应中止/跳过该训练批次并报警，不能把异常当成 `r_R5=0` 惩罚模型。

## 9. CoT 自洽奖励

CoT 的自动奖励只能验证“是否与输出字段自洽”，不能证明它是模型真实内部推理，也不能证明视觉事实正确。

从 cot 文本中只识别完整的、版本登记的 evidence/risk_type 字面量。令：

- `M`：cot 中出现的 evidence 枚举集合；
- `P`：模型输出的 evidence 集合；
- `coverage=|M∩P|/|P|`，当 `P=∅` 时，仅在 `M=∅` 时取 1；
- `no_extra=1` 当且仅当 `M⊆P`；
- `type_mention=1` 当且仅当 cot 明确出现预测 risk_type。
- `order_ok=1` 当且仅当全部预测 evidence 已在 cot 中得到提及，并且预测 risk_type 的确认出现在这些 evidence 提及之后；缺失任一 evidence 时该项为 0。预测 evidence 为空时，要求 cot 先显式写出 `evidence=[]`，再确认 risk_type。

\[
r_{cot}=0.40\,coverage+0.20\,no\_extra+0.20\,type\_mention+0.20\,order\_ok
\]

它只占总奖励 0.05，原因是模型仍可按顺序机械复制枚举来拿满分。观察内容是否真的支持 evidence，仍依赖高质量 `ref_*`、SFT、KL 和离线人工/模型检查，不能被这个分数替代。

对于空 evidence 的 `*_other`，还应在离线检查中验证 cot 是否包含风险域依据和回退说明；首版在线 reward 不用脆弱关键词把自然语言“理解”伪装成可靠判断。

## 10. description 为什么没有在线语义分

`description` 是自由文本，既没有封闭枚举，也不被 R5 消费。候选方法都存在明显问题：

- 与单条参考 description 做 BLEU/ROUGE 会惩罚正确改写；
- embedding 相似度不保证视觉事实正确；
- 在线 LLM/VLM judge 增加显存或外部调用成本，并引入 judge 偏差、随机性和新的奖励投机面；
- 把 `human_label` 给 judge 还可能鼓励事后合理化。

首版只在格式门中检查 description 为非空、长度合法的字符串；质量主要由完整 JSON SFT reference 和非零 KL 保存。必须单独记录空泛描述率、事实错误抽检和与 cot/evidence 的冲突率。如果短跑发现 description 明显退化，再另行研究可复现的视觉 judge，不能静默加入首版。

## 11. 长度与截断

### 11.1 软超长惩罚

先在全部 `ref_*` completion 上统计 tokenizer token 长度。定义：

- `L_soft`：参考 completion 长度的 P95，向上取整；
- `L_max`：GRPO 的 `max_completion_length`，必须高于参考 P99 并留出小幅余量；
- `L`：实际 completion token 数。

首轮运行固定 `max_completion_length=512` 作为保护上限；正式数据完成后再以P99加余量校准 `L_max`。GRPO prompt 使用独立的prompt-only真实编码长度，开启 `group_by_length`；超长输入使用 `truncation_strategy=delete`，不得左截断system taxonomy。模型原生thinking关闭，长度只统计单个四字段JSON completion。

\[
p_{len}=clip\left(\frac{L-L_{soft}}{L_{max}-L_{soft}},0,1\right)
\]

MS-SWIFT 内置 `soft_overlong` 使用等价的 token 长度线性惩罚，可直接复用。[MS-SWIFT reward 源码](https://github.com/modelscope/ms-swift/blob/main/swift/rewards/orm.py)

被 `L_max` 截断且没有形成完整 JSON 的 completion 同时会失去格式与全部语义奖励。不能通过奖励鼓励删除必需字段来换取更短输出。

### 11.2 重复率仅监控

最终需求不设置重复惩罚。重复输出与超长输出存在相关性，另设奖励分量会增加重复计权和中文正常复述被误伤的风险。训练期间仍应记录 token 级重复率，作为异常生成的诊断指标；若后续实测发现大量未超长但高度循环的输出，再单独提交奖励变更，不在首版预置。

MS-SWIFT 内置 repetition reward 使用 `text.lower().split()` 的词级 n-gram，中文通常没有空格，因此它也不适合作为本项目的默认监控实现。[MS-SWIFT `RepetitionPenalty`](https://github.com/modelscope/ms-swift/blob/main/swift/rewards/orm.py)

## 12. 最终总公式与 MS-SWIFT 映射

### 12.1 总公式

\[
R = 0.20r_{fmt} + g(0.25r_{type}+0.35r_{ev}+0.15r_{R5}+0.05r_{cot})
    -0.03p_{len}
\]

| 分量 | 范围 | 权重 | 主要职责 |
|---|---:|---:|---|
| `r_fmt` | `[0,1]` | 0.20 | JSON、schema、类型、枚举、canonical 形式 |
| `r_type` | `{0,1}` | 0.25 | 46 类精确命中 |
| `r_ev` | `[0,1]` | 0.35 | 证据集合完整且不过报 |
| `r_R5` | `{0,1}` | 0.15 | 最终 10 类业务结果校验 |
| `r_cot` | `[0,1]` | 0.05 | 与模型自身核心字段机械自洽 |
| `p_len` | `[0,1]` | -0.03 | 软超长 |

正向权重之和为 `1.00`。格式不合法时 `g=0`，不会调用 R5；总分仍可能由渐进格式分和长度惩罚区分。

### 12.2 推荐配置

```text
reward_funcs = [format, risk_type, evidence_f1, r5_advice, cot_consistency,
                soft_overlong]
reward_weights = [0.20, 0.25, 0.35, 0.15, 0.05, 0.03]
scale_rewards = batch
kl_in_reward = false
beta = 0.04
dynamic_sample = false
disable_dropout = true
use_vllm = false
```

其中 `soft_overlong` 函数自身返回 `[-1,0]`，所以权重保持正数。

建议每个 reward 分量作为独立函数输出，便于 MS-SWIFT 记录均值、标准差和相关性；所有函数共用同一个严格 parser 结果，避免解析逻辑漂移。

## 13. KL 设置

KL 不是普通 reward。首版使用：

- SFT adapter 初始化 policy；
- SFT adapter 的 GRPO 初始状态作为冻结 reference；
- `beta=0.04`；
- `kl_in_reward=false`，即 KL 作为独立 loss 项，不先混进 reward 再归一化；
- `disable_dropout=true`，减少 policy/reference log-prob 随机差异。

`0.04` 是 MS-SWIFT 当前 GRPO 默认值，也是 DeepSeekMath GRPO 的常见起点，不是本项目已证明的最优值。[DeepSeekMath](https://arxiv.org/abs/2402.03300)、[MS-SWIFT 参数](https://github.com/modelscope/ms-swift/blob/main/docs/source_en/Instruction/Command-line-parameters.md#grpo-arguments)

只建议做 `0.02 / 0.04 / 0.08` 的小范围对照：

- KL 过低：description/cot/JSON 可能随 R5 reward 漂移；
- KL 过高：会把 SFT 标签错误和保守输出持续锚定，抑制 GRPO 改进。

## 14. 类别不平衡与零方差

### 14.1 类别不平衡

不要给不同 human_label/risk_type 设置不同 reward 倍率。GRPO 先做同 prompt 组内比较，类别倍率与 advantage normalization 容易相互抵消或扭曲更新尺度。

最终需求选择在训练前通过定向数据收集解决类别不平衡。首版对整理后的训练集使用常规随机打乱与采样，不依据 `human_label` 或 `risk_type` 设置加权/平衡 sampler。数据准备阶段应补充稀有类别、困难负样本和已知误报切片，并避免用完全相同的媒体/target 伪造样本量；reward 公式在所有类别上保持同一尺度。

### 14.2 组内 reward 零方差

MS-SWIFT/TRL 记录 `frac_reward_zero_std`。同组全对、全错、全部格式失败或输出完全相同都会产生零 advantage。[TRL GRPO metrics](https://huggingface.co/docs/trl/en/grpo_trainer#logged-metrics)

首版保持 `dynamic_sample=false`：自动重采样会同时丢弃已学会的简单样本和完全失败的困难样本，可能改变训练分布，还会在 V100 上增加 rollout 成本。

先区分零方差原因：

- 全部正确：降低该类样本采样率；
- 全部格式失败：返回 SFT/serializer 修复；
- 全部同一错误：提高采样温度或加入边界样本；
- reward 过粗：检查 evidence F1、格式子分是否真正产生差异。

只有确认大量零方差来自“已解决的简单 prompt”时，才考虑动态采样或显式 hard-example sampler。[DAPO](https://arxiv.org/abs/2503.14476)

## 15. 对抗性攻击表

| 攻击/失败模式 | 如果奖励设计不当 | 本方案防线 | 剩余风险 |
|---|---|---|---|
| 非法 JSON 中塞入正确枚举 | 正则 reward 给高分，生产 parser 失败 | 严格 JSON；`J=0` 时语义全关 | 一组全非法仍零方差，应修 SFT |
| Markdown 围栏包 JSON | 人看正确但生产协议不符 | 只允许单个 object，无前后缀 | 无 |
| 重复 key 覆盖前值 | parser 与 reward 对同一文本理解不同 | duplicate-key 检测 | 实现必须测试 |
| 增加 `human_label/vlm_advice` key | 泄漏或自报最终答案 | exact key set；额外 key 关闭语义门 | 无 |
| 输出非法 risk/evidence | 可能触发 R5 异常或默认分支 | 封闭枚举门控；不调用 R5 | 无 |
| evidence 堆砌 | recall 高、R5 可能命中 | set-F1 precision 惩罚 | reference evidence 不全会反向伤害 |
| 隐藏跨域 evidence | 利用 R5 获得正确 advice | set-F1 recall，evidence 权重最高 | 依赖 ref_evidence 完整性 |
| 错字段但映射到同一 advice | 只用 R5 时拿满分 | risk/evidence 合计 0.60，R5 仅 0.15 | R5 仍提供少量捷径分 |
| 永远预测多数类 good | 在不平衡数据上表面收益高 | 每 prompt 参考/R5 reward；训练前补齐稀有类数据 | 数据收集后仍需检查真实类别覆盖 |
| cot 只复制枚举 | 机械一致性拿满分 | cot 仅 0.05，不代表视觉正确 | 仍可能生成空泛 cot |
| cot 增加数组外 evidence | 文本显得“充分” | `no_extra` 子项 | 自然语言同义词难检测 |
| description 编造事实 | R5 完全看不到 | 非零 KL、离线事实检查；不使用伪精确 judge | 首版在线无法完全防止 |
| description 输出空泛套话 | 满足非空格式 | SFT+KL、长度/空泛率监控 | 发现后需另研 judge |
| 超长 cot 获取更多“推理”信用 | rollout 成本上升 | cot 权重低、P95 软超长 | 阈值依赖真实 token 分布 |
| 截断在核心字段之后 | R5 字段看似可抽取但 JSON 不完整 | 严格完整 object，格式失败 | max length 配置必须安全 |
| 中文输出出现循环重复 | 浪费 rollout token，可能不影响语义分 | token 重复率监控 + P95 软超长 | 若短输出循环显著出现，再单独评估奖励变更 |
| 操纵 `vlm_confidence` | 堆叠规则信号换高分 | confidence 不进入 reward | 无 |
| R5 服务故障 | 错把基础设施问题当模型错误 | fail-fast/跳批并报警 | 需要稳定的本地接口和缓存 |
| 所有 completion 同分 | GRPO 无 advantage | 分量监控、SFT readiness、sampler调整 | GRPO 的固有边界 |
| reward 分量高度相关 | 同一正确性重复放大 | R5 低权重、batch scaling、相关矩阵 | 权重仍需短跑校准 |
| 在线 LLM judge 被诱导 | 图片文字/输出攻击 judge | 首版不使用在线 judge | description 无稠密语义分 |

## 16. 伪代码

```python
def rewards(completion, ref_risk_type, ref_evidence, human_label,
            response_token_ids, schema, r5):
    parsed = strict_parse(completion, schema)  # also detects duplicate keys

    J = parsed.is_single_json_object
    K = J and parsed.has_exact_keys
    T = K and parsed.has_valid_field_types_and_nonempty_values
    E = T and parsed.has_legal_enums_and_empty_rule
    C = E and parsed.is_canonical_order

    r_fmt = (
        0.25 * J
        + 0.20 * (J and K)
        + 0.20 * (J and K and T)
        + 0.25 * (J and K and T and E)
        + 0.10 * (J and K and T and E and C)
    )
    gate = float(J and K and T and E)

    if gate:
        r_type = float(parsed.risk_type == ref_risk_type)
        r_ev = set_f1(set(parsed.evidence), set(ref_evidence))
        r5_result = r5(parsed.risk_type, parsed.evidence)
        r_r5 = float(r5_result.advice == human_label)
        r_cot = cot_self_consistency(parsed.cot,
                                     parsed.evidence,
                                     parsed.risk_type,
                                     schema)
    else:
        r_type = r_ev = r_r5 = r_cot = 0.0

    p_len = soft_overlong(response_token_ids, schema.L_soft, schema.L_max)
    total = (
        0.20 * r_fmt
        + gate * (0.25 * r_type + 0.35 * r_ev
                  + 0.15 * r_r5 + 0.05 * r_cot)
        - 0.03 * p_len
    )
    return total
```

实际 MS-SWIFT plugin 建议分别返回各分量并共享解析缓存，以获得逐分量日志；上面合并写法只是展示数学语义。

## 17. 最小验证方案

### 17.1 离线 reward replay

在不训练模型的情况下，对参考 completion 和人工构造的攻击 completion 执行 reward：

- 完全正确 canonical JSON 应接近 1；
- 正确语义但 key 顺序不 canonical，应只损失格式子分；evidence 数组换序不应损失任何分数；
- R5 命中但 risk/evidence 错误，分数必须显著低于 reference 命中；
- evidence 每漏一项或多一项，set-F1 应单调下降；
- 非法 JSON、额外 key、非法 enum 不得调用 R5；
- R5 基础设施异常必须触发测试失败，而不是返回模型负分；
- 空/空、空/非空、非空/空 evidence 的边界与公式一致；
- 正常与异常输出的 token 重复率应分开记录，确认首版无须将其加入奖励。

必须将第 15 节攻击表固化为 reward 单元测试。

### 17.2 GRPO 短跑

至少记录：

- 各 reward 分量的 mean/std/P5/P50/P95；
- 分量两两 Pearson/Spearman 相关性；
- 总 reward、advantage、KL 和 clip ratio；
- JSON parse、schema、enum、canonical 顺序通过率；
- risk_type accuracy；
- evidence precision/recall/F1 与预测集合大小；
- R5 advice 命中率；
- cot coverage/no-extra/type-mention；
- completion P50/P95/max token、截断率和 token 重复率；
- `frac_reward_zero_std`，并按全对/全错/格式失败拆因；
- 按 10 类、46 类、静态图/GIF 的 reward 分布；
- R5 异常、parser 异常和 reward NaN 次数。

短跑只比较：

1. `scale_rewards=batch` 与 `none`；
2. `beta=0.02/0.04/0.08`；
3. 若 R5 与核心字段相关度过高，`w_R5=0.15` 与更低值。

不要同时搜索所有 reward 权重，否则无法归因，也容易在小开发集上调出偶然最优。

## 18. 何时停止或回退

出现以下任一情况，应停止 GRPO 并修复前置环节：

- SFT actor 的 JSON parse/schema 通过率仍低，导致多数 group 全部格式失败；
- reward parser 与生产 parser 对同一输出理解不一致；
- R5 调用存在非确定性或基础设施错误；
- `ref_evidence/ref_risk_type` 存在系统性视觉错误；
- 总 reward 上升但 reference evidence F1、risk_type 或 R5 结果下降；
- evidence 集合持续变大、description/cot 明显空泛或输出长度持续逼近上限；
- KL 快速升高且自由文本退化；
- 大量 group reward 零方差，且无法通过 sampler/采样参数解释。

## 19. 未选择的方案

| 方案 | 不作为首版的原因 |
|---|---|
| 只用 R5 0/1 reward | 稀疏、可规则投机、忽略真实 evidence/risk_type |
| risk/evidence/R5 等权 | 高度相关，重复计权 |
| evidence exact-match only | 多 evidence 样本过于稀疏，缺少部分改进信号 |
| evidence F2 | 降低过报代价，容易堆砌 evidence |
| 使用 R5 confidence | 未证明校准与单调性，容易操纵 |
| 加二元结果 reward | 与 R5 advice 重复，且二元映射不归 VLM |
| description 的 BLEU/ROUGE | 惩罚正确改写，不保证视觉真实 |
| 在线 LLM/VLM judge | 成本、偏差、非确定性、prompt injection 和新投机面 |
| `scale_rewards=gdpo` | 当前分量相关性高，逐分量归一化可能放大重复信号 |
| 默认开启 dynamic sampling | 可能同时丢弃简单全对与困难全错样本，增加 V100 rollout |
| 类别相关 reward 倍率 | 与组相对 advantage/归一化纠缠；类别覆盖应在训练前通过数据收集解决 |
| 奖励课程自动换权重 | 引入非平稳目标；首版固定权重更易审计和归因 |

## 20. 一手来源

- [MS-SWIFT GRPO Get Started：算法、rollout、metrics](https://github.com/modelscope/ms-swift/blob/main/docs/source_en/Instruction/GRPO/GetStarted/GRPO.md)
- [MS-SWIFT Command-line parameters：reward、scale、KL、dynamic sample](https://github.com/modelscope/ms-swift/blob/main/docs/source_en/Instruction/Command-line-parameters.md)
- [MS-SWIFT 自定义 reward 开发文档](https://github.com/modelscope/ms-swift/blob/main/docs/source_en/Instruction/GRPO/DeveloperGuide/reward_function.md)
- [MS-SWIFT `compute_advantages` 源码](https://github.com/modelscope/ms-swift/blob/main/swift/rl_core/advantage.py)
- [MS-SWIFT reward 源码：repetition、soft overlong](https://github.com/modelscope/ms-swift/blob/main/swift/rewards/orm.py)
- [TRL GRPO Trainer：reward functions、KL、scale、zero-std metrics](https://huggingface.co/docs/trl/en/grpo_trainer)
- [DeepSeekMath：GRPO 原始论文](https://arxiv.org/abs/2402.03300)
- [DeepSeek-R1：cold-start、规则奖励与多阶段训练](https://arxiv.org/abs/2501.12948)
- [DAPO：动态采样、overlong reward 与训练稳定性](https://arxiv.org/abs/2503.14476)
- [Understanding R1-Zero-Like Training：GRPO 标准差缩放与长度偏差审查](https://arxiv.org/abs/2503.20783)
- [Scaling Laws for Reward Model Overoptimization](https://arxiv.org/abs/2210.10760)

## 21. 待确认决策顺序

按对抗性审查流程，建议逐项确认，不一次锁死全部细节：

1. “渐进格式分 + 严格语义门 + 分解任务奖励”的总体结构（已确认）；
2. 首版正向权重 `0.20/0.25/0.35/0.15/0.05`（已确认）；
3. description 首版不使用在线语义 judge（已确认）；
4. `scale_rewards=batch`（已确认）；
5. `beta=0.04` 且 KL 独立于 reward，并以 SFT 初始策略冻结快照为 reference（已确认）；
6. 首轮 `max_completion_length=512`；数据 token 统计完成后按P95/P99确定正式 `L_soft/L_max`（已确认计算原则）；
7. 数据集制作 prompt 与训练/推理 actor prompt 独立版本化；actor 使用一个 canonical 模板及紧凑 taxonomy，标签/R5 信息不可见，并由 SFT、GRPO、评估和部署共用（已确认）。每条媒体一个 actor prompt 实例、同一实例生成多个 completion（已确认）；`dynamic_sample=false`（已确认）；类别不平衡通过数据收集解决，首版不使用标签加权 sampler（已确认）；`num_generations=4` 为首选，smoke test 不能稳定运行时降为 `2`（已确认）；rollout 使用 `do_sample=true`、`temperature=0.9`、`top_p=0.95`、`top_k=-1`、`num_beams=1`、`repetition_penalty=1.0`（已确认）。
