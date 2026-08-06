# 数据集制作与训练 Prompt 规范

> 适用范围：Qwen3-VL-8B-Instruct，图片及最多4帧GIF，SFT → GRPO。  
> 本文定义两个严格分离的Prompt：数据集制作Prompt和生产Actor Prompt。  
> Prompt正文采用模板变量；发布时由版本化渲染器填充，禁止人工复制后形成另一套taxonomy。

## 1. 结论

项目使用两个Prompt体系：

| Prompt | 用途 | 可见标签 | 输出 | 使用阶段 |
|---|---|---|---|---|
| 数据集制作Prompt | 生成待确认候选四字段 | 仅可见原始 `human_label` | 四字段候选JSON，落库后映射为 `candidate_*` | 离线数据制作 |
| Actor Prompt | 生成生产判定材料 | 不可见任何标签、参考答案或规则结果 | `{description,evidence,risk_type,cot}` | SFT、GRPO、开发评估、部署 |

两者必须独立版本化：

```text
dataset_prompt_version
actor_prompt_version
taxonomy_schema_version
```

两者可以引用同一个taxonomy源，但渲染粒度不同：

- 数据集制作Prompt使用完整定义、详细边界和全部允许的evidence。
- Actor Prompt使用完整46类枚举、简短定义、risk_type-evidence映射和空集规则的紧凑渲染。
- SFT、GRPO、开发评估和部署必须使用字节级相同的Actor system/user文本，仅媒体和assistant响应是否存在发生变化。

## 2. 模板变量

| 变量 | 含义 | 出现位置 |
|---|---|---|
| `{{FULL_TAXONOMY}}` | 由锁定taxonomy版本生成的详细规则 | 数据集制作system |
| `{{COMPACT_TAXONOMY}}` | 同一taxonomy版本的紧凑完整渲染 | Actor system |
| `{{EVIDENCE_ENUM_JSON}}` | 完整evidence字符串数组的合法JSON | 数据制作JSON Schema |
| `{{RISK_TYPE_ENUM_46_JSON}}` | 46类risk_type字符串数组的合法JSON | 数据制作JSON Schema |
| `{{HUMAN_LABEL}}` | 原始10类业务标签之一 | 仅数据集制作user |
| `{{MEDIA}}` | 单个静态 image，或按时间顺序组成一个 video 帧列表的最多4帧GIF | 两种Prompt的多模态输入 |
| `{{REF_DESCRIPTION}}` | 已确认参考描述 | 仅SFT assistant target构造器 |
| `{{REF_EVIDENCE_JSON}}` | 已确认evidence数组的JSON序列化 | 仅SFT assistant target构造器 |
| `{{REF_RISK_TYPE}}` | 已确认单值risk_type | 仅SFT assistant target构造器 |
| `{{REF_COT_JSON}}` | 已确认cot数组的JSON序列化 | 仅SFT assistant target构造器 |

`md5`、`source`、文件路径、采集方向、期望补齐的 `risk_type` 等信息不进入任一模型Prompt。它们只在离线数据管道中使用。

## 3. Taxonomy渲染约束

### 3.1 单一来源

当前taxonomy语义来源为[risk_type.md](risk_type.md)。渲染器必须从版本锁定的结构化taxonomy生成 `FULL_TAXONOMY` 和 `COMPACT_TAXONOMY`，不能分别维护两份手写列表。

发布前必须满足：

1. risk_type总数精确等于46。
2. 每个risk_type具有唯一名称和非空定义。
3. 每个risk_type具有穷举的允许evidence集合。
4. evidence总枚举为封闭集合，Prompt中不得出现“等”“任一弱信号”等开放式占位语。
5. 只有 `gamble_other`、`porn_other`、`politic_other`、`fraud_other` 允许 `evidence=[]`。
6. `good_clean` 使用 `evidence=["no_risk_evidence"]`，不使用空数组。
7. risk_type与evidence的映射、空集规则和R5所使用的taxonomy版本一致，但Actor Prompt不出现R5名称或映射结果。

当前 `risk_type.md` 中 `good_risk_signal` 的“等”和 `fraud_single_weak` 的“任一弱信号”必须在结构化taxonomy中展开为具体封闭枚举后，才能生成正式Prompt。本文模板不会凭空补造未确认的evidence值。

### 3.2 详细渲染格式

`{{FULL_TAXONOMY}}` 对每个risk_type输出：

```text
risk_type: <唯一枚举>
所属业务域: <域名>
定义: <详细定义>
允许evidence: [<穷举枚举>]
选择条件: <正向条件>
排除条件: <与相邻类别的边界>
允许空evidence: true|false
```

并在46类之后追加跨类边界，例如 `gamble_ad` 与 `gamble_entry`、明确裸露与suggestive、仿冒与正常官方内容、纯登录框架与风险登录入口等。

### 3.3 紧凑渲染格式

`{{COMPACT_TAXONOMY}}` 必须保留完整46类，但每类只输出一行：

```text
<risk_type> | <简短定义> | allowed_evidence=[...] | empty=true|false
```

紧凑不等于省略类别或使用开放枚举。若渲染后不是46行，构建失败。

## 4. 数据集制作Prompt

### 4.1 System Prompt正文

以下代码块是数据集制作system模板正文：

```text
你是多模态内容审核数据集的候选标注生成器。你将看到一个静态图片输入，或由按播放时间顺序排列的最多4帧组成的GIF video输入，以及该媒体已有的human_label。你的任务是仅根据可观察内容生成一个待后续校验和确认的四字段候选JSON。

human_label的作用仅限于：当画面同时真实支持多个risk_type时，帮助选择唯一主业务域。human_label不是视觉事实，不能据此编造画面中不存在的文字、物体、行为或evidence。

如果human_label对应业务域没有足够的可观察依据，不得强行制造一致；应输出画面实际最支持的risk_type和完整evidence，后续流程会处理标签冲突。如果画面既没有风险信号，也没有已枚举的弱风险信号，使用good_clean与no_risk_evidence。

严格遵守以下规则：
1. 只能依据所附媒体。不要猜测链接跳转后、二维码扫描后、被遮挡区域后或未展示页面中的内容。
2. GIF帧按时间顺序理解；不得声称看到未提供的帧。
3. description只描述与审核有关的客观可见事实，保持简短；不得复述human_label，不得写规则路径或最终业务处理结果。
4. evidence必须是taxonomy封闭枚举中的字符串数组。保留画面中全部真实可观察信号，包括跨业务域信号；不得为了匹配human_label而删除、替换或虚构evidence。
5. risk_type必须是46类封闭枚举中的一个字符串，只能输出一个主类型。多风险同时存在时，在视觉证据支持的前提下优先选择human_label对应域的主类型；否则选择证据最充分的类型。
6. cot必须是包含1至3个非空短字符串的数组。先说明可观察事实如何支持输出数组中的每项evidence，再说明这些evidence如何支持同一个risk_type。
7. cot不得引入evidence数组之外的新evidence，不得出现与risk_type冲突的第二个最终类别。
8. 只有gamble_other、porn_other、politic_other、fraud_other允许evidence为空数组。使用这些类型时，cot必须说明可见的风险域依据以及不能落入更具体小类的原因。
9. good_clean必须输出evidence=["no_risk_evidence"]。good_risk_signal只能在存在已枚举弱信号、但根据边界不足以构成其他具体类型时使用。
10. 输出必须是单个合法JSON对象，只能包含description、evidence、risk_type、cot四个键，并严格按此顺序。不要输出原生thinking、`<think>`标签、Markdown、代码围栏、解释、前后缀或其他键。
11. evidence数组内部顺序不承载业务语义；无需为了顺序改写内容。不要主动复制相同evidence凑数。

输出结构：
{"description":"string","evidence":["closed_enum"],"risk_type":"closed_enum","cot":["step 1","step 2"]}

以下是当前版本的完整taxonomy与边界规则：
{{FULL_TAXONOMY}}
```

### 4.2 User Prompt正文

以下文本对每条样本渲染一次，并与媒体一起发送：

```text
human_label={{HUMAN_LABEL}}
请审核所附媒体并生成候选四字段JSON。只输出JSON对象。
```

静态图作为单个 image 输入，动态分辨率上限为 `max_pixels=1048576`（1024视觉token）；GIF使用统一媒体管道生成的最多4帧，按时间顺序组成一个 video 帧列表，每帧上限为 `max_pixels=262144`（256视觉token）。不得把GIF帧表示成多张独立 image。数据制作模型看到的媒体必须与Actor训练、评估和部署一致。

### 4.3 候选输出示例

```json
{"description":"页面展示六合彩号码、生肖图和投注引导文字。","evidence":["online_lottery_activity"],"risk_type":"gamble_lottery","cot":["画面中的报码、生肖和投注引导支持evidence=online_lottery_activity。","该evidence对应网络彩票投注场景，因此risk_type=gamble_lottery。"]}
```

模型输出使用生产四键；落库时由数据管道映射为：

```text
description -> candidate_description
evidence    -> candidate_evidence
risk_type   -> candidate_risk_type
cot         -> candidate_cot
```

候选结果不是训练真值。只有通过JSON、schema、封闭枚举、可见性、cot一致性以及锁定规则检查，并完成参考确认后，才生成对应 `ref_*`。

### 4.4 数据制作JSON Schema

结构化输出接口可使用以下schema骨架；两个enum必须由taxonomy版本注入：

```json
{
  "type": "object",
  "additionalProperties": false,
  "required": ["description", "evidence", "risk_type", "cot"],
  "properties": {
    "description": {"type": "string", "minLength": 1},
    "evidence": {
      "type": "array",
      "items": {"type": "string", "enum": {{EVIDENCE_ENUM_JSON}}
    },
    "risk_type": {
      "type": "string",
      "enum": {{RISK_TYPE_ENUM_46_JSON}}
    },
    "cot": {
      "type": "array",
      "minItems": 1,
      "maxItems": 3,
      "items": {"type": "string", "minLength": 1}
    }
  }
}
```

JSON Schema本身不能可靠表达顶层键的序列化顺序、条件空集和cot语义；这些仍由canonical parser和静态校验器检查。

### 4.5 自动重试Prompt

仅对JSON解析、键集合、字段类型、封闭枚举或cot机械一致性失败进行一次自动修复。修复时重新附上相同媒体、相同system和以下user文本：

```text
human_label={{HUMAN_LABEL}}
上一次候选输出未通过机械校验。
校验错误：{{VALIDATION_ERRORS}}
上一次输出：{{PREVIOUS_RESPONSE}}

请重新观察同一媒体，修正完整四字段JSON。不得仅根据错误信息猜测视觉事实，不得删除真实evidence以迎合human_label。只输出新的完整JSON对象。
```

以下情况不自动用Prompt“修正”：

- evidence从媒体不可见；
- risk_type视觉语义错误；
- 规则结果与 `human_label` 不一致；
- reference存在真实歧义。

这些情况进入拒绝、冲突或参考确认流程。不能通过反复提示把冲突样本诱导成预期标签。

## 5. Actor Prompt

### 5.1 System Prompt正文

以下代码块是SFT、GRPO、开发评估和部署共同使用的Actor system正文：

```text
你是多模态内容审核模型。请仅依据所附静态图片，或由按播放时间顺序排列的最多4帧组成的GIF video输入，生成一个结构化判定材料JSON。

严格遵守以下规则：
1. 只能依据可观察内容。不要猜测链接跳转后、二维码扫描后、被遮挡区域后或未展示页面中的内容。
2. GIF帧按时间顺序理解；不得声称看到未提供的帧。
3. description只描述与审核有关的客观可见事实，保持简短；不得写规则路径或最终处置结论。
4. evidence必须是下方taxonomy允许的封闭枚举字符串数组，并包含所有真实可观察的相关信号。
5. risk_type必须从下方46类封闭枚举中选择一个且只能选择一个，并与evidence一致。
6. cot必须是包含1至3个非空短字符串的数组：先确认可观察事实对输出数组中每项evidence的支持，再确认这些evidence对同一个risk_type的支持。
7. cot不得引入evidence数组之外的新evidence，不得给出第二个最终risk_type。
8. 只有gamble_other、porn_other、politic_other、fraud_other允许evidence=[]；使用时cot必须说明可见的风险域依据和退回other的原因。
9. 既没有风险信号、也没有已枚举弱风险信号时，使用risk_type=good_clean和evidence=["no_risk_evidence"]。存在已枚举弱信号但不足以构成其他具体类型时，按照taxonomy选择相应类型。
10. 输出必须是单个合法JSON对象，只能包含description、evidence、risk_type、cot四个键，并严格按此顺序。
11. 不要输出原生thinking、`<think>`标签、Markdown、代码围栏、解释、前后缀或其他键。项目字段cot只用于简短确认evidence与risk_type，不等于原生长推理。evidence数组内部顺序不承载业务语义。

输出结构：
{"description":"string","evidence":["closed_enum"],"risk_type":"closed_enum","cot":["step 1","step 2"]}

当前版本taxonomy：
{{COMPACT_TAXONOMY}}
```

该正文不得增加任何样本标签、参考值、候选值、规则引擎名称或结果、置信度、二元结果、样本标识和来源信息。禁止项检查见第8节。

### 5.2 User Prompt正文

所有样本使用完全相同的user文本：

```text
请审核所附媒体。仅依据可观察内容，按system中的固定schema输出一个JSON对象。
```

不为同一媒体制作多种问法，不追加类别暗示，不根据样本动态改变措辞。

### 5.3 Actor输出示例

```json
{"description":"页面展示六合彩号码、生肖图和投注引导文字。","evidence":["online_lottery_activity"],"risk_type":"gamble_lottery","cot":["画面中的报码、生肖和投注引导支持evidence=online_lottery_activity。","该evidence对应网络彩票投注场景，因此risk_type=gamble_lottery。"]}
```

## 6. SFT样本构造

### 6.1 Assistant Target模板

SFT target不由模型再次生成，而由已确认 `ref_*` 使用canonical serializer确定性构造：

```text
{"description":{{JSON_STRING(REF_DESCRIPTION)}},"evidence":{{REF_EVIDENCE_JSON}},"risk_type":{{JSON_STRING(REF_RISK_TYPE)}},"cot":{{REF_COT_JSON}}}
```

最终target必须是UTF-8单行JSON，无缩进、无Markdown、无前后文本。字段顺序固定为：

```text
description -> evidence -> risk_type -> cot
```

evidence数组保留参考revision中的顺序；首版不为数组排序或重复增加预处理。

### 6.2 MS-SWIFT概念样本

```json
{
  "messages": [
    {"role": "system", "content": "{{RENDERED_ACTOR_SYSTEM}}"},
    {"role": "user", "content": "请审核所附媒体。仅依据可观察内容，按system中的固定schema输出一个JSON对象。"},
    {"role": "assistant", "content": "{{CANONICAL_REF_JSON}}"}
  ],
  "images": ["/absolute/path/example.jpg"]
}
```

GIF按统一媒体管道转成一个有序 video 帧列表或框架等价的单个视频多模态字段，不能转成四个独立 image；Prompt文本不改变。

### 6.3 Loss范围

- system、user、视觉token、padding：不计算loss。
- assistant完整JSON及回合结束token：计算loss。
- 必须抽样解码非 `-100` labels，验证其精确等于完整assistant JSON及结束token。

## 7. GRPO、开发评估和部署

三个阶段都使用第5节完全相同的Actor system/user和taxonomy版本：

- GRPO输入没有参考assistant；隐藏列只供reward计算。
- 开发评估不向Actor暴露参考字段。
- 部署不追加规则结果或先验类别提示。
- rollout和确定性评估可以使用不同生成参数，但Prompt文本不能变化。
- 数据制作、GRPO、开发评估和部署均关闭模型原生thinking；若接口提供开关，显式设置 `enable_thinking=false`。
- taxonomy升级必须生成新的 `taxonomy_schema_version` 和 `actor_prompt_version`，旧模型不能静默切换到新Prompt。

## 8. 防泄漏与静态校验

### 8.1 Actor渲染结果禁用内容

发布前对最终渲染后的Actor system/user做字符串和结构扫描，禁止出现：

```text
human_label
candidate_
ref_
vlm_advice
vlm_confidence
r5 / R5
md5
source
expected_risk_type
target_risk_type
```

同时禁止包含：

- 10类业务真值；
- 二元判定标签或映射；
- 规则引擎内部路径、结果或置信度；
- 任何样本级期望答案。

`good_clean`、`good_risk_signal` 等是46类正式taxonomy成员，不属于泄漏。

### 8.2 Prompt构建测试

每个Prompt版本至少通过：

1. `FULL_TAXONOMY` 和 `COMPACT_TAXONOMY` 的risk_type集合完全相同且恰为46类。
2. 两种渲染的evidence总集合完全相同。
3. 每个risk_type的允许evidence映射一致。
4. Actor禁用字符串扫描通过。
5. 数据集制作user只注入合法10类 `human_label` 值。
6. Actor user文本对全数据集完全相同。
7. 四字段示例、JSON Schema和canonical serializer通过同一parser。
8. 空evidence仅允许四个 `*_other`。
9. 静态图和GIF帧包使用相同 `media_pipeline_version`；静态图1024视觉token、GIF 256视觉token/帧的上限通过processor实测，GIF序列化为单个video。
10. 保存最终渲染文本hash；训练配置引用hash，不读取滚动“latest”Prompt。

## 9. 两轮数据收集中的Prompt使用

已确认采用“两轮收集、自动盘点、只补缺口”：

1. 第一轮按10类 `human_label` 配额收集常规业务数据。
2. 使用第4节同一个数据集制作Prompt生成候选并完成参考确认。
3. 自动统计46类 `ref_risk_type`、good混淆方向和有效样本数量。
4. 第二轮在数据管道外选择可能补足缺口的媒体，但仍使用同一个数据集制作Prompt。
5. 第二轮不得把期望补齐的 `risk_type`、evidence或采集方向写进Prompt；否则会诱导候选模型确认预期答案。
6. 非-good困难正例不设专项硬配额，自然遇到则保留；不为寻找困难样本创建第三套Prompt。
7. 达到10,000条门禁后有效样本并完成既定配额后停止；原始抓取数量不写死。

## 10. 版本与变更规则

以下任一变化需要升级对应Prompt版本并重跑静态测试：

- system或user正文改变；
- taxonomy定义、枚举、映射或空集规则改变；
- JSON字段、类型、顺序或cot规则改变；
- 媒体输入语义改变；
- 数据制作重试规则改变。

仅模型生成参数改变不等于Prompt文本改变，但必须记录新的 `candidate_generation_config`。模型snapshot改变必须记录新的 `candidate_model_revision`。

Prompt版本、taxonomy版本、最终渲染hash、模型revision和媒体管道版本共同决定一批候选是否可复现。

## 11. 实施前仍需补齐的机器可执行资产

本文已经给出Prompt正文和渲染契约。正式运行前还必须完成：

1. 将 `risk_type.md` 转成结构化、可校验的taxonomy文件。
2. 展开完整封闭evidence枚举，消除“等”和“任一弱信号”。
3. 实现 `FULL_TAXONOMY` / `COMPACT_TAXONOMY` 单源渲染器。
4. 实现canonical JSON serializer和条件空集校验器。
5. 冻结首个 `dataset_prompt_version`、`actor_prompt_version` 和 `taxonomy_schema_version` 标识。

在这五项完成前，本文可用于评审和pilot，但不应把手工复制的taxonomy文本当成最终生产Prompt。
