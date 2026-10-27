# 每日学习主题索引

## 记录规则

- 本文件记录每天涉及的内容、详细教材入口与学习状态；详细讲解放在 `YYYY-MM-DD_序号_主题名.md` 中。
- 序号每天从 `01` 开始，按已有学习顺序排列；文件名使用简洁英文主题标识，文档标题使用中文。同日多个主题分开记录，相关主题可合并为一个专题。
- 回补缺失章节可用旧日期前缀与字母插入编号保持课程相邻；正文标记实际补写日期，在旧日表中注明回补，并在实际补写日记录，不把回补当作旧日已经学习。
- 新学习日新增日期条目与专题文件；答疑优先补到对应专题。明确要求不入文档的问题仍不补录。
- 先整体后局部，使用 LaTeX、形状表与图示；自测可选，不将“材料已提供”当作“已掌握”。
- 学习优先级遵循 [准备索引](../index-2026-10-08.md)：应用层与算法 / 数据优先，重要 Infra 提供概要与资料索引。
- 历史记录中的“下一步”表示当时计划，当前安排以各日期条目的最新状态为准。

## 2026-10-08：Transformer 与 Attention 基础

| 主题 | 涉及内容 | 详细教材 |
|---|---|---|
| Transformer 总体结构 | Block、残差 / Norm / FFN、Encoder / Decoder、causal 与生成流程 | [Transformer 总体结构](2026-10-08_01_transformer_overview.md) |
| 位置表示 | 同维相加与拼接区别、抵消与梯度、RoPE 概览 | [位置表示](2026-10-08_02_position_encoding.md) |
| Softmax 与损失 | 注意力权重与预测概率、交叉熵、其他损失与训练阶段 | [Softmax 与训练损失](2026-10-08_03_softmax_and_losses.md) |
| Attention / MHA | Q/K/V、缩放方差推导、mask、拆头、投影参数量、FFN 与固定宽度 | [Attention / MHA](2026-10-08_04_attention_mha.md) |

状态：已开展阅读与多轮答疑，未进行统一掌握测验。推荐按上表顺序复习。

<details>
<summary>当日目标、学习方式与反馈（保留历史记录）</summary>

### 1. 学习位置与状态

- 学习索引：[工作能力提升与实践手册](../index-2026-10-08.md)。
- 当前阶段：Week 1 补底子，Transformer 基础。
- 今日主题：先建立 Transformer 总体图景，再理解 Scaled Dot-Product Attention 与多头注意力（MHA）。
- 状态：已提供自学讲义；阅读进度与掌握情况待用户反馈。
- 后续主题：理解本节后学习 MQA / GQA 与 KV Cache，然后进入 RoPE、RMSNorm、SwiGLU；自测为可选项，不作为继续学习的硬性条件。

记录原则：每天按日期保存主题、核心知识、练习反馈、疑问与下一步；不把“读过”当成“掌握”，也不预先填写学习时长或测验成绩。

学习方式：先阅读当天 Markdown 讲义，遇到不懂的概念、公式或例子时在对话中提问。答疑后将相关解释与反馈补充到当天记录，不主动进行问答式测验。后续学习日沿用这种方式，在实际开展学习时创建当天记录，不预填未来日期的学习成果。

建议阅读顺序：按上方同日专题列表阅读；可选自测不要求提交答案。

阅读提示：公式使用 LaTeX 排版。在 VS Code 中按 `Ctrl+Shift+V` 打开 Markdown 预览；源文件中会显示公式标记，预览中才会显示分数、根号、上下标与矩阵。

图示说明：下文三张图来自 Aston Zhang、Zachary C. Lipton、Mu Li、Alexander J. Smola 编写的《Dive into Deep Learning》，按 [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/) 许可使用（[项目许可](https://github.com/d2l-ai/d2l-en/blob/master/LICENSE)）。图像未修改，保存于本地 `images` 目录，可离线预览；各图附来源与中文读图提示。

### 2. 今日目标

完成本节后，应能：

0. 画出 Transformer 的总体流程，说明 embedding、位置表示、注意力、前馈网络、残差连接与归一化的职责。
1. 解释 Q、K、V 的职责，而不只是背公式。
2. 写出 Attention 公式，标注张量形状与 softmax 方向。
3. 解释为什么除以 $\sqrt{d_k}$，以及 causal mask 的作用。
4. 解释 MHA 如何拆分、拼接不同的头。

### 9. 学习反馈

- 学习方式反馈：用户选择先阅读 learning_records 中的 Markdown 讲义，遇到不理解的内容时主动提问。
- 排版偏好：数学表达使用 LaTeX 行内或独立公式，张量形状用表格展示，不将公式放进代码块；后续讲义沿用。
- 内容顺序偏好：先讲总体架构与模块职责，再展开 MHA 等局部机制；后续主题先补齐必要前置知识。
- 前置材料补充：Transformer 总体流程、block 结构、Encoder / Decoder 架构区别、位置表示、LM Head，以及训练与生成的区别。
- 位置表示答疑：加法式位置编码是同维度逐元素相加，不是拼接；补充数字示例，并区分现代 LLM 中常见的 RoPE 与输入端的位置向量相加。
- 位置相加答疑：局部坐标可以抵消，加法不保证两个任意向量都可恢复；加法对两条输入支路的导数为单位矩阵，前向结果为零不意味着梯度为零。掌握情况仍待阅读反馈。
- Causal 答疑：指当前位置不能读取未来位置，不是因果推断；补充可见范围、当前位置输入与下一 token 目标的区别，以及训练并行与生成逐步进行的区别。
- Causal mask 阶段答疑：区分训练、推理 prefill 与 decode；单 token decode 在仅暴露有效历史缓存时无需额外遮住不存在的未来 token，并区分因果限制、padding / 缓存有效性与滑动窗口。
- 生成与读取答疑：causal mask 限制 Attention 读取未来输入位置，不限制 LM Head 对词表候选打分；下一个 token 由最后一个已知位置的隐藏状态预测，不需先建立该未来 token 的输入表示。
- Softmax 答疑：说明非负归一化、分数差与份额比、软选择和可微性；区分注意力权重与输出概率，并补充交叉熵梯度、温度与数值稳定性。Softmax 不是唯一选择，也不保证概率校准。
- 损失函数答疑：概览回归、分类、分布匹配、对比与排序损失；说明 Decoder-only 预训练 / SFT 主要使用 token 交叉熵，并区分损失掩码、causal mask 与偏好 / 强化学习目标。
- 本轮材料：补充单头注意力逐步计算例子、拆头与输出投影解释、self/cross-attention 区别、Transformer 层中 MHA 的位置，以及可折叠自测参考答案。
- 用户自测回答：待补充。
- 已确认掌握：暂无，等待用户阅读反馈或答疑中的实际表现。
- 待澄清问题：等待学习过程中提出。
- 下一步：用户自行阅读本讲义；有疑问时补充答疑，准备好后再进入 MQA / GQA 与 KV Cache。

</details>

## 2026-10-09：张量运算与 Transformer 核心模块

| 主题 | 涉及内容 | 详细教材 |
|---|---|---|
| 张量形状与矩阵乘法 | B / h / n / 特征轴、行列向量、reshape 与换轴、QK 转置与 AV | [张量形状与矩阵运算](2026-10-09_01_tensor_shapes.md) |
| KV Cache / MQA / GQA | LM Head 形状、历史表示与缓存复用、按层 KV、头共享、基础显存与参数估算、GQA 效果实验 | [KV Cache / MQA / GQA](2026-10-09_02_kv_cache_mqa_gqa.md) |
| RoPE | 二维旋转、Q/K 点积的相对位移、多频率、与 KV Cache / GQA 的关系、长上下文边界 | [RoPE 旋转位置编码](2026-10-09_03_rope.md) |
| RMSNorm | Norm 在 block 中的位置、逐 token 特征归一化、均方根算例、LayerNorm 对比、训练与推理边界 | [RMSNorm 与 LayerNorm](2026-10-09_04_rmsnorm.md) |
| SwiGLU | FFN 非线性、SiLU、门控双分支、逐元素乘法、形状与参数预算 | [SwiGLU 与门控 FFN](2026-10-09_05_swiglu.md) |
| MoE（2026-10-19 回补） | token 路由、专家合并、总 / 激活参数、训练平衡与计算 / 显存边界 | [MoE 架构](2026-10-09_06_moe_architecture.md) |

状态：已进行张量形状、KV / GQA、RoPE 与 Norm 答疑；用户确认本轮 KV Cache、RoPE 与 Norm 学习告一段落，进入 SwiGLU。已提供 SwiGLU 讲义，阅读反馈待补充；不等同于所有知识点已验证掌握。

后续：阅读 SwiGLU 并按疑问答疑，之后优先进入微调与训练数据；按需要复习前置知识。Infra 不扩展为底层调度与部署专项。

<details>
<summary>当日学习安排、资料与反馈（保留历史记录）</summary>

### 1. 学习位置与方式

- 学习索引：[工作能力提升与实践手册](../index-2026-10-08.md)。
- 前置讲义：[2026-10-08：Transformer 总体结构与 Attention / MHA](2026-10-08_01_transformer_overview.md)。
- 当前阶段：Week 1，Transformer 基础与自回归推理。
- 今日主题：先补齐张量形状与矩阵乘法，再理解 KV Cache 为什么能复用历史，以及 MQA / GQA 为什么减少 KV 头。
- 状态：已提供讲义，阅读进度与掌握情况待反馈；不预填学习时长或成绩。
- 学习方式：先阅读，遇到不懂的地方提问；自测可选，不主动测验。

建议先读张量形状专题，再读 KV Cache / MQA / GQA 专题。

公式请在 VS Code Markdown 预览中阅读（`Ctrl+Shift+V`）。本讲义的头共享图是自绘示意，不是某个具体模型的架构截图。

### 10. 资料与学习反馈

资料：

- [Transformers：How caching works](https://huggingface.co/docs/transformers/cache_explanation)，用于理解按层缓存与 decode 更新。
- [GQA 论文](https://arxiv.org/abs/2305.13245)，研究从 MHA checkpoint 训练适配到 GQA。
- [MQA 论文：Fast Transformer Decoding: One Write-Head is All You Need](https://arxiv.org/abs/1911.02150)。

学习记录：

- 已提供：KV Cache 生成流程、头共享图示、形状表、显存算例、投影参数推导。
- 用户阅读反馈与疑问：待补充。
- 前置疑问：用户希望先熟悉 B、h、token 与特征轴的含义，以及行列、拆头和矩阵乘法关系；已补充 [张量形状专题](2026-10-09_01_tensor_shapes.md)与可选练习。
- 已验证掌握：暂无，不将昨天的讨论或今天的讲义自动视为掌握。
- 下一步：根据本节疑问答疑；准备好后学习 RoPE 的旋转与相对位置信息，再进入 RMSNorm / SwiGLU。

</details>

## 2026-10-10：SwiGLU 复习、微调与数据配方

| 主题 | 涉及内容 | 详细教材 |
|---|---|---|
| SwiGLU 复习 | SiLU 函数图，普通 FFN 与 SwiGLU 的激活位置；本轮学习结束 | [已有 SwiGLU 专题](2026-10-09_05_swiglu.md) |
| SFT / 训练数据 | 监督微调目标、对话格式、teacher forcing、右移与损失范围、数据质量、公开数据集与开源工具选型、泄露与评测 | [SFT 与训练数据](2026-10-10_01_sft_and_training_data.md) |
| SFT 数据图鉴（2026-10-24 回补） | 原课 12 个数据集的 domain、原始字段、Input / Output、对话树与工具轨迹；官方计数与自拟示例分开 | [01a 数据图鉴](2026-10-10_01a_sft_and_training_data.md) |
| LoRA / QLoRA | 低秩更新、参数与形状、冻结及梯度、目标模块、合并、量化基础权重与效果边界 | [LoRA / QLoRA](2026-10-10_02_lora_qlora.md) |
| 数据分布与混合采样 | 线上 / 训练 / 评测分布、样本与监督 token 占比、配方、重复曝光、采样与消融 | [数据分布与混合采样](2026-10-10_03_data_distribution_and_sampling.md) |
| 遗忘诊断与缓解 | 能力退化与行为变化、量化对照、双评测、回放、checkpoint、KL / 蒸馏、参数约束与任务隔离 | [SFT 后的遗忘](2026-10-10_04_forgetting_diagnosis_and_mitigation.md) |
| 有依据问答 SFT 实验设计 | 用户选择领域问答；固定资料、引用与不足边界、文档划分、基线与回放对照、独立评测 | [有依据问答实验设计](2026-10-10_05_grounded_qa_sft_experiment.md) |
| 偏好数据与奖励模型 | chosen / rejected、上下文与标注规则、标量 reward、排序损失、代理风险与独立评测 | [偏好数据与 Reward Model](2026-10-10_06_preference_data_and_reward_model.md) |
| 强化学习基础 | 直接用 LLM 生成解释策略、状态、动作、rollout、回报、Critic 与优势、离散动作的策略梯度 | [LLM 强化学习入门](2026-10-10_07_rl_basics_for_llm.md) |
| PPO | rollout、策略 / RM / Critic、优势、old / ref 区别、概率比与 clip | [PPO 总览](2026-10-10_08_ppo.md) |
| DPO | 生成策略与参考策略、回答 log 概率、偏好损失、数值例子、β、SFT / PPO 对比与 GRPO 预告 | [直接偏好优化](2026-10-10_09_dpo.md) |

状态：用户选择 Hugging Face 的 Transformers、PEFT、TRL 官方技术文档作为有依据问答实验资料；已核对首轮三页的许可与固定版本，提供四条候选问答。尚未准备完整训练数据、安装工具、训练或产出评测结果。

硬件：用户确认本地 RTX 4080 Laptop GPU、12GB；首轮建议 Qwen3-0.6B + LoRA，尚未检查环境或实测配置。

后续：确认训练运行方式与依赖环境，再落地资料片段、人工审核与独立数据划分；理论后续为偏好数据 / Reward Model → DPO。不将候选样例或配置建议记为已运行实验。

实施进度：用户批准开始；GPU 已实测，已创建 [独立实验目录](../experiments/grounded_qa/README.md)、固定来源与 8 条训练 / 3 条诊断候选，6 项数据校验测试通过。独立训练环境正在安装，尚无 GPU 训练结果；诊断候选不是独立泛化测试集。

并行理论进度：已讨论 Bradley–Terry 偏好关系、可微损失与奖励模型规模；已提供 DPO 教材，下一理论专题为 PPO / GRPO。未运行奖励模型或 DPO 训练。

顺序修正：用户指出缺少 PPO 及强化学习前置知识，已补充教材并将文件顺序统一为第 07 专题强化学习基础 → 第 08 专题 PPO → 第 09 专题 DPO，随后再学 GRPO。先暂停 PPO 公式，从 LLM 生成例子补基础；不改变学习日期或掌握状态，未执行 PPO 训练。

## 2026-10-11：后训练与应用生成

| 主题 | 涉及内容 | 详细教材 |
|---|---|---|
| GRPO | 同题多回答、组内奖励与优势、无需独立 Critic、PPO 裁剪、KL 与实现差异、零方差和奖励风险 | [组相对策略优化](2026-10-11_01_grpo.md) |
| 蒸馏与合成数据 | 教师文本 SFT、软分布 / 温度 / KL、学生 on-policy 指导、合成覆盖与验证、独立评测 | [蒸馏与合成数据](2026-10-11_02_distillation_and_synthetic_data.md) |
| 生成参数与结构化输出 | 贪心 / 采样、temperature / top-p / top-k、停止与预算、JSON / schema 约束、业务校验与工具调用边界 | [生成与结构化输出](2026-10-11_03_generation_and_structured_output.md) |
| Prompt / 上下文设计 | 任务与证据分离、few-shot、上下文预算、历史与状态、提示注入边界、版本和对照实验 | [Prompt 与上下文](2026-10-11_04_prompt_and_context_design.md) |
| 应用评测 | 组件 / 端到端范围、黄金集与留出数据、规则 / 人工 / LLM judge、指标分母、配对对照、灰度与坏例闭环 | [应用评测体系](2026-10-11_05_application_evaluation.md) |

状态：已讨论 DPO / 在线 RL、数学与长程 Agent 训练，以及教师在学生前缀上的具体指导方式；现提供生成参数与结构化输出讲义，阅读与掌握情况待反馈，未执行 RL、蒸馏或生成对照实验。

实验线程：SFT 权重下载在官方与镜像路径均遇到中断，目前暂停；没有完整权重哈希验证或 GPU 训练结果。本次仅继续理论，不自动重新下载。

后续：阅读应用评测并答疑，随后学习 RAG 总体链路与适用边界，再深入解析、chunking 和检索。已讨论生成参数的 API 支持、beam search、重复惩罚与 stop；未执行 Prompt 或应用评测对照。

## 2026-10-12：RAG、文档处理与混合检索

| 主题 | 涉及内容 | 详细教材 |
|---|---|---|
| RAG 总览 | 离线建库、在线检索与生成、多证据引用、RAG / SFT / 长上下文 / 工具边界、分层评测与可靠性 | [RAG 总体链路](2026-10-12_01_rag_overview.md) |
| 文档解析与 Chunking | 结构提取、切块策略、overlap、父子文档、来源与版本、长度预算、切块评测与重排数据衔接 | [解析与切块](2026-10-12_02_document_parsing_and_chunking.md) |
| Embedding / 向量检索 | 文本编码、query / document 协议、余弦 / 点积 / 距离、矩阵形状、对比训练、ANN 概要与重排分工 | [Embedding 与向量检索](2026-10-12_03_embeddings_and_vector_retrieval.md) |
| BM25 / 混合检索 | 倒排与词项化、词频饱和 / IDF / 长度校正、分数尺度、RRF 融合与预算对照 | [BM25 与混合检索](2026-10-12_04_bm25_and_hybrid_retrieval.md) |
| Re-rank 数据与目标 | Cross-encoder、相关性 / 支持标签、实际召回候选、hard negatives、pointwise / pairwise / listwise、排序与证据评测 | [Re-rank 训练](2026-10-12_05_reranker_data_and_objectives.md) |
| 查询优化 / 上下文压缩 | rewrite、multi-query、问题分解、HyDE、证据筛选 / 展开、压缩与引用溯源、预算对照 | [查询与上下文优化](2026-10-12_06_query_optimization_and_context_compression.md) |
| 完整 RAG 实验设计 | 数据与知识库范围、组件消融、预算匹配、日志、召回 / 重排 / 压缩 / 生成失败归因、端到端评测 | [RAG 实验与归因](2026-10-12_07_rag_experiment_and_failure_analysis.md) |

状态：接续应用评测，已提供 RAG 讲义和流程图；阅读与掌握情况待反馈，未搭建或运行 RAG。

实验线程：此前 SFT 下载暂停，本次不自动恢复；没有完整权重验证或训练结果。

后续：阅读完整 RAG 实验设计并答疑，再进入工具调用与 Agent 基础。已讨论整组人工排序、Agent 反馈与 CTR 监督；本篇仅实验设计，未运行检索、评分或排序训练，未验证掌握。

## 2026-10-15：Agent 基础与规划、真实 Harness 阅读计划

| 主题 | 涉及内容 | 详细教材 |
|---|---|---|
| Agent 总体架构 | chatbot / RAG / workflow / Agent、模型与 runtime、工具 / 环境反馈、任务状态、完成验证、预算与最小评测 | [Agent 架构与适用边界](2026-10-15_01_agent_architecture_and_boundaries.md) |
| Function Calling | 双工具契约、schema、call ID、结果回填、结构 / 授权 / 完成校验、显式错误、并行依赖与幂等边界 | [工具调用与契约](2026-10-15_02_function_calling_and_tool_contracts.md) |
| 最小 Agent loop | 多轮响应处理、ReAct、工具回填、澄清与终止、模型 / 工具 / 时间预算、无进展检测与模拟测试 | [最小循环与控制流](2026-10-15_03_minimal_agent_loop.md) |
| 状态 / 持久化 / 恢复 | 协议与业务状态、checkpoint、操作记录、结果未知窗口、幂等与授权、取消 / 回滚 / 并发、故障测试 | [状态与恢复](2026-10-15_04_state_persistence_and_recovery.md) |
| 任务分解 / 规划 / 重规划 | 办公任务、计划与执行依赖 / 验收区别、条件变更与失效结果；源码细节已移出 | [任务规划与重规划](2026-10-15_05_task_planning_and_replanning.md) |
| Agentic retrieval / 上下文工程 | 可访问 / 已保存 / 本轮可见分层、统计、有界读取、落盘 / 摘要、按需加载；源码细节已移出 | [主动检索与上下文工程](2026-10-15_06_agentic_retrieval_and_context_engineering.md) |
| Skills（2026-10-19 回补，6a） | 方法包、目录 / 正文 / 资源分层、加载与执行、历史保留、触发与权限评测 | [Skills 与渐进式披露](2026-10-15_06a_skills_and_progressive_disclosure.md) |
| Memory | 任务状态与跨任务记忆、写入范围 / 来源、检索、纠错、过期 / 删除、用户隔离与评测 | [Agent Memory](2026-10-15_07_agent_memory.md) |
| 长程 Coding / Computer-use | 需求与验收、代码定位 / 修改 / 测试 / 修复、独立预期、界面观察与过期、执行核查、风险和成本 | [长程任务与验证](2026-10-15_08_long_horizon_coding_and_computer_use.md) |
| DeepSeek Harness 项目全貌 | 固定版本项目定位、装配 / 执行两条线、目录、profile / bundle / plugin、Cordis、session / step / turn 与安全边界 | [项目入门与核心概念](2026-10-15_09_deepseek_harness_overview.md) |
| Harness 架构阅读计划 | A＋B 后重点读 Deep Agents、DeepSeek Harness，按需对照 OpenCode；概念到源码 / 轨迹 / 测试映射，核对项目身份与版本 | [学习索引：A＋B 桥接实践](../index-2026-10-08.md) |
| 办公 / 个人 Agent 产品阅读 | 确认 Meta Muse 身份，WorkBuddy / Muse 官方与第三方资料分级；新增 Rowboat / OpenClaw 开源产品学习方向，固定版本与应用验收 | [产品阅读与开源替代报告](../agent_product_reading_workbuddy_muse_2026-10-15.md) |

状态：Agent 阶段 A 的架构、工具契约、loop 与状态恢复教材和图示已提供；阅读与掌握情况待反馈，未实现 Agent、持久化或故障测试。

后续：A＋B 概念教材之后，已开始独立的 DeepSeek Harness 项目课，先提供全貌与术语，下一篇再追一次完整调用链，不同时引入多套项目。阅读与掌握情况待反馈；未克隆、安装或运行项目，模拟器和故障测试仍待实践。

学习方式反馈：用户指出项目映射缺少项目全貌、难以理解；已改写规划第 5 节和上下文第 8 节，将源码内容迁至 [后续项目阅读笔记](../harness_implementation_notes_2026-10-15.md)。当前无需阅读该笔记；后续项目课从总体架构、目录与核心术语开始。

桥接增补：候选扩展到办公与个人产品方向，整体仍选 2 个精读 + 1 个对照，不全量部署五个项目。独立严谨产品横评尚有证据缺口，报告未将官方案例或带推广的教程当成性能证明。

实验线程：此前 SFT 权重下载保持暂停，本次不恢复、不安装框架或启动训练；没有新增实验结果。

## 2026-10-16：DeepSeek Harness 执行主链、失败与取消

| 主题 | 涉及内容 | 详细教材 |
|---|---|---|
| 只读任务的静态源码主链 | 内部 messages 格式与两次请求快照；headless 入口、Agent 创建、inbox、请求组装、流式响应、工具执行、续接和交付；第 3～9 节各有设计问题与局部时序图 | [一次读文件任务的完整主链](2026-10-16_01_deepseek_harness_task_lifecycle.md) |
| 失败、请求重试与取消 | 状态到动作的决策表；工具错误回填、同 step 不同 attempt、normal / always 与退避、失败片段与 interrupted 前缀、取消排空及副作用边界 | [失败、重试与取消](2026-10-16_02_deepseek_harness_failures_retries_and_cancellation.md) |
| 包与类交互地图 | 347 个包，TS / JS / Python 中的类、可识别函数式插件与接口声明；包依赖、继承 / 实现、构造、成员类型及声明的服务依赖；离线缩放、搜索、局部聚焦与源码跳转 | [DeepSeek Harness 交互图](../architecture_maps/deepseek_harness_explorer.html) |

状态：已读取固定提交的入口、工厂、默认 loop、工具调度、执行管线和 read 工具源码，并提供普通主链与时序图；后续已核对模型请求重试策略、工具错误与协作取消的主要路径。所有 provider、Web / SDK 传输及异常组合尚未完整追踪；没有实际运行轨迹或效果结论。

阅读反馈：原文缺少 messages 格式前置，逐步骤叙述难以抓住设计；已补内部 Message / content / source、工具调用关联和两次请求的数据变化，并为第 3～9 节逐一补充设计取舍与细粒度英文消息时序图。

架构图：固定提交选取的 3,223 个源码 / 包声明文件均核对 Git blob 哈希，AST 解析 2,876 个源码文件；图内提供覆盖范围、未解析引用与关系证据。接口按声明显示，同名增强声明不代表多个运行实例；未把动态事件协作或所有函数调用冒充成静态类关系。浏览交互图不需要安装 Harness 或连接模型。

后续（2026-10-17 调整）：改用下方分层架构索引，旧主链与失败教材作为按需查阅资料；不自动续写原来的逐流程路线。阅读与掌握情况待反馈。

实验线程：不安装或启动 Harness，不调用模型，不恢复暂停的 SFT 权重下载。

本地源码更新：按用户要求，已下载到 [projects/deepseek-harness](../projects/deepseek-harness)，检出教材固定提交 `571902028a75c64c1d767f4ae5fd099485f0ddf0`。保留 Git 信息及该提交完整源码，历史为浅历史；通过代理分批补齐对象并校验工作区与 Git 连通性。此次仅下载，未安装依赖、构建或运行；前文“未克隆 / 下载”的说明属于当时状态。

## 2026-10-17：从源码细节转向架构设计与实施验收

| 主题 | 涉及内容 | 详细教材 |
|---|---|---|
| 分层架构索引 | 系统全貌 → 模块职责 → 可展开主流程与源码 / 测试入口；设计选择、代价及阅读停止条件 | [架构学习与源码索引](2026-10-17_01_deepseek_harness_architecture_index.md) |
| 架构师指导 Coding Agent | 职责分工、最小目标、接口与状态规则、分批交付、任务卡及证据审查 | [Coding Agent 实施方法](2026-10-17_02_harness_architect_to_coding_agent.md) |
| 功能与系统设计验收 | 8 项功能、16 项设计要求；刺激、通过条件、证据、投递决策表及放行标准 | [验收清单](2026-10-17_03_harness_acceptance_checklist.md) |
| 官方文档层级与最小 MVP 路径 | 文档分类、阅读依赖和重要性；7 站选读 12 篇的指定章节，明确跳过范围及追加阅读触发条件 | [docs 层级与 MVP 路径](2026-10-17_04_deepseek_harness_docs_mvp_reading_path.md) |
| DeepSeek Harness 七站专题教材 | 总入口与 7 篇可直接阅读的教材；重组 12 篇原文，提炼概念、子流程、设计问题、原文理由与分析、例子及验收映射 | [专题总入口](deepseek_harness/README.md) |

学习方式：精炼、清楚、抓重点；以设计问题组织材料，不按函数逐段讲解。每个主题连接项目证据、最小实现思路和验收项；已有长篇教材与包图保留为下钻资料。先自行阅读，有疑问再问，不主动测验。

同日补充：架构索引新增 X1～X8 跨模块设计主题，展开异常处理与预算控制；M0～M6 均改为“子流程 → 处理内容 / 关键设计点”的分层 bullet。验收新增总预算、过载、排障与数据保护、格式兼容、故障归属，并同步映射到实施批次。

证据状态：本轮核对本地架构 / AgentLoop / 测试文档，抽查请求历史与资源生命周期测试源码；未执行测试。自建目标与验收标准均属拟定要求，不代表上游已实现或通过。

新增证据：核对工具超时实现及重复调用提醒、压缩、持久化、遥测文档。工具超时是协作取消，重复提醒是建议；这些不证明已有全任务预算控制器。完整总预算实现仍待核对，未运行测试。

官方文档盘点：固定版本 docs 下有 473 个 Markdown 文件，其中 234 个为中文对侧、239 个为非 `.zh.md` 文件；已核对目录、主要入口和选读章节，未通读全部文档。学习路径区分概念、流程、子系统参考、实现 / 测试、生成目录及维护记录。

专题化更新：7 站教材已放入 [deepseek_harness](deepseek_harness/README.md)，作为默认直接阅读材料；原始文档和既有索引保留作核对与下钻。第 3、6 站标出 API 文档部分重试描述与固定版本 Driver 不一致，采用已核对的同 step 重试行为。例子为教学自拟，不是运行轨迹。

写作结构：按最新反馈，七站统一为“是什么 → 为什么要做 → 怎么做 → 为什么这么做”，分别说明定义、需求、流程与方案取舍，再连接 MVP / 验收；小机制沿用相同分析维度，原文理由与讲解者分析明确区分。

下一步：顺序阅读专题七站；已理解的内容可跳过，遇到精确语义再回原文和源码。材料整理完成不等于用户已掌握或系统已通过验收。不修改上游源码，不自动安装依赖、启动 Harness、调用模型或恢复 SFT。

## 2026-10-18：Agent 生命周期与 Multi-Agent 协作

- 更新[专题第 1 站](deepseek_harness/2026-10-17_01_system_and_plugins.md)：区分 Agent 接口、AgentLoop 服务、ReactLoopAgent 实例、Driver 异步活动；补充多实例创建、活动先后启动、同实例状态协作与作用域。
- 同步修正七站术语：Driver 不是独立类实例，但也不能直接等同于 Agent 对象；活动、实例、turn 和取消信号有不同生命周期。
- 静态核对 factory、会话 / 子 Agent 创建入口及 wakeDriver、kick、turn、whenIdle、maintenance 分支。未运行 Harness；此前明确不更新文档的其他讨论不补录。
- 新增[扩展章 8：Multi-Agent](deepseek_harness/2026-10-18_08_multi_agent.md)：整理本次多会话与多角色答疑，区分 server 装配、headless 任务启动和显式预创建；补充 Team 配置、本地父子共用机制、Activation / Session 管理与场景化验收。
- 已接入专题与主索引，七站最小路径保持不变。Team 场景补丁、创建测试和自拟示例明确区分；仅完成学习材料与静态核对，不代表已执行测试或掌握全部内容。

## 2026-10-19：结束 Harness 第一遍学习，恢复 Agent 主干并开始实践

- **缺失章节回补**：新增 [MoE](2026-10-09_06_moe_architecture.md)、[Skills 与渐进式披露](2026-10-15_06a_skills_and_progressive_disclosure.md)、[Harness Skills 扩展 2a](deepseek_harness/2026-10-17_02a_skills_and_progressive_disclosure.md)。分别邻接历史基础课、上下文课和 Harness 第 2 站；文件前缀用于课程排序，实际编写为今天，尚待用户阅读。
- **Skills 讲解重写**：用户反馈流程难懂，已将 [Harness 扩展 2a](deepseek_harness/2026-10-17_02a_skills_and_progressive_disclosure.md)改为单一任务贯穿的“扫描发现 → messages 目录与 tools 定义 → LLM 请求加载 → 下一请求取得正文 → 普通工具执行”，补入请求 / 响应示例，缓存与作用域细节后置；未运行示例。
- **MoE 讲解重写**：按用户要求，将 [MoE 基础课](2026-10-09_06_moe_architecture.md)改为“普通 FFN 对照 → 单 token 的 4 选 2 算例 → 参数 / 计算 / 存储 → 联合训练与负载平衡”，明确专家接收完整向量、并行分支与加权合并；补充基线和权重存储算例，变体细节后置。未训练或部署模型。
- **多 Agent 证据补充**：在[第 12 课](2026-10-19_04_multi_agent_and_delegation.md)补入 Claude Research 生产案例及受控 benchmark 对照，区分厂商内部相对收益、费用口径与可推广边界；不称为跨任务公认最佳实践。
- **用户确认**：DeepSeek Harness 第一遍学习已走完。当天答疑涉及 Driver / turn / step、Inbox 接纳与请求 payload、异步等待、signal、scope、dispose、phase 与 activityDone；不等同于实践或全面掌握。
- **第 9 课**：[安全、权限与人工介入](2026-10-19_01_agent_permissions_and_human_approval.md)。以“读取资料、生成周报、批准发送”串联不可信内容、最小权限、操作版本、执行前复核与结果未知。
- **第 10 课**：[鲁棒性与成本预算](2026-10-19_02_agent_reliability_and_budgets.md)。延续周报场景，讲解错误决策、重试放大、deadline、幂等、原子预算预留、限流 / 熔断及显式降级；给出计数和账本验收方案。
- **第 11 课**：[MCP 与连接器](2026-10-19_03_mcp_and_connectors.md)。核对 2025-11-25 官方规范，区分 Host / Client / Server、tools / resources / prompts、Function Calling、传输与授权；串起发现、模型调用、协议请求、结果回填与错误边界。不宣称固定规范是最新版本。
- **第 12 课**：[多 Agent 与委派](2026-10-19_04_multi_agent_and_delegation.md)。比较 router、supervisor、handoff、并行分工；说明任务契约、上下文交接、共享状态、失败与取消，以及相对单 Agent / workflow 的同预算评测。不重复 Harness 源码。
- **第 13 课（阶段 D）**：[Agent 评测与执行环境](2026-10-19_05_agent_evaluation_and_environments.md)。用户要求回到主线，以“生成周报、请求批准、收到拒绝”贯穿任务定义、独立环境与 reset、执行收束、产物 / 状态 / 过程检查、grader、错误分类和公平对比；提供五个待实施用例，不把 Agent 自述或模拟测试当作真实业务完成。
- **第 14 课**：[指标、可观测性与上线](2026-10-19_06_agent_metrics_observability_and_rollout.md)。沿用周报拒绝发送场景，从 100 个教学任务计算严格成功率、工具成功率、费用与 p95，再沿一次失败的 trace 区分上下文遗漏、模型行为和正确的策略拒绝；连接数据保护、回放、发布门槛、shadow / 灰度与回滚。数字和门槛均非生产结果，未接入监控或部署系统。
- **第 15 课**：[Agent 训练数据与学习目标](2026-10-19_07_agent_training_data_and_objectives.md)。从拒绝后仍请求发送的坏例判断工程与训练边界，再讲真实 request 视图、轨迹验证、SFT 监督位置、同条件偏好、环境 rollout 与奖励；补充失败 / 恢复数据、时序泄漏、任务族划分和与不训练方案的对照。未导出真实训练数据、下载权重或启动训练。
- **索引去重整理**：按用户反馈，将[主索引](../index-2026-10-08.md)收敛为一条 Agent 顺序主线；明确第 13 课判单次任务、第 14 课汇总 / 诊断 / 发布、第 15 课构造学习数据、第 16 课集成实施。旧四阶段计划改为当前安排，业务场景改为选题池，系统设计与数据清单指向已有章节；保留 Infra 专项和资料入口，源码历史候选折叠查阅。本次不推进新课、不改写已提供教材。
- **当前实践（第 16 课）**：[实践 01：最小 Agent 闭环设计任务](2026-10-19_08_agent_project_practice_01.md)。用户确认资料 / 周报 Agent、Python、架构主导模式：用户设计与验收，我协助实现。直接把集成课融入项目，第一轮只用内存假资料、模拟模型与双只读工具；提供接口、调用时序和终止边界三道题及待实现验收项，不提前代答。
- **可选扩展主题**：在[主索引的 Harness 设计对照](../index-2026-10-08.md#harness-comparative-reading)加入“OpenClaw 与 DeepSeek Harness”和“LangChain Deep Agents 与 DeepSeek Harness”。用户确认本次仅列主题与分析提纲，正文待展开；前者关注产品目标、入口 / 会话、持续运行与渠道 / 设备，后者关注组装方式、运行时职责、状态与扩展机制。统一用相同任务分析主要取舍，不预写未验证的比较结论；不增加必修课，也不打断当前实践。
- **后续顺序**：用户提交设计 → 评审确认 → 检查环境并编码 / 测试 → 用户验收与题目讲评，再按项目缺口递进。不继续默认扩读其他 Harness；真实模型与外部服务另行确认。
- **状态**：任务卡已提供，设计待提交；尚未检查 Python 环境、编写项目代码或执行验收，也未启动 Harness、发送邮件或恢复暂停的 SFT 实验。不将材料提供记为已掌握。
- **协议课边界**：仅查阅公开官方文档，未安装 MCP SDK、启动 server、连接企业系统或发送本地资料。

## 2026-10-21：对照岗位要求补充大模型数据工程专题

- **对照范围**：[hw_candy 岗位说明](../jd/hw_candy.md)、[个人经历](../experience/zewei_en_202610.md)和现有学习记录。已有 SFT、配比、蒸馏、评测与 Agent 轨迹基础，不判为完全未学；记录覆盖不足也不等于缺乏相关工作经验。
- **新增入口**：[大模型数据工程补充专题](hw_data_engineering/README.md)。包含岗位覆盖对照、与已有经历的衔接、8 篇可直接阅读的讲义及可选综合练习。
- **明显缺项**：预训练多源语料管线、大规模分布式去重、多语言数据均衡、多模态数据与跨模态对齐；质量验证、使用治理和数据飞轮补成完整闭环。
- **已有内容深化**：预训练 token 配比与预算、合成数据验证器校准及成本、轨迹增强算子与状态一致性；不重复已有训练算法推导。
- **状态**：材料已提供，全部待阅读；未采集数据、运行管线、调用教师模型或训练，不预填掌握情况和实验成绩。现有 Agent 实践任务和暂停的 SFT 实验保持不变。

## 2026-10-22：补齐现代 LLM 训练流水线与能力总地图

- **用户指出的缺口**：已有 Transformer、SFT / DPO / RL 与数据工程局部教材，但缺少完整训练阶段、各 domain 的能力演进、典型工作和产物之间的总图。
- **新增前置教材**：[现代 LLM 训练流水线：阶段、能力域与典型工作](2026-10-22_01_llm_training_pipeline_and_capability_map.md)。按 8 类工作包串起目标与评测、数据 / 训练准备、主预训练、定向续训、示范学习、偏好 / 奖励优化、交付适配、部署反馈；不是固定 8 次训练。
- **能力矩阵与工作内容**：覆盖知识 / 专业领域、数学、代码、多语言、长上下文、Agent、多模态、指令 / 格式、安全 / 校准；补典型数据、学习信号、产物、验收及代码任务贯穿例子。
- **关键边界**：训练阶段不独占能力，DPO 与 RL 不固定串行；Base 也可接受生成 / 执行评测；外部测试器运行代码不等于模型自主测试与修复。长上下文、多模态、合成 / 蒸馏和治理按分支或贯穿工作解释。
- **资料与衔接**：核对 OLMo 3、Qwen3、DeepSeek-R1 的公开路线；已接入主索引、Transformer / SFT 前置提示和数据专题入口，不回写旧日期的掌握状态。
- **状态**：讲义已提供，待阅读与答疑；未训练模型、执行 benchmark 或启动新实验，当前 Agent 实践和暂停的 SFT 保持原状态。

## 2026-10-23：DeepSeek-R1 论文深度解读专题

- **用户需求**：建立独立的 R1 论文精读子目录，从训练总地图下钻到具体研究方案，不只解释模型名或总结榜单。
- **入口**：[专题 README 与阅读顺序](deepseek_r1/README.md)、[可视化论文地图](deepseek_r1/overview.html)。
- **正文覆盖**：8 章讲解研究问题、R1-Zero 与奖励、GRPO 公式和手算、R1 四阶段、学生蒸馏、评测协议、失败尝试与可复现边界、精读练习与小规模实验设计。
- **关键辨析**：无前置 SFT 不等于无数据；第三阶段用筛选数据重新微调 V3-Base；最终 R1 的一般任务使用 RM；学生报告采用文本 SFT；多次采样估计 pass@1 不等于 pass@k 或投票。
- **版本与证据**：以 arXiv:2501.12948v1 为主，补官方模型说明；不混入 R1-0528、后续修订或社区库默认配置。提供原文章节与表格索引，教学数字单独标注。
- **状态**：教材已提供，全部待阅读；未下载权重、采样、训练或运行 benchmark。公式手算校验与链接检查不计为模型实验，原 Agent 实践与暂停的 SFT 状态不变。

## 2026-10-24：补充 SFT 数据集的 domain 与输入输出图鉴

- **用户需求**：在 SFT 主课后插入 `01a`，对原文各数据集中的数据按 domain 和 Input / Output 格式做可视化。
- **新增教材**：[2026-10-10_01a_sft_and_training_data.md](2026-10-10_01a_sft_and_training_data.md)、[HTML 图鉴](2026-10-10_01a_sft_and_training_data.html)。覆盖原课 12 个数据集，按 8 个图解板块组织。
- **内容**：对话树、messages、prompt-completion、长推理、代码、函数调用与 Agent 轨迹；补 No Robots 10 类、OpenThoughts 3 域、Cascade 8 域、OpenCode 6 个环境 split，以及 Tulu / Dolci 来源映射。
- **证据边界**：字段依据发布方卡片与有限结构检查，样本内容自拟；xLAM 卡片返回 401，只依据 APIGen 公开材料画逻辑输入输出。未获取受限数据或接受访问条款。
- **状态**：教材待阅读，实际补写日为 2026-10-24；没有把 01a 文件名记作历史学习完成。未下载完整数据集、训练模型、执行样例工具或审计全量质量。
