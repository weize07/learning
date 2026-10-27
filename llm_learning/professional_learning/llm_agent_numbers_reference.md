# LLM / Agent 数字速查表

> 核对日期：**2026-10-10**。用于日常工作查阅与能力积累，不是模型排行榜或完整厂商产品目录。
> 优先使用官方模型卡、配置、数据卡和论文；未公开或未核实的参数不填推测值。数字为一次性快照，不自动更新。

## 1. 先分清数字口径

| 项目 | 正确读法 |
|---|---|
| 1M / 1B / 1T | 通常指 $10^6$ / $10^9$ / $10^{12}$；参数、token、样本必须注明单位 |
| 模型名的“8B”“120B” | 通常为规模标识 / 约数，不保证精确等于张量元素总数 |
| MoE 总参数 | 包括各专家等模型参数；用于理解模型规模，不等于每个 token 全部参与计算 |
| MoE 激活参数 | 厂商报告的每 token 活跃规模；不是 GPU 只需存这部分权重 |
| 非 embedding / effective 参数 | 与含 embedding 的总量不同，不要混作 MoE 激活参数 |
| 上下文长度 | 指指定版本与模式支持范围，通常包含输入和生成；不等于有效记忆或长文准确率 |
| 数据样本数 | 一行可能是问答、整段多轮会话或轨迹，不等于 token 数 |
| HF 建库日期 | API 的 `createdAt`，可能早于公开首发；与论文日期、最近修改日期不同 |

稠密模型下不单列“每 token 激活参数”，避免误以为所有 embedding 表每次都被完整访问。多模态模型还应检查视觉 / 音频编码器是否计入总量。

## 2. 各厂商公开模型：近期代表与常用技术参照

本表覆盖有明确公开数字的代表版本，不声称每一行都是该厂商截至核对日的最新旗舰。“—”表示不适用或未单列，不代表零。

| 厂商 / 系列 | 指定模型 | 架构 / 口径 | 总参数 | 每 token 激活参数 | 日期线索与来源 |
|---|---|---|---:|---:|---|
| 阿里 Qwen | Qwen3-8B | 稠密，规模标识 | 约 8B | — | [官方 config](https://huggingface.co/Qwen/Qwen3-8B/blob/main/config.json)，用于训练 / LoRA 算例 |
| 阿里 Qwen | Qwen3-32B | 稠密，规模标识 | 约 32B | — | [官方 config](https://huggingface.co/Qwen/Qwen3-32B/blob/main/config.json) |
| 阿里 Qwen | Qwen3-235B-A22B | MoE | 235B | 22B | [官方模型卡](https://huggingface.co/Qwen/Qwen3-235B-A22B)，2025 系列 |
| 阿里 Qwen | Qwen3.5-397B-A17B | 混合注意力 + MoE；卡片为语言模型部分口径 | 397B | 17B | [官方模型卡](https://huggingface.co/Qwen/Qwen3.5-397B-A17B)；HF 建库 2026-02-16 |
| DeepSeek | DeepSeek-V3 | MLA + MoE | 671B | 37B | [官方模型卡](https://huggingface.co/deepseek-ai/DeepSeek-V3)，经典规模参照 |
| DeepSeek | DeepSeek-V4-Pro（preview） | MoE / 混合稀疏注意力 | 约 1.6T | 49B | [官方模型卡](https://huggingface.co/deepseek-ai/DeepSeek-V4-Pro)；HF 建库 2026-04-22，报告 2026-06 |
| DeepSeek | DeepSeek-V4-Flash（preview） | MoE / 混合稀疏注意力 | 284B | 13B | [V4 官方说明](https://huggingface.co/deepseek-ai/DeepSeek-V4-Pro)，与 Pro 在同一卡片介绍 |
| Z.ai / 智谱 | GLM-5 | MoE | 744B | 40B | [官方模型卡](https://huggingface.co/zai-org/GLM-5)；HF 建库 2026-02-11 |
| Moonshot / 月之暗面 | Kimi-K2-Instruct | MoE | 1T | 32B | [官方模型卡](https://huggingface.co/moonshotai/Kimi-K2-Instruct)；HF 建库 2025-07-11，指定初版参照 |
| Meta | Llama 4 Scout | 多模态 MoE | 109B | 17B | [官方模型卡](https://github.com/meta-llama/llama-models/blob/main/models/llama4/MODEL_CARD.md)；发布 2025-04-05，16 experts |
| Meta | Llama 4 Maverick | 多模态 MoE | 400B | 17B | [官方模型卡](https://github.com/meta-llama/llama-models/blob/main/models/llama4/MODEL_CARD.md)；发布 2025-04-05，128 experts |
| OpenAI | gpt-oss-120b | MoE，名字不等于精确参数量 | 117B | 5.1B | [官方模型卡](https://huggingface.co/openai/gpt-oss-120b)；HF 建库 2025-08-04 |
| OpenAI | gpt-oss-20b | MoE | 21B | 3.6B | [官方系列说明](https://huggingface.co/openai/gpt-oss-120b) |
| NVIDIA | Nemotron-Cascade-2-30B-A3B | MoE，官方约数 | 30B | 3B | [官方模型卡](https://huggingface.co/nvidia/Nemotron-Cascade-2-30B-A3B)；HF 建库 2026-03-18 |
| Mistral | Mistral Small 3.1 24B | 多模态，官方约数 | 24B | — | [官方模型卡](https://huggingface.co/mistralai/Mistral-Small-3.1-24B-Instruct-2503)，2025-03 版本 |
| Google | Gemma 4 31B | 稠密，卡片报告 | 30.7B | — | [官方模型卡](https://ai.google.dev/gemma/docs/core/model_card_4)，关联报告 2026-07 |
| Google | Gemma 4 12B Unified | encoder-free 多模态 | 11.95B | — | [官方模型卡](https://ai.google.dev/gemma/docs/core/model_card_4) |
| Google | Gemma 4 26B A4B | MoE，名字为约数 | 25.2B | 3.8B | [官方模型卡](https://ai.google.dev/gemma/docs/core/model_card_4) |
| Google | Gemma 4 E2B | 含逐层 embedding 的 effective 口径 | 5.1B（含 embedding） | 非 MoE 激活口径 | [官方模型卡](https://ai.google.dev/gemma/docs/core/model_card_4)：effective 2.3B |
| Google | Gemma 4 E4B | 同上 | 8B（含 embedding） | 非 MoE 激活口径 | [官方模型卡](https://ai.google.dev/gemma/docs/core/model_card_4)：effective 4.5B |

近期模型核对 revision（HF SHA 前缀）：Qwen3.5 `8472618`、V4-Pro `b5968e9`、GLM-5 `c183ef8`、Kimi K2 `fd1984e`、gpt-oss-120b `b5c939d`、Cascade-2 `6327cdb`。动态网页可能修改，正式选型应保存完整 revision 与卡片。

**闭源与内部模型：**

| 类别 | 本表处理 |
|---|---|
| OpenAI 闭源 GPT、Anthropic Claude、Google Gemini 等 API 模型 | 本次未取得可核对的官方参数量，不填网传估算；gpt-oss / Gemma 不能替代对应闭源模型的参数信息 |
| 各厂商内部 Reward / Router / Judge / Draft 模型 | 未公开具体版本与参数量就记为未知，不能由服务名或功能推算 |
| Agent 的 Planner、Executor、Verifier | 是角色，不是固定模型规模；可能同一模型多次调用，也可能使用不同模型 |

## 3. 辅助模型：公开可查的代表

“辅助模型”指系统中承担检索、打分、审核等职责的模型，不声称这些公开权重就是厂商线上内部系统。

| 角色 | 指定模型 / 研究 | 参数量 | 其他关键数字 | 来源 / 日期 |
|---|---|---:|---|---|
| Embedding | Qwen3-Embedding-0.6B | 0.6B | 输出最高 1024 维，支持 32～1024 维 | [官方卡片](https://huggingface.co/Qwen/Qwen3-Embedding-0.6B)，HF 建库 2025-06-03 |
| Embedding | Qwen3-Embedding-4B | 4B | 最高 2560 维 | [官方系列表](https://huggingface.co/Qwen/Qwen3-Embedding-0.6B) |
| Embedding | Qwen3-Embedding-8B | 8B | 最高 4096 维；系列上下文 32K | [官方系列表](https://huggingface.co/Qwen/Qwen3-Embedding-0.6B) |
| Reranker | Qwen3-Reranker 系列 | 0.6B / 4B / 8B | 32K 上下文；排序输出不等于 embedding 向量 | [官方卡片](https://huggingface.co/Qwen/Qwen3-Reranker-0.6B) |
| Reward Model | Skywork-Reward-V2-Qwen3-8B | 约 8B 骨干 | 系列还有小至 0.6B 的版本；主配方报告 26M 偏好对 | [官方卡片](https://huggingface.co/Skywork/Skywork-Reward-V2-Qwen3-8B)，HF 建库 2025-06-26；不同变体不能共用一个训练量 |
| Reward Model | InstructGPT 研究中的 RM | 6B | 为包括 175B 策略在内的模型提供奖励 | [2022 原论文](https://arxiv.org/abs/2203.02155)，历史实例，不是当前内部 RM 的规模 |
| 内容审核 | Llama Guard 3 8B | 约 8B 骨干 | 14 类风险标签，8 种语言 | [官方卡片](https://huggingface.co/meta-llama/Llama-Guard-3-8B)，基于 Llama 3.1 |
| 多模态辅助编码器 | Gemma 4 31B 的 vision encoder | 约 550M | 卡片分别列出编码器规模，不自行推断是否应再加到主表总量 | [官方模型卡](https://ai.google.dev/gemma/docs/core/model_card_4) |
| 推测解码 Draft | 指定 draft 模型需另查 | 未统一 | 有的小模型独立运行，有的模型带专用 MTP / draft 组件 | [Gemma 官方概览](https://ai.google.dev/gemma/docs/core)，不凭主模型名推测 draft 参数 |

理解要点：输出只有一个 reward 标量，并不意味着奖励模型参数很少；1024 维 embedding 也不意味着 embedding 模型只有 1024 个参数。

## 4. 常用 SFT / 偏好数据量级

日期优先标 HF 建库日期或明确版本日期；规模是原始发布数据的指定 split / 配方，**不是每次训练的采样次数或总训练 token**。

| 数据集 | 日期口径 | 数据量 | 单位 / 范围 | 来源与注意 |
|---|---|---:|---|---|
| OASST1 | HF 建库 2023-04-13 | 161,443 | 消息，不是完整会话数 | [官方卡片](https://huggingface.co/datasets/OpenAssistant/oasst1)：35 种语言、超过 10,000 棵完整标注会话树 |
| UltraChat 200k | HF 建库 2023-10-24 | 207,865 | `train_sft` 会话样本 | [官方卡片](https://huggingface.co/datasets/HuggingFaceH4/ultrachat_200k)：`test_sft` 23,110；不要混算 generation splits |
| No Robots | HF 建库 2023-11-10 | 9,500 + 500 | train + test 示范 | [官方卡片](https://huggingface.co/datasets/HuggingFaceH4/no_robots)，合计 10,000 |
| Tulu 3 SFT mixture | HF 建库 2024-11-08 | 约 0.94M | 混合 SFT 样本 | [官方卡片](https://huggingface.co/datasets/allenai/tulu-3-sft-mixture)：API train metadata 为 939,343，正文为 939,344，不将其当成精确一致总数 |
| Aya Dataset | HF 建库 2024-01-31 | 202,362 + 1,750 | default train + test 问答 | [官方卡片](https://huggingface.co/datasets/CohereLabs/aya_dataset)；不把 demographics 配置的行数加进问答量 |
| Orca-Math 200k | HF 建库 2024-03-01 | 200,035 | train 问题与答案 | [官方卡片](https://huggingface.co/datasets/microsoft/orca-math-word-problems-200k)，合成回答需验证 |
| Magicoder OSS-Instruct | HF 建库 2023-12-03 | 约 75K | 编程指令 | [官方项目](https://github.com/ise-uiuc/magicoder) / [数据卡](https://huggingface.co/datasets/ise-uiuc/Magicoder-OSS-Instruct-75K)，规模标识非精确计数 |
| xLAM Function-Calling | HF 建库 2024-06-13 | 60,000 | 函数调用样本 | [官方卡片](https://huggingface.co/datasets/Salesforce/xlam-function-calling-60k)；不等于 60,000 条完整多步 Agent 任务 |
| OpenThoughts3 | HF 建库 2025-05-28 | 1,200,000 | 推理样本 | [官方卡片](https://huggingface.co/datasets/open-thoughts/OpenThoughts3-1.2M)：850K 数学 + 250K 代码 + 100K 科学 |
| Dolci-Instruct-SFT | HF 建库 2025-11-18 | 2,152,112 | train 混合样本 | [官方卡片](https://huggingface.co/datasets/allenai/Dolci-Instruct-SFT)，2026 修改日期不代表全是新样本 |
| Nemotron-Cascade-2 SFT | 明确发布 2026-03-19 | 数百万～千万 / 子域 | 多领域样本，按 config 分开计 | [官方卡片](https://huggingface.co/datasets/nvidia/Nemotron-Cascade-2-SFT-Data)当前统计：数学 5,226,364、科学 2,717,163、General Chat 13,972,873；不沿用二手表的总数，正式使用时重数指定版本并去重 |
| Skywork Reward V2 主训练配方 | 关联报告 2025-07 | 26M | 偏好对 | [官方模型卡](https://huggingface.co/Skywork/Skywork-Reward-V2-Qwen3-8B)，不表示这 26M 对全部开放下载 |

典型公开数据可从 $10^4$ 条人工示范到 $10^6$ 以上混合 / 合成样本，规模不同不直接说明效果高低；应看有效监督 token、质量、覆盖、重复与任务适配。

## 5. 预训练量与评测量：别和 SFT 样本混算

### 5.1 指定模型报告的预训练 token

| 模型 | 预训练量 | 来源 |
|---|---:|---|
| DeepSeek-V3 | 14.8T token | [官方卡片](https://huggingface.co/deepseek-ai/DeepSeek-V3) |
| GLM-5 | 28.5T token | [官方卡片](https://huggingface.co/zai-org/GLM-5)；其对照 GLM-4.5 为 23T |
| Llama 4 Scout | 约 40T token | [Meta 官方卡片](https://github.com/meta-llama/llama-models/blob/main/models/llama4/MODEL_CARD.md) |
| Llama 4 Maverick | 约 22T token | [同一卡片](https://github.com/meta-llama/llama-models/blob/main/models/llama4/MODEL_CARD.md) |

训练 token 是处理量，不必等于独立、去重的语料规模；可能多次曝光、包含多模态口径。不能换算成固定篇数。

### 5.2 常见评测基准规模

| 基准 | 量级 | 口径与来源 |
|---|---:|---|
| GSM8K | train 7,473；test 1,319 | [官方发布数据](https://github.com/openai/grade-school-math) / [HF main 元数据](https://huggingface.co/api/datasets/openai/gsm8k)，不将 train 与 test 混用 |
| HumanEval | 164 | [官方发布](https://github.com/openai/human-eval) / [HF 元数据](https://huggingface.co/api/datasets/openai/openai_humaneval)，编程题，不是 164 个生成样本 |
| SWE-bench Verified | 500 | [官方数据卡](https://huggingface.co/datasets/SWE-bench/SWE-bench_Verified)，人工筛查的 issue test 子集 |

这些是指定历史版本参照，不是 2026 年最新题库排名。公开基准存在污染风险；Agent 成绩还受工具、环境、步数预算和采样次数影响，不能只看模型名称。

## 6. 参数与显存：工程常用估算

### 6.1 权重存储下限

$$
S_{\text{weights}}\approx N_{\text{params}}\times s
$$

| 存储 | 每参数理论字节数 | 8B 参数权重理论大小 |
|---|---:|---:|
| FP32 | 4 | 32 GB，约 29.8 GiB |
| FP16 / BF16 | 2 | 16 GB，约 14.9 GiB |
| INT8 | 1 | 8 GB，约 7.45 GiB |
| 4 bit | 0.5 | 4 GB，约 3.73 GiB |

这里只算权重有效负载；量化 scales、未量化模块、KV、激活、优化器、碎片和工作区另算。量化不保证等比例变快。GB 为 $10^9$ 字节，GiB 为 $2^{30}$ 字节。

### 6.2 KV Cache

标准、各层相同 KV 结构、全上下文、同 dtype 的简化模型：

$$
S_{\text{KV}}=2LBTh_{kv}d_hs
$$

例如 $L=32,B=1,T=4096,h_{kv}=8,d_h=128,s=2$：

$$
S_{\text{KV}}=536{,}870{,}912\text{ bytes}=512\text{ MiB}
$$

相同条件下 MHA 的 32 个 KV 头占 2 GiB；MQA 的 1 个 KV 头占 64 MiB。MLA、混合注意力、滑动窗口等不直接套此式。

### 6.3 投影与 LoRA

| 项目 | 权重参数量，不计偏置 |
|---|---|
| 标准等宽 MHA，Q/K/V/O | $4d^2$ |
| 一般同每头维度 GQA | $2dh_qd_h+2dh_{kv}d_h$ |
| 普通 FFN，宽度 d → f → d | $2df$ |
| 常见 SwiGLU | $3df$ |
| LoRA，一个 in × out 投影 | $r(d_{\text{in}}+d_{\text{out}})$ |

$4096\times4096$ 原矩阵为 16,777,216 个权重；r=8 的 LoRA 为 65,536，约 1/256。原权重仍存在；训练参数减少不等于整体 FLOPs 减少同一比例。

## 7. 两个真实模型的 Attention 维度

| 模型 | d_model | Q heads | KV heads | head_dim | Q in → out | K/V in → out |
|---|---:|---:|---:|---:|---|---|
| [Qwen3-8B](https://huggingface.co/Qwen/Qwen3-8B/blob/main/config.json) | 4096 | 32 | 8 | 128 | 4096 → 4096 | 4096 → 1024 |
| [Qwen3-32B](https://huggingface.co/Qwen/Qwen3-32B/blob/main/config.json) | 5120 | 64 | 8 | 128 | 5120 → 8192 | 5120 → 1024 |

行向量公式 $XW$ 中 W 为 `[in,out]`；PyTorch Linear.weight 通常为 `[out,in]`。Q 总宽度不必等于 d_model，输出投影负责回到残差接口宽度。

## 8. Agent / RAG 数字：公式比“万能配置”更可靠

| 场景 | 数字关系 | 注意 |
|---|---|---|
| Agent 上下文预算 | 系统提示 + 历史 + 证据 + 工具描述 / 结果 + 生成预留 ≤ 可用窗口 | tokenizer 实测，不用汉字数当 token 数 |
| 单请求 token 成本 | 输入 token × 输入单价 + 输出 token × 输出单价 | 所有模型调用累计；缓存、长上下文与思考计费查看供应商 |
| 多步成本 | 所有步输入 / 输出累计 | 历史重复发送可能多次计费，不能只数最终回答长度 |
| 纯向量存储 | 文档向量数 × 输出维度 × 字节数 | 1M 条 × 1024 维 × FP32 = 4.096 GB；不含索引、元数据和副本 |
| pass@1 | 单次生成条件下的通过比例 | 与 best-of-N、pass@k 不直接比较；公开采样预算 |
| 串行工具延迟 | 各步骤模型与工具等待累加 | 并行不一定全部可并行；关注 p50 / p95 与超时 |
| 失败率估计 | 独立同分布条件下，n 次零失败的 95% 单侧上界约 3/n | 100 次零失败不等于 100% 可靠；聚类 / 分布漂移时不能直接套 |

**没有全行业固定值**的配置：chunk size、top-k、rerank 候选数、Agent 最大步数、重试次数、temperature、LoRA rank、回放比例。速查表应记录实际项目的预算与选择理由，不把随手举例的 8 / 16 / 32、80/20 等当通用标准。

## 9. 更新规则与现有实验

- 动态榜单分数、API 价格、stars 单独保留时间与评测 / 统计平台，不混作永久数字。
- 数据集规模随 revision 变化；经典数据集的年份与新版本要分开。
- 本次只是官方公开资料核对，未独立审核全部模型权重 / 数据行，也未测完各模型效果。
- 本地硬件实测：RTX 4080 Laptop，12282 MiB 显存；PyTorch 2.8.0+cu128，CUDA runtime 12.8，BF16 支持。依据当前实验环境，不作为所有同型号机器的可用容量。
- 本地 [SFT pilot](experiments/grounded_qa/README.md)：8 条训练候选 + 3 条共享来源的诊断问题；无独立验证 / 测试集，无正式任务分数。不要将其写成“训练了一个已验证领域模型”。

相关入口：[学习路线](index-2026-10-08.md)、[每日学习概览](learning_records/topics_by_date.md)、[SFT 数据集与工具表](learning_records/2026-10-10_01_sft_and_training_data.md)。
