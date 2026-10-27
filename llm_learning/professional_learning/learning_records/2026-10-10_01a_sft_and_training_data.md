# SFT 数据图鉴：按 domain 看原始字段、Input 与 Output

- 课程位置：SFT 第 01 课之后、第 02 课 LoRA 之前；实际补写日期为 **2026-10-24**，不补记为 10 月 10 日已学习。
- 前置：[SFT 与训练数据](2026-10-10_01_sft_and_training_data.md)；后续：[LoRA / QLoRA](2026-10-10_02_lora_qlora.md)、[分布与混合采样](2026-10-10_03_data_distribution_and_sampling.md)。
- 可视化页面：[打开本篇图鉴](2026-10-10_01a_sft_and_training_data.html)。正文是页面的内容源；`flow`、`sequence` 和 `chart` 代码块在页面中渲染为图。
- 范围：原课 §7.4 的 **12 个具名数据集**。混合集的子来源只用于解释 domain，不递归扩展成另一份数据集大全。
- 状态：材料待阅读；未训练模型、执行样例工具或审计全量数据。

## A 先看地图：domain 与存储格式是两个维度

**domain 回答“学什么”，Input / Output 回答“给什么条件、模仿什么行为”。** 同一 domain 可以使用不同字段格式。

Input 表示推理时可见的条件。Output 表示拟学习的助手行为。训练时二者会共同进入 token 序列；目标回答通过 teacher forcing 提供前缀，不等于推理时允许偷看答案。

```flow LR
原始记录 -> 角色与字段适配
角色与字段适配 -> Input 条件
角色与字段适配 -> Output 示范
Input 条件 -> 模型模板与分词
Output 示范 -> 模型模板与分词
模型模板与分词 -> 监督目标与 loss mask
```

### A.1 十二个数据集，五种主要结构

下表的 domain 是阅读分类，不是一套跨数据集共享的官方标签。

| 数据集 | 主要 domain / 任务 | 原始核心结构 | Input → Output |
|---|---|---|---|
| OpenAssistant OASST1 | 通用助手、多语言、多轮 | 消息节点：`parent_id / role / text` | 祖先路径 → 某个 assistant 节点 |
| UltraChat 200k | 通用英文对话、写作、解释 | `prompt / prompt_id / messages` | 对话前缀 → assistant 回答 |
| No Robots | 10 类人工指令 | `category / messages / prompt` | 任务与材料 → 人工示范 |
| Aya Dataset | 多语言指令 | `inputs / targets / language_code` | inputs → targets |
| Orca-Math Word Problems 200k | 数学应用题 | `question / answer` | 题目 → 解题说明与答案 |
| Magicoder OSS-Instruct 75K | 编程、代码指令 | `problem / solution / seed / lang` | problem → solution |
| xLAM Function-Calling 60k | API / 函数调用 | 本次未核验原始文件；按 APIGen 公共文档解释 | 请求与可用 API → 调用名与参数 |
| Tulu 3 SFT Mixture | 通用、数学、代码、多语言、安全 | `id / messages / source` | 对话条件 → 对应来源的助手示范 |
| Dolci-Instruct-SFT | 多能力指令与工具 | `domain / source_dataset / messages` | 条件与工具定义 → 回答或调用 |
| OpenThoughts3-1.2M | 数学、代码、科学推理 | `domain / conversations / source / difficulty` | human 内容 → gpt 推理与回答 |
| Nemotron-Cascade-2-SFT-Data | 8 个 domain/config | `domain / source / generator / messages` | 按域的对话条件 → 回答或动作 |
| Nemotron-SFT-OpenCode-v1 | CLI 编程 Agent、技能、澄清 | `messages / tools` 与任务、环境元数据 | 已知状态 → 下一助手动作或答复 |

### A.2 别把四类标签混成一个 domain

| 维度 | 示例 | 意义 |
|---|---|---|
| 能力 / 任务 | 数学、摘要、代码、工具调用 | 期望强化的行为 |
| 来源 | Tulu 的 `source`、Dolci 的 `source_dataset` | 数据从哪里来，不必只含一个任务 |
| 语言 | Aya 的 `language_code`、OASST 的 `lang` | 跨任务切片；多语言不等于翻译任务 |
| 轨迹配置 | OpenCode 的 `agent_skills` split | 可用工具与上下文条件，不是业务行业 |

没有官方 domain 列的数据，需按任务人工或模型标注后统计。不能因为它叫通用数据，就认定每类均匀分布。

### A.3 示例与证据约定

- **字段事实**：来自发布方卡片、结构元数据或明确注明的少量记录检查。
- **示例内容**：下文问题、答案、代码和工具均为自拟，不复制原始长记录。原始字段示例只保留解释所需的字段，不是可直接导入的完整训练行。
- **转换示意**：自行设计的中间格式会明确标出，不冒充原始 schema。
- **分布数字**：仅绘制发布方明确列出的计数；未扫描全库。不用候选池规模、浏览器预览数量代替完整样本数。

英文数据的 JSON 示例保留英文；中文释义用于阅读，不表示原数据已经翻译。采样、去重和 loss 配置仍会改变实际训练曝光。

## B 通用指令：对话树、对话列表与任务类别

### B.1 OpenAssistant OASST1：一行是一个消息，不是完整问答

来源：[OASST1 数据卡](https://huggingface.co/datasets/OpenAssistant/oasst1)。

**domain**：多语言通用助手。可出现解释、写作、追问；`lang` 是语言，`labels` 是质量标注，并非统一任务分类。

发布格式包括扁平消息和嵌套消息树。扁平记录通过 `message_id / parent_id` 恢复路径；`prompter` 需映射为训练协议的 `user`。

自拟最小节点：

```json
[
  {"message_id":"u1","parent_id":null,"role":"prompter","lang":"en","text":"Explain evaporation in one sentence."},
  {"message_id":"a1","parent_id":"u1","role":"assistant","lang":"en","text":"Evaporation is the change of a liquid into gas at its surface."},
  {"message_id":"a2","parent_id":"u1","role":"assistant","lang":"en","text":"A liquid can turn into gas from its surface."}
]
```

```flow
用户节点 u1 -> 助手候选 a1
用户节点 u1 -> 助手候选 a2
助手候选 a1 -> 后续用户追问
后续用户追问 -> 后续助手回答
```

| domain 阅读切片 | Input | Output |
|---|---|---|
| 科学解释 | u1 的问题 | 选中的 a1 或 a2 |
| 多轮澄清 | u1、a1、后续追问 | 该路径的后续助手回答 |
| 目标语言指令 | 对应语言的祖先消息 | 同路径、符合语言要求的回答 |

`a1` 与 `a2` 是兄弟候选，不能串成两轮连续回答。质量、删除、排名信息用于筛选；不把它们直接写进助手答案。划分训练 / 测试时应按树或相关会话分组。

### B.2 UltraChat 200k：messages 已经是多轮顺序

来源：[UltraChat 数据卡](https://huggingface.co/datasets/HuggingFaceH4/ultrachat_200k)。

**domain**：广泛的英文合成对话；没有独立的官方 `domain` 列。下列解释 / 改写是教学切片，不是官方类别枚举。

```json
{
  "prompt_id":"demo-ultra-1",
  "prompt":"Explain a rainbow in one sentence.",
  "messages":[
    {"role":"user","content":"Explain a rainbow in one sentence."},
    {"role":"assistant","content":"A rainbow forms when sunlight is refracted, reflected, and dispersed by water droplets."},
    {"role":"user","content":"Make that explanation easier for a child."},
    {"role":"assistant","content":"Raindrops bend sunlight and spread it into colors."}
  ]
}
```

| 训练目标 | Input | Output |
|---|---|---|
| 首轮解释 | 第一个 user | 第一个 assistant |
| 按上下文改写 | user₁ + assistant₁ + user₂ | assistant₂ |
| 多轮 assistant-only | 完整因果序列 | 被选定的两个 assistant 回合 |

`prompt` 与首条消息可能重复，不能默认拼接两遍。SFT 入口是 `train_sft`；`test_sft` 保持独立。`train_gen / test_gen` 用途不同，不因字段相同就视为相同训练 split。

### B.3 No Robots：用官方 category 看任务差异

来源：[No Robots 数据卡](https://huggingface.co/datasets/HuggingFaceH4/no_robots)。

**domain**：10 类人工指令，主要单轮但包含多轮。字段为 `prompt / prompt_id / messages / category`。

下面每行都是**自拟输入输出**，计数来自数据卡全量 10,000 条口径，不是仅训练 split。

| 官方 category | 条数 | Input 示例 | Output 示例 / 形态 |
|---|---:|---|---|
| Generation | 4560 | 写一封延期一天的通知 | 一封完整短通知 |
| Open QA | 1240 | 为什么影子会改变方向？ | 不依赖给定材料的解释 |
| Brainstorm | 1120 | 想三个读书会名字 | 三个候选名称 |
| Chat | 850 | 用轻松语气介绍自己 | 角色一致的对话回答 |
| Rewrite | 660 | 把“马上给我”改得礼貌 | “方便时请发给我，谢谢。” |
| Summarize | 420 | 材料：周二开会，周三交稿；压成一句 | “周二开会，次日交稿。” |
| Coding | 350 | 写一个返回两数最大值的函数 | 函数代码 |
| Classify | 350 | 把“太好用了”分成正面或负面 | “正面” |
| Closed QA | 260 | 材料：活动周五举行；问哪天？ | “周五” |
| Extract | 190 | 从“订单 A17，金额 28 元”抽字段 | `{"order_id":"A17","amount":28}` |

```chart bar horizontal unit=条
Generation | 4560
Open QA | 1240
Brainstorm | 1120
Chat | 850
Rewrite | 660
Summarize | 420
Coding | 350
Classify | 350
Closed QA | 260
Extract | 190
```

自拟字段实例：

```json
{
  "prompt_id":"demo-nr-1",
  "category":"Extract",
  "prompt":"Extract the order ID and amount as JSON: order A17, amount 28.",
  "messages":[
    {"role":"user","content":"Extract the order ID and amount as JSON: order A17, amount 28."},
    {"role":"assistant","content":"{\"order_id\":\"A17\",\"amount\":28}"}
  ]
}
```

Output 在记录里是文本字符串；字符串内容可以是合法 JSON。不要把“外层样本 JSON”与“助手输出的 JSON”混为一层。

数据卡正文残留 `train_sft / test_sft` 的称呼，但当前配置声明为 **`train / test`**。使用时核对指定 revision 的实际 split。许可为 CC BY-NC 4.0，非商业限制不能被格式转换消除。

## C 多语言：Aya 的语言轴不等于任务轴

### C.1 Aya Dataset：inputs → targets

来源：[Aya 数据卡](https://huggingface.co/datasets/CohereLabs/aya_dataset)。

使用 `default` 配置的人工指令记录，不是 `demographics`，也不是 Aya Collection。

```json
{
  "inputs":"将“会议改到周五”改写得更礼貌。",
  "targets":"您好，会议时间调整至周五，感谢您的理解。",
  "language":"Simplified Chinese",
  "language_code":"zho",
  "annotation_type":"original_annotations"
}
```

文本为自拟；语言标签用于结构示范，具体枚举应核对所选记录。完整数据还包含 `user_id`，它不是回答目标。

```flow LR
inputs -> user 内容
targets -> assistant 内容
language_code -> 语言切片
annotation_type -> 来源与标注方式切片
user 内容 -> messages
assistant 内容 -> messages
```

### C.2 同一语言可以训练不同任务

下表是**教学分类**，不是 Aya 官方 task 标签表，也没有声称这些任务等量存在。

| 语言 × 任务切片 | Input | Output | 验证点 |
|---|---|---|---|
| 中文 × 改写 | 句子 + 语气要求 | 中文改写句 | 保留原意 |
| 西语 × 问答 | 西语问题 | 西语回答 | 事实与语言一致 |
| 英文 × 摘要 | 英文材料 + 长度限制 | 短摘要 | 信息支持与长度 |
| 跨语言 × 翻译 | 文本 + 明确目标语言 | 目标语言文本 | 语义保持 |

多语言数据不等于翻译数据；多数语言内任务可以完全不跨语言。

`annotation_type` 区分原始人工标注与人工修订。`demographics` 保存标注者背景，不应直接当成问答内容；不需要的个人标识也不放入模型输入。

数据卡说明测试集只覆盖部分语言。不能拿整个测试集的均值，替代每个目标语言的独立验收。

## D 数学与科学推理：答案文本有多长、包含什么？

### D.1 Orca-Math Word Problems 200k：question → answer

来源：[Orca-Math 数据卡](https://huggingface.co/datasets/microsoft/orca-math-word-problems-200k)。

**domain**：英文小学数学应用题；没有独立的 domain 字段。答案由 Azure GPT-4 Turbo 生成，不是因名为 `answer` 就自动为真。

```json
{
  "question":"Mia has 4 bags with 3 marbles each. She gives away 2 marbles. How many remain?",
  "answer":"There are 4 * 3 = 12 marbles. After giving away 2, 12 - 2 = 10 remain."
}
```

```flow LR
question 题目 -> user 输入
answer 解答文本 -> assistant 目标
assistant 目标 -> 推导与最终答案
```

| 任务切片（教学分类） | Input 条件 | Output |
|---|---|---|
| 数量运算 | 对象、数量和增减 | 运算说明与结果 |
| 比率问题 | 基数、比例与问题 | 比率推导与答案 |
| 多步应用题 | 多个关系与约束 | 连贯的分步解答 |

原始 `answer` 是一段文本，不是分别保存的 `reasoning` 和 `final_answer` 两列。若自己拆分，需验证解析规则与边界。

当前卡片只列 `train`，不能从“没有 test split”推出可以拿训练行作最终泛化评测。卡片还明确排除教育系统 / 机构使用；教程说明结构，不等于建议把数据投入教学产品。

### D.2 OpenThoughts3-1.2M：conversations 中包含长推理

来源：[固定 revision 61bcf9d 数据卡](https://huggingface.co/datasets/open-thoughts/OpenThoughts3-1.2M/blob/61bcf9d/README.md)。

字段为 `difficulty / source / domain / conversations`。后者使用 `from / value`，而不是直接使用 `role / content`。

自拟短例只说明形态，不代表真实长轨迹的长度或质量：

```json
{
  "domain":"math",
  "source":"demo-source-not-a-real-row",
  "difficulty":1,
  "conversations":[
    {"from":"human","value":"A box has 6 red and 4 blue pens. How many pens are there?"},
    {"from":"gpt","value":"<think>Add the two groups: 6 + 4 = 10.</think>\nThere are 10 pens."}
  ]
}
```

`difficulty: 1` 是占位，不赋予它未经核验的官方难度等级含义。

```flow LR
human 的 value -> user 的 content
gpt 的 value -> assistant 的 content
assistant 的 content -> 推理文本与最终回答
```

本次检查的公开首条记录确实使用 `human / gpt`，回答含 `<think>` 与 `</think>`。这只证实该行形态，不保证全库每行都有完整标记。

| 官方 domain | 发布方计数 | Input 示例（自拟） | Output 形态 |
|---|---:|---|---|
| 数学 | 850000 | 几何条件与待求量 | 推导、检查、最终结果 |
| 代码 | 250000 | 题面、约束、输入输出规格 | 算法分析与实现 |
| 科学 | 100000 | 物理 / 化学 / 生物问题 | 科学推理与答案 |

```chart bar horizontal unit=记录
数学 | 850000
代码 | 250000
科学 | 100000
```

这是发布方的 1.2M 记录配比，约为 70.83% / 20.83% / 8.33%，不是监督 token 占比。

卡片还说明对筛选后的问题重复生成回答，不能把 1.2M 全读成互不重复的题目。按问题来源和相关性划分，避免同题不同解答跨训练 / 测试。

长推理字段不宜直接截断；需检查最终答案、闭合标记和有效监督 token 是否保留。

## E 代码生成：Magicoder 的 seed 不是学生必须看到的输入

### E.1 Magicoder OSS-Instruct 75K

来源：[发布方数据卡](https://huggingface.co/datasets/ise-uiuc/Magicoder-OSS-Instruct-75K)、[官方项目](https://github.com/ise-uiuc/magicoder)、[结构元数据](https://datasets-server.huggingface.co/info?dataset=ise-uiuc%2FMagicoder-OSS-Instruct-75K&config=default)。

结构元数据指向 revision `5f839b1f368a76b161028bb9edff055db34022b2`。字段包括：

| 字段 | 角色 | 是否默认成为模型问答内容 |
|---|---|---|
| `problem` | 合成的编程需求 / 问题 | Input |
| `solution` | 合成的解答 | Output |
| `seed` | 启发合成的开源代码片段 | 不默认加入学生输入 |
| `lang` | 编程语言信息 | 分析 / 筛选；不是自然语言标签 |
| `index / raw_index` | 记录定位 | 不加入 |
| `openai_fingerprint` | 生成相关元信息 | 不加入 |

```flow
开源代码 seed -> 教师合成问题与解答
教师合成问题与解答 -> problem
教师合成问题与解答 -> solution
problem -> 学生 Input
solution -> 学生 Output
```

**生成数据时的输入，不等于训练学生时的输入。**

自拟精简记录：

```json
{
  "lang":"python",
  "seed":"def add(a, b):\n    return a + b",
  "problem":"Write a function total_cost(price, quantity) that returns their product.",
  "solution":"def total_cost(price, quantity):\n    return price * quantity"
}
```

### E.2 按任务切片看 Input / Output

下表是阅读和训练选型用的分类，不是该数据集公开的完整 domain 枚举。

| 代码任务 | Input 应提供 | Output 可以包含 | 不能混淆 |
|---|---|---|---|
| 从规格生成 | 功能、签名、约束 | 实现与解释 | 实现不是运行结果 |
| 修复 / 重写 | 有问题的代码与目标行为 | 修复后的实现 | 必须保留题面实际给出的代码 |
| 算法求解 | 输入输出规格、复杂度约束 | 算法分析与代码 | 示例通过不等于隐藏测试通过 |

数据没有因为叫“代码数据”就自动包含可执行测试列。测试与环境需另外确认；不要把自拟代码示例称为“已执行验证”。

本节没有运行示例代码。采用数据还需核对代码来源许可和生成服务条款。

## F 工具使用：函数调用样本与完整 Agent 轨迹不同

### F.1 xLAM Function-Calling 60k：学会选函数与填参数

来源：[原课数据集入口](https://huggingface.co/datasets/Salesforce/xlam-function-calling-60k)、[APIGen 官方项目说明](https://apigen-pipeline.github.io/)、[APIGen 论文 v1](https://arxiv.org/html/2406.18518v1)。

**本次访问限制**：HF 原始 README 返回 HTTP 401；未获取受限记录，也未代替用户接受访问条款。下面只画公开方法中的逻辑结构，不声明已经核验 HF 文件列名、嵌套类型或序列化方式。

**domain**：函数调用能力，跨 API 使用场景。公开文档报告 3,673 个 API、21 个类别；本篇不虚构未逐项核验的类别清单或样本配比。

```flow LR
用户请求 -> 调用规划模型
可用 API 定义 -> 调用规划模型
调用规划模型 -> 函数名与参数列表
函数名与参数列表 -> 外部执行器
```

**自拟、已适配的逻辑示例；不是原数据行：**

```json
{
  "input":{
    "request":"Count the characters in the text 'apple'.",
    "available_apis":[
      {"name":"text_length","parameters":{"text":{"type":"string"}}}
    ]
  },
  "output":[
    {"name":"text_length","parameters":{"text":"apple"}}
  ]
}
```

| 能力切片 | Input | Output |
|---|---|---|
| 选一个工具 | 请求 + 候选 API | 选中的函数名与参数 |
| 组织参数 | 请求中的实体、值、约束 | 符合 API 类型的参数 |
| 一次请求涉及多个调用 | 子请求 + API 集合 | 调用列表，按任务声明的依赖组织 |

示例的 Output 不是字符数量 `5`。调用后的执行结果属于另一个环节。

APIGen 的格式、执行、语义验证是数据生产流程；不能据此断言每条发布记录都包含完整执行反馈和多轮修复轨迹。新版本多轮数据也不自动等于这份 60k 数据。

### F.2 Nemotron-SFT-OpenCode-v1：下一动作依赖已经发生的交互

来源：[固定 revision 556d523 数据卡](https://huggingface.co/datasets/nvidia/Nemotron-SFT-OpenCode-v1/blob/556d523/README.md)。

**domain**：OpenCode CLI 场景的指令与工具操作。卡片列出 6 个 split，它们更接近环境配置，而非互斥业务 domain。

| split | Input 增加 / 改变什么 | Output 训练重点 |
|---|---|---|
| `general` | 普通问题；含有 / 没有 AGENTS.md 的成对设置 | 常规回答与代码任务行为 |
| `bash_only_tool` | 主要通过 bash、todo 工具工作 | 用允许的工具完成操作 |
| `agent_skills` | 候选技能及按需加载信息 | 选技能、加载后行动 |
| `bash_only_tool_skills` | bash、todo、skill 的受限组合 | 通过技能理解可用命令 |
| `question_tool` | 可以向用户提问并获取反馈 | 澄清之后再行动 |
| `agent_skills_question_tool` | 技能与提问能力同时存在 | 澄清、加载、执行的组合 |

本次对 `general` 与 `agent_skills_question_tool` 各检查首条记录结构；其他 split 的说明依据卡片，未逐条验证。

| 已观察字段 | 结构与作用 |
|---|---|
| `question_category / complexity_level / question` | 任务类别、复杂度与问题；首条见 `fact_seeking_qa` 或 `code_debugging` |
| `agent_prompt` | 辅助文本字段；不因名字含 prompt 就额外拼入输入，应以实际消息和训练适配规则为准 |
| `messages` | `system / user / assistant / tool` 消息；有的样本仅有三条消息 |
| `tools` | 工具对象含 `id / description / inputSchema.jsonSchema` |
| `messages[].tool_calls` | 助手调用含 `id / type / function`；`function.arguments` 为 JSON 字符串 |
| tool 消息的 `content` | 在所查轨迹中是数组，元素含 `type / toolCallId / toolName / output` |
| `uuid / metadata / enabled_tools / skills_path / hf_split` | 追踪与环境信息；不能全当成回答文本 |

所查记录同时有 `tools` 内容与空 `enabled_tools`，不能直接把空数组解释为“没有工具”。

**观察到的消息结构 + 自拟内容，省略其他字段：**

```json
{
  "messages":[
    {"role":"user","content":"Read notes.txt and report the number of lines."},
    {"role":"assistant","content":null,"tool_calls":[
      {"id":"call-demo-1","type":"function","function":{"name":"read","arguments":"{\"path\":\"notes.txt\"}"}}
    ]},
    {"role":"tool","content":[
      {"type":"tool-result","toolCallId":"call-demo-1","toolName":"read","output":{"type":"text","value":"one\ntwo"}}
    ]},
    {"role":"assistant","content":"The file has 2 lines."}
  ]
}
```

这里的 `read` 工具名与参数是教学设计；真实训练需同时提供对应工具定义，不能只靠该片段运行。

```sequence
User -> Assistant: request
Assistant -> Tool: call(name, arguments)
Tool --> Assistant: result
Assistant -> User: answer
```

| 拆成两个监督时刻 | Input | Output |
|---|---|---|
| 调工具前 | system、工具定义、user 与此前历史 | 助手调用，包括函数名与参数 |
| 调工具后 | 上述历史 + 实际工具结果 | 助手最终回答或下一调用 |

在 assistant-only 设置中，工具结果是条件，不是助手要伪造的目标。不能把未来执行结果提前放入第一次调用的 Input。

该数据的 tool `content` 可为数组，assistant `content` 可为 null。把全部消息强行转成一个字符串，会丢失调用 ID 和结构。迁移到目标模型时需显式转换，不是假定原样符合 OpenAI 或任意训练库协议。

## G 多能力混合：source、domain 与工具字段要保留

### G.1 Tulu 3 SFT Mixture：按来源组织，不自带统一 domain 列

来源：[Tulu 3 数据卡](https://huggingface.co/datasets/allenai/tulu-3-sft-mixture)。

原始字段：`id / messages / source`。`messages` 内为 `role / content`。

**自拟示例；source 值是教学占位，不是官方枚举：**

```json
{
  "id":"demo-tulu-1",
  "source":"demo-math-source",
  "messages":[
    {"role":"user","content":"What is 7 multiplied by 8?"},
    {"role":"assistant","content":"7 multiplied by 8 is 56."}
  ]
}
```

| domain 阅读分类 | 卡片列出的来源例子 | Input → Output |
|---|---|---|
| 通用指令 / 对话 | FLAN v2、No Robots、OASST、WildChat | 指令或历史 → 回答 |
| 数学 | Persona MATH / GSM / Algebra、NuminaMath-TIR | 题目与约束 → 解答 |
| 编程 | Persona Python、Evol CodeAlpaca | 需求与代码上下文 → 实现 |
| 指令遵循 | Persona IF | 指令与格式限制 → 合规回答 |
| 多语言 | Aya | 目标语言输入 → 对应语言输出 |
| 安全与拒答 | CoCoNot、WildGuardMix、WildJailbreak | 请求及边界 → 合适回答或拒绝 |
| 表格 / 科学任务 | TableGPT、SciRIFF | 表格或科学文本与任务 → 抽取、分析或回答 |

这是来源到能力的阅读映射，不代表每个来源只含一种任务。卡片总数与元数据存在一条记录的差异，本篇不据此绘制精确来源份额图。

### G.2 Dolci-Instruct-SFT：同时有 domain 和 source_dataset

来源：[固定 revision bd3c8f3 数据卡](https://huggingface.co/datasets/allenai/Dolci-Instruct-SFT/blob/bd3c8f3/README.md)。

原始字段：`id / messages / source_dataset / domain`。每条 message 声明 `role / content / functions / function_calls`。

后两个字段在 schema 中是**字符串**，普通问答中可以为 null；不能自动视为已解析的调用对象。

```json
{
  "id":"demo-dolci-1",
  "domain":"Math",
  "source_dataset":"Tulu 3 Persona MATH",
  "messages":[
    {"role":"user","content":"Compute 9 + 6.","functions":null,"function_calls":null},
    {"role":"assistant","content":"9 + 6 = 15.","functions":null,"function_calls":null}
  ]
}
```

`Math` 与 `Tulu 3 Persona MATH` 在本次所查首行可见；题目、答案和 ID 为自拟。其余 domain 不假造精确字段取值。

| domain 阅读分类 | Input | Output | 额外需要 |
|---|---|---|---|
| 数学 / 逻辑 / 可验证推理 | 条件、问题、约束 | 解答与结果 | 答案验证 |
| 代码 / Python 算法 | 需求、输入输出规格 | 代码或解释 | 独立测试 |
| 通用对话 / 事实任务 | 问题与必要材料 | 面向用户的答复 | 来源与事实支持 |
| 多语言 | 指定语言的任务 | 对应语言的回答 | 分语言评测 |
| 精确指令遵循 | 限制与任务内容 | 满足限制的回答 | 结构与内容分开检查 |
| 安全 | 请求及可回答边界 | 适当响应 | 同时检查误拒 |
| 工具使用 | 对话与函数定义 | 函数调用或后续回答 | 保留 functions / function_calls |

**迁移时的两个不同来源：**

```flow LR
domain -> 能力分桶
source_dataset -> 来源追踪与去重
messages -> 模型可见序列与监督目标
```

卡片说明部分 OpenThoughts 来源在 Instruct 配方中移除了推理轨迹。相同来源不保证相同输出格式，不能把原始 OpenThoughts 的长推理直接当成 Dolci 每条样本的形态。

### G.3 混合数据最容易发生的输入错误

- 不把 `source`、生成模型名或质量分自动添加到 prompt。
- 不把工具定义和调用参数字段丢掉，只保留 `content`。
- 不把混合集与其子来源再全量拼接，忽略重复曝光。
- 不用样本数推算监督 token；同样一条记录可以是一句回答或一段长轨迹。

集合的许可不替代各来源许可；格式统一也不说明子来源都可商用。

## H Nemotron-Cascade-2：8 个 domain 的输入输出与核对边界

### H.1 相同外层 messages，不同内层任务

来源：[固定 revision 9f36020 数据卡](https://huggingface.co/datasets/nvidia/Nemotron-Cascade-2-SFT-Data/blob/9f36020/README.md)。

卡片按 8 个 config 提供数据。本次查询了对应的公开结构元数据，均声明：

```json
{
  "domain":"demo-domain-value",
  "source":"demo-source",
  "generator":"demo-generator",
  "messages":[
    {"role":"user","content":"Explain why a metal spoon feels cold."},
    {"role":"assistant","content":"Metal transfers heat from your hand quickly, so it feels cold."}
  ]
}
```

这是**结构示例**，三个元数据值为占位；不声称它们是实际枚举。结构接口来自当前服务，不等于固定 revision 的所有行均逐一审核。

### H.2 按 8 个 domain 展开数据内容

表内任务内容是依据卡片能力范围构造的教学示例，没有执行任何动作。

| domain/config | Input：模型需要看到什么 | Output：拟模仿什么 |
|---|---|---|
| `math` | 数学题、已知条件、求值或证明要求 | 解答、推导或证明 |
| `science` | 物理、化学、生物问题与条件 | 科学解释和推理答案 |
| `chat` | 用户请求与对话历史 | 对话答复、写作或解释 |
| `instruction_following` | 任务 + 字数、格式、数量等限制 | 同时满足内容与限制的回答 |
| `safety` | 用户请求与允许回答边界 | 合适答复、拒绝或安全替代 |
| `conversational_agent` | 请求、工具定义、历史观察 | 下一工具动作或用户答复 |
| `swe` | Issue、代码上下文或仓库观察 | 定位、修复、测试生成或 Agent 动作 |
| `terminal_agent` | 终端任务、可用工具和先前输出 | 命令 / 工具动作与后续答复 |

`swe` 同时包含 agentless 与 agentic 数据，不能把每条 SWE 记录都解释成完整工具循环。

### H.3 用同一目标对比三种 Output

**教学目标：确认一个程序为何返回错误结果。**

| 数据形态 | Input | Output |
|---|---|---|
| 普通代码问答 | 问题 + 已给出的函数 | 解释错误原因或给修复代码 |
| Agentless SWE | Issue + 定位阶段需要的代码材料 | 当前阶段的定位 / 补丁 / 测试 |
| Agent 轨迹 | 任务 + 当前已知仓库状态 + 工具 | 下一动作；执行结果再进入下一步 Input |

```flow
任务与已有状态 -> 助手动作
助手动作 -> 环境执行
环境执行 -> 新观察
新观察 -> 下一步条件
下一步条件 -> 后续助手动作
```

虽然外层字段都叫 `messages`，工具协议可能保存在 `content` 文本里。结构元数据没有证明存在标准 `tool_calls` 列；转换时不能凭空补出“官方调用字段”。

### H.4 哪些数字不应该画成分布？

本次大部分 Cascade config 的结构接口标记 `partial: true`。其中的 `num_examples` 只是服务处理到的部分，不是完整 domain 规模。

卡片也有各域样本计数，但本篇仅用它确认领域列表，不将未经全量核对的计数重新包装为独立统计。缺少可靠口径时，内容表比看似精确的饼图更合适。

### H.5 证据与检查范围

| 资料 | 本次采用的证据 | 限制 |
|---|---|---|
| OASST1、UltraChat、No Robots、Aya、Orca-Math、Tulu | 发布方当前卡片的字段 / split / 分类 | main 可变化，查阅日期为 2026-10-24 |
| Magicoder | 卡片、官方项目、结构元数据中的 revision | 不据此保证每条代码通过测试 |
| Dolci、OpenThoughts3 | 原课固定卡片版本 + 各一条当前 viewer 记录结构 | 单行检查不是分布或质量审计 |
| Cascade-2 | 原课固定卡片版本 + 8 个 config 的结构元数据 | 服务部分处理结果不能当全量 |
| OpenCode-v1 | 固定卡片 + 两个 split 的首行结构 | HTTP Range 限量读取，未下载完整 JSONL |
| xLAM 60k | APIGen 官方公开说明与论文 | HF 401；仅给逻辑映射，原始字段待有权访问后核验 |

补充结构入口：

- [Cascade math 元数据](https://datasets-server.huggingface.co/info?dataset=nvidia%2FNemotron-Cascade-2-SFT-Data&config=math)：其余 config 使用上表的精确名称。
- [OpenCode general 文件入口](https://huggingface.co/datasets/nvidia/Nemotron-SFT-OpenCode-v1/blob/556d523/general/data.jsonl)、[技能与澄清文件入口](https://huggingface.co/datasets/nvidia/Nemotron-SFT-OpenCode-v1/blob/556d523/agent_skills_question_tool/data.jsonl)。
- [OpenThoughts 单行结构入口](https://datasets-server.huggingface.co/rows?dataset=open-thoughts%2FOpenThoughts3-1.2M&config=default&split=train&offset=0&length=1)、[Dolci 单行结构入口](https://datasets-server.huggingface.co/rows?dataset=allenai%2FDolci-Instruct-SFT&config=default&split=train&offset=0&length=1)。

数据可用于什么、怎样格式化、质量是否达标，是三项独立检查。本次完成的是**内容与格式图解**，没有接受访问条款、执行工具、启动训练或证明模型收益。

返回：[SFT 主课](2026-10-10_01_sft_and_training_data.md) · [每日索引](topics_by_date.md)。
