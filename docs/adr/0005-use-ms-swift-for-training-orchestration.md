# 使用 MS-SWIFT 统一编排 SFT 与 GRPO

本项目选用 MS-SWIFT 的标准 Transformers 后端统一编排 Qwen3-VL-8B-Instruct 的多模态 SFT 与 GRPO，不采用 Megatron-SWIFT，也不以手工拼装的 PEFT/TRL 作为默认工程框架。MS-SWIFT 对 Qwen3-VL、多模态数据、LoRA adapter 接续、KL、自定义 reward 和分布式训练提供了更完整的一体化接口，可以减少两阶段之间的模板、processor、adapter 与 reference 语义漂移。底层仍可使用 Transformers、PEFT、TRL/DeepSpeed；若 MS-SWIFT 在正式 smoke test 中出现无法绕过的阻断性问题，再切换到原生 Hugging Face `Transformers + TRL + PEFT + Accelerate/DeepSpeed` 备用栈，不并行维护两套正式实现。完整证据与硬件约束见 `训练框架调研.md`。
