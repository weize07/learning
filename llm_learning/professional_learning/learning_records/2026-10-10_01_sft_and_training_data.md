# SFT：监督微调与训练数据

- 学习日期：2026-10-10。
- 每日概览：[topics_by_date](topics_by_date.md)。
- 学习索引：[工作能力提升与实践手册](../index-2026-10-08.md)。
- 前置阅读：[Softmax 与训练损失](2026-10-08_03_softmax_and_losses.md)、[Transformer 总体结构](2026-10-08_01_transformer_overview.md)。
- 流程总图（2026-10-22 补充）：[现代 LLM 训练流水线与能力地图](2026-10-22_01_llm_training_pipeline_and_capability_map.md)，先区分各阶段作用、数据域与能力域，再进入本篇 SFT 细节。
- 状态：讲义已提供，阅读与掌握情况待反馈；自测可选。
- 数据样本图解（2026-10-24 回补）：[01a · 按 domain 看 12 个数据集的 Input / Output](2026-10-10_01a_sft_and_training_data.md)，配套[可视化页面](2026-10-10_01a_sft_and_training_data.html)。先区分原始字段、训练条件与监督目标。

现在从模型结构转向应用 / 算法 / 数据。先看第 1～3 节的完整流程，再看样本、标签与评测。不先讨论分布式训练或 GPU 调优。

## 1. SFT 在整个训练流程中的位置

SFT 是 Supervised Fine-Tuning，即监督微调：用示范输入与目标输出，继续训练已有模型，让它更符合目标任务的行为。

```text
预训练模型 / 已有指令模型
          +
任务示范数据：输入 → 期望输出
          ↓
整理数据 → 模型对应的格式与 tokenizer → 输入与标签
          ↓
模型前向计算 → 对目标 token 计算交叉熵
          ↓
反向传播 → 更新允许训练的参数
          ↓
独立评测 → 检查收益、退化与失败案例
```

常见总体路线是预训练 → SFT → 可选偏好优化 / 强化学习，但实际模型的训练阶段可以交错或不同，不是每个模型都必须经过全部阶段。

SFT 通常仍是 next-token 预测，没有为每条问题增加独立的分类输出层。变化主要在示范数据及参与损失的范围。

## 2. 它让模型学什么？什么时候值得做？

适合的目标包括：

- 输出稳定的任务格式，例如固定字段的结构化抽取。
- 更好遵循特定指令、回答风格或业务流程。
- 学习领域任务的输入到输出映射。
- 学习工具调用的参数组织与何时请求工具，前提是训练格式与推理协议一致。

SFT 也可能学习事实知识，但不能保证准确记忆、持续更新或可靠引用。因此不要把它当成必须查证的动态知识库。

| 方法 | 主要改变什么 | 常见适用情况 |
|---|---|---|
| Prompt / few-shot | 本次上下文，不更新权重 | 先建立基线，任务可通过示例与说明完成 |
| RAG / 工具 | 推理时提供知识与外部能力 | 知识更新、溯源、业务查询与执行 |
| SFT | 模型参数或适配器 | 有可靠示范，需改善任务行为 |
| DPO / RL 等 | 按偏好或奖励优化策略 | 后续专题，目标不只是模仿示范 |

应用决策顺序：先定义指标并评测 prompt 基线；需要知识时补检索 / 工具；确认存在可训练的行为差距和可靠数据后，再做 SFT 实验。这些方法可以组合。

微调不能替代后端权限校验、schema 校验或事务约束。模型学会正确格式，不代表永远不会生成错误参数。

## 3. SFT 与 LoRA 是同一回事吗？

不是，它们回答不同问题：

- **SFT：用什么数据和目标训练？** 模仿示范输出，通常用 token 交叉熵。
- **Full fine-tuning / LoRA：哪些参数如何更新？** 全量微调更新较多原模型参数；LoRA 通过低秩适配器训练一部分新增参数等。

因此可以有“全量 SFT”，也可以有“LoRA SFT”。LoRA 同样可以服务其他训练目标，不只用于 SFT。

下一专题再推导 LoRA / QLoRA；本节先把数据和损失学清楚。

## 4. 一条样本如何变成模型输入？

### 4.1 原始数据：内容与角色

以下是示意数据，不保证每个训练框架直接接受同一字段结构：

```json
{
  "messages": [
    {
      "role": "user",
      "content": "从“订单编号 A123，金额 99 元”中提取订单编号和金额，只输出 JSON。"
    },
    {
      "role": "assistant",
      "content": "{\"order_id\":\"A123\",\"amount\":99}"
    }
  ]
}
```

这里不只是收集“好的回答”，还需要保留使回答成立的条件：问题、必要上下文、工具结果、角色和轮次。

### 4.2 Chat template：让角色与边界可识别

对话模型通常需要对应的模板，将消息序列转换成含角色和结束标记的 token 序列。例如概念上：

```text
[用户角色标记] 问题 [消息结束]
[助手角色标记] 目标回答 [回合结束]
```

这些方括号是解释用标签，不是某个模型真实特殊 token。不同模型的模板不同，应使用目标 tokenizer / 模型规定的模板。

训练时包括目标回答；推理时通常只提供已有上下文和适用的助手起始标记，让模型生成回答。不要把真实目标回答误塞进推理输入再评测。

模板错误、重复添加特殊 token、结束标记缺失，可能让模型学习到错误边界。不能只凭原始 JSON 看起来正确，就认为输入也正确。

## 5. Teacher forcing：训练时为什么把答案放进输入？

训练时，预测回答的第 $r$ 个 token，使用的是**真实示范回答的前 $r-1$ 个 token**，而不是模型刚刚采样的 token。这称为 teacher forcing。

```text
问题 + 真实回答前缀 → 预测真实回答的下一个 token
```

真实未来 token 虽然在整个训练张量里，但 causal mask 阻止提前读取。这样一条序列中的多个预测位置可以并行训练。

推理时没有真实答案前缀，只能使用模型自己已生成的回答前缀。因此训练损失下降不保证自由生成时效果同幅度提升，需要实际生成评测。

## 6. 哪些位置计算损失？用小例子看标签右移

为了专注对齐，暂时省略真实角色标记、BOS 等。假设序列只有：

```text
输入 token：[P0, P1, A0, A1, E]
```

P 表示 prompt，A 表示答案，E 表示答案结束标记。采用只监督答案与结束标记的配置：

| 输入位置 | 本位置输入 | 本位置 logits 的下一 token 目标 | 是否算损失 |
|---|---|---|---|
| 0 | P0 | P1 | 否 |
| 1 | P1 | A0 | 是 |
| 2 | A0 | A1 | 是 |
| 3 | A1 | E | 是 |
| 4 | E | 无后续目标 | 否 |

**最后一个 prompt 位置的 logits 负责预测第一个答案 token。** “prompt 不算损失”更准确地说是“prompt token 不作为监督目标”，不能简单删除所有由 prompt 输入位置产生的预测。

某些框架接收未右移的 labels，再在内部将 logits 与 labels 错开一位。例如这类接口中的概念标签可为：

```text
input_ids：[P0,   P1,   A0, A1, E]
labels：   [忽略, 忽略, A0, A1, E]
```

内部用位置 1 的 logits 对齐位置 2 的 label A0。一些框架用 `-100` 表示忽略目标；这是接口约定，不是真实 token ID。应检查具体实现，不要内部已经右移、外部又右移一次。

一般的目标回答损失：

$$
\mathcal L_{\text{SFT}}
=-\frac{1}{M}\sum_{r=1}^{M}
\log P_\theta(a_r\mid p,a_1,\ldots,a_{r-1})
$$

$p$ 是 prompt / 对话上下文；$M$ 是本例参与监督的目标 token 数，是否含结束标记按配置决定。多个样本如何加权或归约也由训练设置决定。

### 6.1 三种“mask”不要混淆

| 限制 | 回答什么问题 |
|---|---|
| Causal mask | 此 query 能读取哪些过去 / 当前输入位置？ |
| Padding mask / 有效长度 | 哪些位置是真实内容，哪些是补齐槽位？ |
| Loss mask / 忽略标签 | 哪些目标 token 参与损失？ |

prompt 不参与目标损失，仍可以且通常必须被答案读取。标签里忽略 padding，并不自动保证注意力也排除了所有无效位置，需要配置两方面。

SFT 也可采用全序列损失，不是所有任务都必须 answer-only。completion-only 与多轮 assistant-only 也有范围区别，应先明确想监督哪些角色、哪些轮次。

## 7. 数据质量：比“堆很多问答”更重要

### 7.1 先写任务定义与标注规则

以上面的订单抽取为例：

- 字段类型是什么？金额是数值还是字符串？
- 缺失订单编号时如何输出？
- 多个订单怎样组织？
- 示例中的金额是否确实来自输入？
- 无法确定时用 null、拒绝回答还是请求澄清？

规则先明确，再收集示范。冲突标签会让相同条件对应不同目标，模型难以稳定学习。

### 7.2 处理常见数据问题

| 问题 | 建议检查 |
|---|---|
| 错误答案与幻觉 | 目标是否有上下文依据，标注是否经过验证 |
| 重复 / 近重复 | 避免重复样本占据大部分训练权重 |
| 只覆盖简单题 | 覆盖真实失败类型、缺失信息与边界案例 |
| 风格、字段不一致 | 统一协议或明确条件差异 |
| 过度模板化 | 使用真实多样表达，避免只学表面格式 |
| 超长截断 | 截断后是否仍有条件与完整目标，是否剩下有效监督 token |
| 敏感数据 | 使用有授权、按要求去标识化的数据 |

合成数据可扩充覆盖，但需检查教师错误、重复与任务偏差；数量不能代替质量。

### 7.3 划分数据，防止泄露

训练集用于更新参数；验证集用于选超参数与 checkpoint；测试集用于最终独立评估。

不能只逐条随机分割近重复数据。应按实际相关性考虑按文档、客户、会话、任务模板族或时间等分组划分，并跨集合检查近重复。

例如同一订单的十种改写若分别进入训练和测试，会高估对新订单、新表达的泛化。

### 7.4 常用公开 SFT 数据集与选型入口

配套阅读：[数据图鉴 01a](2026-10-10_01a_sft_and_training_data.md)逐项展开下面 12 个数据集的 domain、字段结构和输入输出；示例内容为自拟，字段证据与访问限制另行标注。

下面是公开指令微调中常见或有代表性的参考数据集，不是使用率排名，也不是推荐全部混入同一次训练。链接指向发布方数据卡，信息于 **2026-10-10** 联网核对；经典资源与近期资源分开列出。

**日期口径：** 未确认首次公开公告时，用 Hugging Face API 的 `createdAt` 标为“建库日期”，不冒充首次公开发布日期；它也可能早于文件上传或公开。`lastModified` 是仓库更新时间，可能只是 README 改动，不代表数据新增。API 来源为 `https://huggingface.co/api/datasets/{发布方}/{数据集名}`，例如 [OASST1 元数据](https://huggingface.co/api/datasets/OpenAssistant/oasst1)。

**经典资源：**

| 数据集名称 | 链接 | HF 建库日期（UTC） | 训练领域 / 目的 | 数据与使用注意 |
|---|---|---|---|---|
| OpenAssistant OASST1 | [OpenAssistant/oasst1](https://huggingface.co/datasets/OpenAssistant/oasst1) | 2023-04-13 | 通用助手、多轮对话、多语言指令遵循 | 众包消息树与质量标注；需选取合适路径、过滤质量并映射角色，不是直接扁平问答 |
| UltraChat 200k | [HuggingFaceH4/ultrachat_200k](https://huggingface.co/datasets/HuggingFaceH4/ultrachat_200k) | 2023-10-24 | 英文通用对话、多轮指令微调 | 过滤后的合成对话；SFT 使用 `train_sft`，不要混用 `test_sft` 或将 generation split 当成相同目的的数据 |
| No Robots | [HuggingFaceH4/no_robots](https://huggingface.co/datasets/HuggingFaceH4/no_robots) | 2023-11-10 | 写作、问答、改写、摘要、分类、抽取等通用指令 | 约 1 万条人工示范，主要单轮；CC BY-NC 4.0，非商业限制需特别注意 |
| Tulu 3 SFT Mixture | [allenai/tulu-3-sft-mixture](https://huggingface.co/datasets/allenai/tulu-3-sft-mixture) | 2024-11-08 | 多能力混合：通用对话、数学、代码、指令遵循、多语言、安全等 | 适合研究混合配方与 `source` 字段；部分来源非商业，集合许可不等于所有子集统一许可；会与本表部分数据重叠 |
| Aya Dataset | [CohereLabs/aya_dataset](https://huggingface.co/datasets/CohereLabs/aya_dataset) | 2024-01-31 | 多语言指令遵循、跨语言任务覆盖 | 人工标注 prompt-completion，含中文等多语言；不是 Aya Collection 或 Aya 模型权重，需核对实际语言覆盖与质量 |
| Orca-Math Word Problems 200k | [microsoft/orca-math-word-problems-200k](https://huggingface.co/datasets/microsoft/orca-math-word-problems-200k) | 2024-03-01 | 数学应用题、解题过程与答案生成 | 约 20 万题，回答由 GPT-4 Turbo 生成；需验证解答并检查评测重叠，数据卡另有范围说明 |
| Magicoder OSS-Instruct 75K | [ise-uiuc/Magicoder-OSS-Instruct-75K](https://huggingface.co/datasets/ise-uiuc/Magicoder-OSS-Instruct-75K) | 2023-12-03 | 代码生成、编程指令微调 | OSS-Instruct 合成代码指令；结合项目资料核对来源、生成服务条款与代码许可，优先用测试验证目标代码 |
| xLAM Function-Calling 60k | [Salesforce/xlam-function-calling-60k](https://huggingface.co/datasets/Salesforce/xlam-function-calling-60k) | 2024-06-13 | 工具选择、参数生成与函数调用 | APIGen 生成并经过格式、执行、语义验证；需适配目标模型的 tool schema / 对话协议，不等同于完整多步 Agent 轨迹 |

**2025～2026 近期资源：经发布方卡片核对的代表性增补，不是截至核对日所有新品的完整清单。**

| 数据集 | 日期与依据 | HF 最近更新（UTC） | 领域 / 目的 | 注意事项 |
|---|---|---|---|---|
| [Nemotron-Cascade-2-SFT-Data](https://huggingface.co/datasets/nvidia/Nemotron-Cascade-2-SFT-Data) | **2026-03-19 发布**，官方 README 明确标注；HF 同日建库 | 2026-03-19 | 数学、科学、对话、指令遵循、安全、对话工具 Agent、SWE 与终端 Agent 的混合 SFT | NVIDIA Open Model License；包含多源合成回答与轨迹，需按子任务筛选；论文 / 卡片中的训练配方不应直接照搬到小规模业务 |
| [Nemotron-SFT-OpenCode-v1](https://huggingface.co/datasets/nvidia/Nemotron-SFT-OpenCode-v1) | **2026-03-13 建库**，公开首发日未另确认 | 2026-03-23 | OpenCode CLI 代码 Agent、工具调用、技能使用与澄清交互 | 当前卡片为 CC BY 4.0；卡片另写 `01/09/2026` 为创建日期，格式与意义不同，不将其当作明确首发日；工具环境需适配 |
| [Dolci-Instruct-SFT](https://huggingface.co/datasets/allenai/Dolci-Instruct-SFT) | **2025-11-18 建库**，不是因 2026 年更新就称为 2026 新数据 | 2026-02-03 | Olmo 3 的混合指令 SFT，含代码、数学、多语言、指令遵循与工具使用 | 当前卡片为 ODC-BY，另有负责任使用要求；约 215 万条、含来源 / 领域字段；检查来源许可，不沿用二手清单的许可判断 |
| [OpenThoughts3-1.2M](https://huggingface.co/datasets/open-thoughts/OpenThoughts3-1.2M) | **2025-05-28 建库**；[关联论文](https://arxiv.org/abs/2506.04178)为 2025-06，二者不是同一日期 | 2025-06-09 | 数学、代码、科学的推理示范，QwQ-32B 生成回答 | Apache 2.0；120 万条，85 万数学 / 25 万代码 / 10 万科学；推理轨迹需要匹配模型格式、验证答案与防止评测污染 |

此次核对可看到的方向变化：从短问答扩展到推理示范、跨能力混合，以及带工具环境的多步 Agent 轨迹。对当前主线，近期优先研究混合数据的来源字段、工具轨迹格式和验证方式，而不是只追求条数或年份新。

近期卡片核对版本：Cascade-2 `9f36020`、OpenCode-v1 `556d523`、Dolci `bd3c8f3`、OpenThoughts3 `61bcf9d`（HF revision SHA 前缀）。规模与质量介绍依据发布方，未独立下载审核全部数据。

通用使用原则：

- “公开可下载”不等于可以无条件商用；检查指定版本的数据许可、来源许可、生成服务条款及其他限制。
- 按数据集说明选择训练 split，保留独立评测数据；不要把评测基准的测试集当成训练集。
- 通用英文数据不能替代真实中文业务样本；根据目标语言、输入长度、任务和输出协议筛选。
- 混合大集合与它的来源子集时，要跨来源去重，否则会隐性提高某类任务的训练权重。
- 合成示范可能有错误，不能将模型生成内容自动视为标准答案。

### 7.5 后续专题：数据分布设计与遗忘控制

会作为应用 / 算法 / 数据主线深入学习，放在 LoRA / QLoRA 之后，而不是仅作为 Infra 概要。当前先明确路线，不预设一个通用最优配方：

1. **分布设计**：按任务、语言、难度、长度、格式和来源建立数据画像；区分真实线上流量与需要加强的失败类型。
2. **混合与采样**：区分样本占比、被采样次数、监督 token 占比和损失权重；长回答可能在按 token 归约的目标中贡献更多，不能只数问答条数。
3. **遗忘诊断**：微调前后用独立的目标任务集与保留能力集比较，分析真实能力退化、模板变化、生成参数变化和评测噪声。
4. **缓解方法**：代表性旧任务数据回放 / 混合、学习率与训练步数控制、checkpoint 选择、参数高效微调，以及适用时的教师蒸馏 / 分布约束。
5. **实验取舍**：通过消融与多任务评测观察目标收益和旧能力损失，设置明确的退化阈值；没有必要复刻不可获得的全部预训练数据。

“旧任务数据”可以是有授权的代表性指令样本，不要求持有基础模型的完整训练语料。用于保留能力的训练数据，也必须与检测遗忘的评测集分离。

**这些是降低遗忘风险的方法，不是保证不遗忘。** 冻结基础权重训练 LoRA 仍会改变启用适配器时的输出；关闭适配器回到基础模型，不等于启用适配器时旧能力没有退化。混入通用数据也可能减少新任务收益，必须做配方实验。

### 7.6 SFT 开源工具包与选型

下面区分直接组织 SFT 的训练工具与可以组合使用的组件。名称及项目入口于 2026-10-10 核对；不是性能排名。支持的模型、硬件与参数以指定版本文档为准。

| 工具 | 项目 / 文档链接 | 定位与典型用途 | 适合的学习 / 使用场景 |
|---|---|---|---|
| Hugging Face TRL | [项目](https://github.com/huggingface/trl) / [SFTTrainer](https://huggingface.co/docs/trl/sft_trainer) | Python 训练库，提供 SFTTrainer，并覆盖其他后训练方法 | 希望看清数据处理、监督范围和训练参数，编写可控的算法实验；可与 PEFT 组合 |
| LLaMA-Factory | [项目](https://github.com/hiyouga/LLaMA-Factory) | 集成微调框架，通过配置、命令行与 WebUI 等入口组织训练 | 快速跑通模型与自定义数据的微调基线，减少脚手架代码；仍需检查模板和 loss 配置 |
| Axolotl | [项目](https://github.com/axolotl-ai-cloud/axolotl) | 配置驱动的 LLM 微调框架，提供数据与训练配置入口 | 用 YAML 管理、复用训练配方，开展多组可复现实验 |
| ModelScope ms-swift | [项目](https://github.com/modelscope/ms-swift) / [中文文档](https://swift.readthedocs.io/zh-cn/latest/) | ModelScope 生态的模型与多模态微调框架，也提供评测、推理等流程 | 关注中文文档、ModelScope 模型与多模态任务，希望使用集成工具链 |
| Unsloth | [项目](https://github.com/unslothai/unsloth) / [文档](https://unsloth.ai/docs) | 包含高效微调实现与示例的工具生态 | 在受限显存条件下尝试支持模型的微调；需验证具体模型、训练路径与硬件兼容性，不照搬宣传加速倍数 |
| Hugging Face PEFT | [项目](https://github.com/huggingface/peft) / [文档](https://huggingface.co/docs/peft/index) | 参数高效适配库，提供 LoRA 等方法 | 控制适配器及可训练参数；通常搭配 TRL / Transformers 等训练组件，而不是独立的完整 SFT 工作流 |
| Hugging Face Transformers | [项目](https://github.com/huggingface/transformers) / [Trainer](https://huggingface.co/docs/transformers/main_classes/trainer) | 模型、tokenizer、模板与通用训练 API | 需要更细的模型或训练控制；SFT 数据格式化、标签与损失范围需按具体实现接好 |

这些工具并非互斥。例如一个实验可以用 Transformers 加载模型，用 PEFT 配置 LoRA，再用 TRL 的 SFTTrainer 组织训练；集成框架也会调用其他库。

**项目热度与活跃度快照**

核对时间：**2026-10-10 11:49（UTC+8）**。Stars 来自 GitHub REST API 的 `stargazers_count`；最近提交取默认分支最新提交的 committer 时间，统一显示为 UTC。所有下列仓库在查询时均未归档。

| 工具 | GitHub stars | 默认分支最近提交（UTC） | 相对核对日的活动线索 |
|---|---:|---|---|
| [TRL](https://github.com/huggingface/trl) | 19,479 | 2026-10-09 16:15 | 近 24 小时有提交 |
| [LlamaFactory](https://github.com/hiyouga/LlamaFactory) | 75,389 | 2026-09-28 09:10 | 近 14 天有提交 |
| [Axolotl](https://github.com/axolotl-ai-cloud/axolotl) | 12,548 | 2026-10-09 11:19 | 近 24 小时有提交 |
| [ms-swift](https://github.com/modelscope/ms-swift) | 15,805 | 2026-10-10 02:47 | 近 24 小时有提交 |
| [Unsloth](https://github.com/unslothai/unsloth) | 77,656 | 2026-10-10 03:35 | 近 24 小时有提交 |
| [PEFT](https://github.com/huggingface/peft) | 21,774 | 2026-10-09 10:21 | 近 24 小时有提交 |
| [Transformers](https://github.com/huggingface/transformers) | 166,950 | 2026-10-09 17:01 | 近 24 小时有提交 |

统计口径来源：`https://api.github.com/repos/{owner}/{repo}` 与 `/commits?per_page=1`，例如 [TRL 仓库 API](https://api.github.com/repos/huggingface/trl)。这是一次性快照，不是自动更新，也没有统计过去一周的提交总数、issue 响应时间或版本兼容性。Stars 衡量关注度；一个文档 / 自动化提交也会更新日期，两者都不能单独证明稳定性或维护质量。

**关联论文与引用数**

引用数不能和 stars 混为一谈，也不代表当前版本的工具效果。统计平台、收录范围不同，数字不能直接跨平台比较。

| 工具 | 关联论文 | 论文引用数（2026-10-10 核对） |
|---|---|---|
| LlamaFactory | [LlamaFactory: Unified Efficient Fine-Tuning of 100+ Language Models](https://arxiv.org/abs/2403.13372) | 暂未取得：Semantic Scholar API 返回 HTTP 429 限流；不采用 README 的引用徽章作实时数值 |
| ms-swift | [SWIFT: A Scalable lightWeight Infrastructure for Fine-Tuning](https://arxiv.org/abs/2408.05517) | **466**，来自 [Semantic Scholar 对应论文记录](https://www.semanticscholar.org/paper/e76567345cb4f1cb0f83f2c20bf849066a7e8346)的 `citationCount` |
| Transformers | [Transformers: State-of-the-Art Natural Language Processing](https://arxiv.org/abs/1910.03771) | 暂未取得：Semantic Scholar API 返回 HTTP 429 限流 |
| TRL / Axolotl / Unsloth / PEFT | 本轮不将方法论文或生态论文强行映射成工具论文 | 未统计，不表示零引用或不存在关联论文 |

例如 PEFT 支持 LoRA，不代表 LoRA 论文的引用数就是 PEFT 工具的引用数。后续更新应重新查询同一统计平台，保留日期；缺失值明确标注，不填猜测数字。

**按当前学习目标的建议：**

- 以 **TRL + Transformers + PEFT** 理解机制，掌握 token / labels / loss mask 与可训练参数。
- 想先低代码跑通基线，可选 **LLaMA-Factory**；偏 ModelScope / 多模态可考虑 **ms-swift**。
- 想配置化维护实验，可考察 **Axolotl**；受限显存时可考察 **Unsloth** 的适用路径。
- 不需要同时学习所有框架。先在一个工具上跑通数据检查、训练、独立评测与模型 / 适配器导出。

选择前核对：

1. 目标模型、chat template、工具调用或多模态格式是否受支持。
2. completion-only / assistant-only、packing、截断、padding 与标签右移的行为是否满足需求。
3. 是否支持所需的全量微调或 LoRA / QLoRA 路径，以及输出适配器如何加载 / 导出。
4. 固定版本、数据版本、随机种子与超参数，保留实际监督 token 检查结果，不只保存一条启动命令。
5. 检查当前 OS、GPU、驱动与依赖要求；提供 UI 或桌面入口不意味着所有训练后端都原生支持 Windows。
6. 工具代码许可、模型权重许可和数据许可分别核对；开源工具不使其他资产自动可商用。

DeepSpeed、FSDP、Accelerate 等属于训练执行与分布式 / 内存管理组件，不是另一种 SFT 数据集或训练目标。当前作为 Infra 第二优先级的资料入口，不先展开底层实现。

## 8. 一个最小可讲清楚的实验

以结构化抽取为例：

1. 明确输入、字段规范、缺失信息处理与业务指标。
2. 用同一个基础模型建立 prompt / few-shot 基线。
3. 建立独立数据集与标注检查，保留数据版本和处理记录。
4. 验证模板、右移和 loss mask：抽查实际 token 与监督位置，确保不是零有效标签。
5. 小规模 SFT，先确定训练链路正确，不盲目扩大预算。
6. 使用一致的提示词与生成配置，在独立集合上比较。
7. 分类分析坏例，必要时补数据、修标签或调整方法。

关注的指标：

- JSON 解析成功率与 schema 合规率。
- 字段准确率、完整记录准确率，缺失字段处理正确率。
- 不受支持信息的编造率。
- 泛化到不同表达、长上下文和边界输入的效果。
- 原有其他任务是否退化，以及实际延迟 / 成本。

训练损失只说明在监督序列上拟合的程度，不等于业务成功率。LoRA 的参数少也不保证不会遗忘或过拟合。

## 9. 可选自测与资料

1. SFT 是架构、参数更新方式，还是训练任务 / 目标？
2. prompt 不算目标损失，模型还能读取它吗？
3. 谁的 logits 预测第一个答案 token？
4. 为什么随机划分近重复问答可能使评测虚高？

<details>
<summary>参考答案</summary>

1. 是训练任务 / 目标与数据设置；可以搭配全量微调或 LoRA。
2. 可以；loss mask 与 attention 可见性是不同设置。
3. 紧邻第一个答案 token 之前的位置，包含实际模板时以序列对齐为准。
4. 测试条件与训练过于相似，可能测到记忆而非独立泛化。

</details>

资料索引：

- [TRL：SFT Trainer](https://huggingface.co/docs/trl/sft_trainer)：训练数据类型与监督范围。
- [Transformers：Chat templates](https://huggingface.co/docs/transformers/chat_templating)：角色与特殊标记。
- [Transformers：Causal language modeling](https://huggingface.co/docs/transformers/tasks/language_modeling)：next-token 训练。

学习记录：

- 已提供：SFT 定位、数据到训练流程、标签对齐、数据质量与实验评测，以及公开数据集和开源工具包选型表。
- 用户关注：后续需要深入学习数据分布设计，以及 SFT 后旧任务遗忘的诊断与缓解；列为独立专题，不保证任何方法能完全防止遗忘。
- 阅读反馈 / 答疑：待补充。
- 已验证掌握：暂无。
- 后续顺序：[01a 数据图鉴](2026-10-10_01a_sft_and_training_data.md)（2026-10-24 回补）→ LoRA / QLoRA → 数据分布与混合采样 → 遗忘诊断与缓解。回补不改写原学习日期与掌握状态。
