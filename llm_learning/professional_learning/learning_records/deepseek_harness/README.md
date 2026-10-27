# DeepSeek Harness 专题：7 站最小学习路径与扩展

**从这里顺序读即可，不必同时打开原始长文。** 本专题把原路径上的 12 篇文档重新组织为 7 篇教材，提炼概念、设计问题、主流程和例子。原文放在每篇末尾，用于核对和继续下钻，不是额外必读任务。

**进度（2026-10-19）**：用户确认第一遍学习已走完，本专题暂作查阅。主干转入[安全、权限与人工介入](../2026-10-19_01_agent_permissions_and_human_approval.md)，不要求继续扩读其他项目；第一遍学习完成不代表已运行 Harness 或通过实践验收。

## 阅读顺序

| 站点 | 教材 | 读完获得什么 |
|---|---|---|
| 1 | [系统全貌与插件架构](2026-10-17_01_system_and_plugins.md) | 职责划分、装配关系、服务 / 事件 / effect 的区别 |
| 2 | [消息、事件与请求](2026-10-17_02_messages_events_requests.md) | 一次工具往返的数据变化；记录、模型输入与传输格式的区别 |
| 2a（可选补充） | [Skills 与渐进式披露](2026-10-17_02a_skills_and_progressive_disclosure.md) | registry / provider / consumer、目录与正文进入 payload 的时机、Office 分层加载 |
| 3 | [执行循环与生命周期](2026-10-17_03_loop_and_lifecycle.md) | 输入接纳、turn / step / attempt、创建和释放的所有权 |
| 4 | [工具执行与权限边界](2026-10-17_04_tools_and_policy.md) | 注册、参数、授权、执行、结果之间的完整关系 |
| 5 | [持久化与恢复](2026-10-17_05_persistence_and_recovery.md) | append / flush 的保证、单写者、中断与未知副作用 |
| 6 | [异常、预算与资源控制](2026-10-17_06_failures_budgets_resources.md) | 失败决策表、预算口径、取消与释放、过载保护 |
| 7 | [从设计到实现与验收](2026-10-17_07_implementation_and_acceptance.md) | 给 Coding Agent 的任务卡、增量交付与证据门槛 |

### 扩展阅读：不增加最小 MVP 的必做范围

| 章节 | 教材 | 读完获得什么 |
|---|---|---|
| 8（可选） | [Multi-Agent 的实例、会话与协作管理](2026-10-18_08_multi_agent.md) | 角色与实例、服务启动与会话创建、Team 配置、父子共用机制、委派结算与恢复 |

第 8 章建议在第 1、3 站之后按需阅读；补充子 Agent / Team 文档与源码，不属于原先 12 篇文档的七站最小路径。

统一场景：**用户询问一个受控目录中的文件，Agent 调用只读工具，再给出有依据的回答。** 其他场景只用于解释崩溃或副作用，不表示 MVP 开放写入、shell 或外部服务。

统一术语：**Agent 是接口；ReactLoopAgent 是默认执行实例的类；Driver 在生命周期讨论中指实例内部的一次异步执行活动，不是独立类实例；AgentLoop 是创建、恢复和管理实例的工厂与生命周期服务。** 同一实例可先后启动多次 Driver 活动，一次活动也可包含多个 turn，不能把 Driver 直接等同于 Agent 实例。详见[第 1 站](2026-10-17_01_system_and_plugins.md)。

每站按四个问题组织；介绍新的机制 / 模块时，也沿用这一分析顺序，不只列术语和操作步骤：

| 问题 | 需要讲清什么 | 不要混成什么 |
|---|---|---|
| 是什么 | 定义、职责、输入输出和边界 | 一上来就列实现步骤 |
| 为什么要做 | 需求、失败场景；不做会怎样 | “别人也这么做” |
| 怎么做 | 数据、主流程、分支和直观例子 | 只有抽象口号 |
| 为什么这么做 | 选择理由、替代方案、收益与代价 | 再重复一次功能目的 |

最后连接 MVP 与验收，原文 / 源码放在折叠区。小机制可以用四项 bullet 简述，不强行扩成四段长文；不主动测验。

## 如何理解“设计思路”

- **原文明确**：项目文档直接说明的规则、目的或失败场景，提供原文链接。
- **分析**：根据机制解释收益、代价和替代方案，不代替作者表态。
- **MVP 建议**：我们拟定的实现要求，不冒充 DeepSeek 已有能力。
- **教学示例**：为说明机制自拟，不是模型运行记录或测试结果；省略字段会明确说明。

阅读版本固定为 `571902028a75c64c1d767f4ae5fd099485f0ddf0`。本专题基于本地文档和关键源码，**没有运行 Harness、模型请求或行为测试**。文档之间有冲突时不混合叙述：第 3、6 站专门说明请求重试的文档差异及源码依据。

## 原始文档到教材的映射

| 原路径文档 | 在本专题怎样使用 |
|---|---|
| [架构](../../projects/deepseek-harness/docs/architecture.zh.md) | 第 1 站职责与装配；第 2、3、5 站数据与执行原则 |
| [Cordis 入门](../../projects/deepseek-harness/docs/cordis-primer.zh.md) | 第 1 站框架最小概念 |
| [LLM 流式输出](../../projects/deepseek-harness/docs/subsystems/llm-streaming.zh.md) | 第 2 站消息 / 请求 / 流；第 6 站错误与 usage |
| [Session](../../projects/deepseek-harness/docs/subsystems/session.zh.md) | 第 2 站记录与投影；第 5 站恢复 |
| [核心](../../projects/deepseek-harness/docs/subsystems/core.zh.md) | 第 3 站输入、执行和所有权 |
| [生命周期图](../../projects/deepseek-harness/docs/agent-lifecycle.zh.md) | 第 3 站顺序与分支；第 6 站重试 |
| [工具](../../projects/deepseek-harness/docs/subsystems/tools.zh.md) | 第 4 站定义、执行数据和策略 |
| [工具流水线](../../projects/deepseek-harness/docs/tool-execution-pipeline.zh.md) | 第 4 站执行 / 拒绝 / 错误路径 |
| [持久化](../../projects/deepseek-harness/docs/subsystems/persistence.zh.md) | 第 5 站保存点、写入所有权和崩溃处理 |
| [通用防御性模式](../../projects/deepseek-harness/docs/defensive-patterns/general.zh.md) | 第 6 站结果报告、并发与资源规则 |
| [工具编写参考](../../projects/deepseek-harness/docs/cookbook/adding-a-tool.zh.md) | 第 7 站工具任务卡与实现要求 |
| [测试策略](../../projects/deepseek-harness/docs/testing.zh.md) | 第 7 站证据层次和验收方法 |

第 6 站另补充工具超时、重复调用提醒的包文档，避免把局部机制误读成全任务预算。本文是重新编写的学习材料，不是原文镜像；原项目版权及许可见 [DeepSeek MIT License](../../projects/deepseek-harness/LICENSE)。

## 范围与其他入口

- **当前必需**：单 Agent 闭环、只读授权、错误 / 取消、事件与基础恢复、预算和独立验收。
- **后置**：UI、桌面、MCP、PTC、多 Agent、复杂压缩、完整迁移与分布式部署。多 Agent 已补[独立学习章](2026-10-18_08_multi_agent.md)，仍不是单 Agent MVP 的必做实现。
- [官方 docs 层级与选读章节](../2026-10-17_04_deepseek_harness_docs_mvp_reading_path.md)：追原文时使用。
- [模块与跨模块设计索引](../2026-10-17_01_deepseek_harness_architecture_index.md)：按 M0～M6、X1～X8 查问题。
- [Coding Agent 实施方法](../2026-10-17_02_harness_architect_to_coding_agent.md)与[统一验收清单](../2026-10-17_03_harness_acceptance_checklist.md)：保留一个权威任务 / 验收入口，不复制出第二套编号。

文件名中的 01～08 是本专题内部顺序：01～07 为最小路径，08 为扩展章；2a 是第 2 站之后的可选补充。原有文件日期记录首次编写时间；后补章节可沿用相邻章节日期和字母插入编号，以保持排序，正文另记实际补写日期（2a 为 2026-10-19）。旧教材和原始文档保留不动；以后此路径的答疑优先补入对应章节。
