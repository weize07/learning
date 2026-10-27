# DeepSeek-R1 论文深度解读

**核心问题：已有预训练能力，怎样通过奖励驱动的探索、筛选后的示范学习与再次对齐，变成更可靠的推理行为？** R1 不只是“换成 GRPO”，也不是“取消 SFT”。本专题把论文的研究问题、训练信号、数据流、实验和证据边界连起来。

- 编写日期：2026-10-23。
- 主论文：*DeepSeek-R1: Incentivizing Reasoning Capability in LLMs via Reinforcement Learning*，固定 [arXiv:2501.12948v1](https://arxiv.org/abs/2501.12948v1)，即 2025 年 1 月首版技术报告。
- 原文入口：[HTML 全文](https://arxiv.org/html/2501.12948v1)、[PDF](https://arxiv.org/pdf/2501.12948v1)、[官方模型仓库](https://github.com/deepseek-ai/DeepSeek-R1)。
- 版本边界：不混入后续论文修订、R1-0528、社区复现或新 GRPO 变体的结果。官方仓库是补充资料，属于可变页面；本次访问日期为 2026-10-23。涉及实验数字，以固定 v1 为准。
- 状态：讲义已提供，全部待阅读；未下载权重、执行模型评测或启动训练。手算与资料核对不等于复现 R1，也不代表已掌握。
- 可视化入口：[R1 论文主线与阅读地图](overview.html)。

## 1. 先读什么？

先看[现代 LLM 训练流水线](../2026-10-22_01_llm_training_pipeline_and_capability_map.md)，再按下表阅读。本专题给出自己的解释和分析，不逐段翻译论文，也不只列原文链接。

| 顺序 | 精读章节 | 读完应能回答 |
|---|---|---|
| 1 | [研究问题、模型家族与贡献](2026-10-23_01_problem_and_contributions.md) | Zero、R1、Distill 分别证明什么？“无 SFT”不等于什么？ |
| 2 | [R1-Zero：奖励如何驱动推理](2026-10-23_02_r1_zero_and_rewards.md) | 没有参考解题过程，梯度从哪里来？Aha moment 能证明什么？ |
| 3 | [GRPO：从目标函数到手算](2026-10-23_03_grpo_objective_and_walkthrough.md) | 组、优势、old、ref、clip、KL 分别干什么？ |
| 4 | [R1 四阶段：权重流与数据流](2026-10-23_04_r1_pipeline_and_data.md) | 为什么 RL 后又做 SFT？哪个阶段重新从 Base 出发？ |
| 5 | [蒸馏：大模型探索、小模型学习](2026-10-23_05_distillation_and_transfer.md) | 学生学的是文本还是 logits？小模型靠 RL 和靠蒸馏的证据有何区别？ |
| 6 | [评测协议与结果精读](2026-10-23_06_evaluation_and_evidence.md) | pass@1、pass@k、cons@64 有何区别？哪些能力没有全面提升？ |
| 7 | [失败尝试、局限与可复现边界](2026-10-23_07_limits_and_reproducibility.md) | PRM / MCTS 为什么在作者实验中不理想？论文还缺什么证据？ |
| 8 | [精读练习与小规模验证设计](2026-10-23_08_reading_exercises_and_experiment_design.md) | 怎样用手算和受控实验验证理解，而不是复述宣传结论？ |

推荐两遍：

- **第一遍建立主线**：1 → 2 → 4 → 5 → 6。第 3 章先看角色表与四回答算例。
- **第二遍理解机制与证据**：完整读 3 → 7 → 8，并回到原文定位公式和表格。

练习可选，不主动测验，不替换当前 Agent 实践，也不恢复暂停中的 SFT 实验。

## 2. 三层证据标记

| 标记 | 含义 | 例子 |
|---|---|---|
| 论文事实 / 作者报告 | 指定版本明确描述或报告的内容，不代表本地复现 | 约 600k 推理样本、约 200k 非推理样本 |
| 机制解释 / 分析 | 根据目标函数和流程作出的解释，不冒充作者消融结论 | 结果奖励无法逐步定位错误；筛选会改变数据分布 |
| 教学示例 / 验证设计 | 为理解而构造的数据或尚未执行的实验 | 四个回答的奖励为 `[1,1,0,0]` |

原论文没有提供的精确数据占比、奖励系数、完整训练配置与成本，不用社区默认值补成“官方配方”。“本文未充分披露”也不等于断言全网永远没有相关材料。

## 3. 原文证据索引

本表用于快速回查；各章仍标明对应的原文章节。

| 要核对的主张 | v1 原文位置 | 本专题位置 |
|---|---|---|
| 从 V3-Base 出发，R1-Zero 不做前置 SFT | [§2.1–2.2](https://arxiv.org/html/2501.12948v1#S2.SS2) | 第 1、2 章 |
| GRPO、裁剪、KL、组相对优势 | [§2.2.1，式 1–3](https://arxiv.org/html/2501.12948v1#S2.SS2.SSS1) | 第 3 章 |
| 正确性与格式奖励；不用神经结果 / 过程 RM | [§2.2.2](https://arxiv.org/html/2501.12948v1#S2.SS2.SSS2) | 第 2 章 |
| 格式模板而非指定解题策略 | [§2.2.3，表 1](https://arxiv.org/html/2501.12948v1#S2.SS2.SSS3) | 第 2 章 |
| AIME 15.6% → 71.0%；投票 86.7%；长度与反思观察 | [§2.2.4，图 2–3、表 2–3](https://arxiv.org/html/2501.12948v1#S2.SS2.SSS4) | 第 2、6、7 章 |
| 数千条冷启动、可读性与人工后处理 | [§2.3.1](https://arxiv.org/html/2501.12948v1#S2.SS3.SSS1) | 第 4 章 |
| 推理 RL 与语言一致性取舍 | [§2.3.2](https://arxiv.org/html/2501.12948v1#S2.SS3.SSS2) | 第 4 章 |
| 600k + 200k；V3-Base 上 SFT 两个 epoch | [§2.3.3](https://arxiv.org/html/2501.12948v1#S2.SS3.SSS3) | 第 4、5 章 |
| 全场景 RL；helpfulness / harmlessness 评分范围 | [§2.3.4](https://arxiv.org/html/2501.12948v1#S2.SS3.SSS4) | 第 4 章 |
| 六个学生的初始化与仅做 SFT | [§2.4](https://arxiv.org/html/2501.12948v1#S2.SS4) | 第 5 章 |
| 采样与外部评测框架、表 4–5 | [§3](https://arxiv.org/html/2501.12948v1#S3) | 第 6 章 |
| 32B 蒸馏与直接 RL 对照 | [§4.1，表 6](https://arxiv.org/html/2501.12948v1#S4.SS1) | 第 5、7 章 |
| PRM、MCTS 的失败尝试 | [§4.2](https://arxiv.org/html/2501.12948v1#S4.SS2) | 第 7 章 |
| 通用任务、语言、提示与软件工程局限 | [§5](https://arxiv.org/html/2501.12948v1#S5) | 第 7 章 |

## 4. 与已有课程的关系

- [RL 基础](../2026-10-10_07_rl_basics_for_llm.md)、[PPO](../2026-10-10_08_ppo.md)、[GRPO 基础](../2026-10-11_01_grpo.md)：解释通用机制；本专题把机制放回 R1 的实验与数据条件。
- [DPO](../2026-10-10_09_dpo.md)：用于辨析离线偏好学习，不把 R1 四阶段中的 RL 擅自替换成 DPO。
- [蒸馏与合成数据](../2026-10-11_02_distillation_and_synthetic_data.md)：解释不同蒸馏形式；本专题核对 R1 实际使用的文本 SFT。
- [MoE](../2026-10-09_06_moe_architecture.md)：解释 671B 总参数与 37B 激活参数；架构不是这篇后训练报告的主要新贡献。
- [数据工程专题](../hw_data_engineering/README.md)：承接数据筛选、验证、血缘和分布变化。
- [Agent 评测](../2026-10-19_05_agent_evaluation_and_environments.md)：承接“外部执行器评分”与“模型自主工具交互”的边界。

返回：[每日索引](../topics_by_date.md) · [学习主索引](../../index-2026-10-08.md)。
