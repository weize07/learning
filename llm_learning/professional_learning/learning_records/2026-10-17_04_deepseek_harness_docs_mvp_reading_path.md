# DeepSeek Harness：文档层级与最小 MVP 学习路径

> **默认阅读入口已整理到[DeepSeek Harness 专题目录](deepseek_harness/README.md)**：7 篇精炼教材可顺序直接阅读，包含概念、设计问题、主流程和例子。本文保留为原始文档分类与章节索引，不必与专题重复通读。

> 固定版本：`571902028a75c64c1d767f4ae5fd099485f0ddf0`。本轮盘点[本地 docs](../projects/deepseek-harness/docs/)，核对入口、章节和文档之间的说明；没有通读全部文档，也没有执行其中的命令。
>
> 配套：[架构与设计主题索引](2026-10-17_01_deepseek_harness_architecture_index.md) · [Coding Agent 实施方法](2026-10-17_02_harness_architect_to_coding_agent.md) · [验收清单](2026-10-17_03_harness_acceptance_checklist.md)。

## 1. 先给结论

**不要按文件名或目录顺序通读。** 这里混合了概念说明、流程图、类型参考、开发教程、生成目录和版本记录，不是一套从头到尾的课程。

最小路径选 **12 篇文档的指定章节，分 7 站阅读**：

**全貌与 Cordis → 消息与事件 → Agent 执行 → 工具执行 → 保存与恢复 → 异常和预算边界 → 实现与验收。**

- 学习目标：能设计“单 Agent、一个模型接口、一个只读工具、事件记录与基础恢复、取消 / 错误 / 预算、可验证交付”的小闭环。
- 不以“读完所有类和事件”为目标；能写出模块职责、主流程和失败规则就停止下钻。
- MVP 是学习与设计范围，不表示开始实现。Web、桌面、MCP、多 Agent、PTC、动态扩展和完整升级迁移暂不进入主线。
- 链接优先用中文对侧；字段、事件及含义有疑问时，通过页面顶部切回英文，再看该页给出的源码。

## 2. 文档的层级与先后关系

下面是按用途整理的**学习层级**，不是项目声明的严格依赖树。

| 层级 | 回答什么 | 代表入口 | 与其他文档的关系 |
|---|---|---|---|
| L0 产品使用 | 怎么启动、配置和使用？ | [用户指南](../projects/deepseek-harness/docs/user/guide/index.zh.md)、[CLI 帮助](../projects/deepseek-harness/docs/cli-help.zh.md) | 操作入口；不是理解架构的全部前置 |
| L1 框架前置 | ctx、插件、事件、effect 是什么？ | [Cordis 入门](../projects/deepseek-harness/docs/cordis-primer.zh.md) | 读核心服务和事件前掌握；不熟时才补教程 |
| L2 系统架构 | 谁负责什么？怎样装配和协作？ | [架构](../projects/deepseek-harness/docs/architecture.zh.md)、[文档图索引](../projects/deepseek-harness/docs/graph-atlas.zh.md) | 建立地图，再选流程；图索引只是导航，不要求遍历全部图 |
| L3 数据与主流程 | 传什么数据？谁推动下一步？ | [消息与流](../projects/deepseek-harness/docs/subsystems/llm-streaming.zh.md)、[Session](../projects/deepseek-harness/docs/subsystems/session.zh.md)、[生命周期图](../projects/deepseek-harness/docs/agent-lifecycle.zh.md)、[工具流水线](../projects/deepseek-harness/docs/tool-execution-pipeline.zh.md) | 先看关键数据，再沿正常 / 异常路径走一遍 |
| L4 子系统约定 | 某一模块的精确行为和接口是什么？ | [子系统索引](../projects/deepseek-harness/docs/subsystems/README.zh.md)、[Cordis API](../projects/deepseek-harness/docs/cordis-api/context.zh.md) | 根据 L3 的具体问题查阅；不是接着把全部接口读完 |
| L5 实现与验证 | 新能力加在哪里？怎样判断实现可靠？ | [扩展手册](../projects/deepseek-harness/docs/cookbook/extension-cookbook.zh.md)、[防御性模式](../projects/deepseek-harness/docs/defensive-patterns/README.zh.md)、[测试策略](../projects/deepseek-harness/docs/testing.zh.md) | 实现前选扩展点和验收方式；测试要求也反过来检查设计 |
| L6 查表与维护 | 精确配置、事件、版本变化在哪里？ | [配置目录](../projects/deepseek-harness/docs/config-catalog.zh.md)、[事件矩阵](../projects/deepseek-harness/docs/event-producer-consumer.zh.md)、[格式状态](../projects/deepseek-harness/docs/session-format-status.zh.md) | 遇到具体配置、签名或兼容问题才打开，不是必读的“最后一大章” |

**最重要的四组关系：**

- **Cordis 入门 → 核心 / 工具子系统**
  - 先理解服务调用、serial、waterfall 和资源释放，才能判断事件监听的顺序和作用。
  - [完整教程](../projects/deepseek-harness/docs/cordis-tutorial/index.zh.md)是练习路径，[Cordis API](../projects/deepseek-harness/docs/cordis-api/context.zh.md)是查表路径；两者都不是入门页之后必须通读的续集。
- **总架构 → 主流程图 ↔ 子系统说明**
  - 架构给职责，流程图给跨模块顺序，子系统给字段与精确行为。
  - 图上遇到不懂的接口才下钻；理解后回到主流程，不沿每个链接继续扩散。
- **消息 / Session → 请求 / 执行 → 持久化**
  - 先分清 Message、SessionEvent、请求和视图，再看它们怎样保存 / 恢复。
  - 事件记录不等于模型历史，Session 内存 append 不等于磁盘 flush。
- **工具执行约定 → 添加工具 → 测试与防御规则**
  - 先知道参数、结果、策略和取消要求，再看注册一个工具的代码。
  - 快速教程演示“能接上”；参考文档与测试要求说明“接上后要遵守什么”。

## 3. 目录分类：哪些优先，哪些后置

固定检出中共有 **473 个 Markdown 文件：234 个中文对侧、239 个非 `.zh.md` 文件**。后者也包含索引和维护文件，不是 239 篇必读教程。表中数量按非 `.zh.md` 文件计；不会重复计算中英文阅读任务。

优先级：**P0 = 下节选中的必读章节；P1 = 具体能力需要时读；P2 = 查表 / 维护时读。** 目录里的文件不自动全部继承 P0。

| 目录 / 分组 | 数量 | 内容与阅读位置 | 优先级 |
|---|---:|---|---|
| [docs 根目录](../projects/deepseek-harness/docs/) | 23 | 架构、Cordis 入门、流程图是主入口；其余包含图目录、API / 配置目录及工程说明 | P0 / P2 |
| [subsystems](../projects/deepseek-harness/docs/subsystems/README.zh.md) | 67 | 每个子系统的说明、类型与生成 API；只选闭环所需的几篇 | P0 / P1 / P2 |
| [cordis-tutorial](../projects/deepseek-harness/docs/cordis-tutorial/index.zh.md) | 8 | 索引 + 7 章动手练习，依次是插件、生命周期、服务、事件、配置、组合、接入 Harness | P1 |
| [cordis-api](../projects/deepseek-harness/docs/cordis-api/) | 6 | Context、事件、fiber、registry、service 等框架参考 | P2 |
| [user](../projects/deepseek-harness/docs/user/) | 19 | guide 是使用专题；develop/basic → framework → practice 是插件开发路径 | P1 |
| [cookbook](../projects/deepseek-harness/docs/cookbook/) | 12 | 加工具、适配器、包、远程 API、Session 格式及维护流程 | 选工具篇 P0，其余 P1 / P2 |
| [defensive-patterns](../projects/deepseek-harness/docs/defensive-patterns/README.zh.md) | 6 | 通用规则与包、远程、客户端、会话等改动的审查要求 | 选通用规则 P0，其余按改动面读 |
| [persistence-changes](../projects/deepseek-harness/docs/persistence-changes/) | 48 | 格式变化、历史格式和发行记录 | P2；先读当前语义，再查历史 |
| [upgrade-guide](../projects/deepseek-harness/docs/upgrade-guide/) | 40 | 升级时的行为变化与迁移指导 | P2；要升级时升为必读 |
| [postmortem](../projects/deepseek-harness/docs/postmortem/) | 5 | 故障案例及经验 | P1；理解对应机制后再复盘 |
| [i18n](../projects/deepseek-harness/docs/i18n/) | 5 | 翻译、术语和文档维护 | P2 |

根目录容易混淆的文件再分三类：

- **结构导航**：[模块依赖图](../projects/deepseek-harness/docs/module-graph.zh.md)、[能力接口图](../projects/deepseek-harness/docs/capability-seams.zh.md)。用于找关系；大量节点不等于阅读顺序。
- **精确查表**：[工具目录](../projects/deepseek-harness/docs/tool-catalog.zh.md)、[配置目录](../projects/deepseek-harness/docs/config-catalog.zh.md)、[事件矩阵](../projects/deepseek-harness/docs/event-producer-consumer.zh.md)、[持久化目录](../projects/deepseek-harness/docs/persistence-catalog.zh.md)、[实验性持久化目录](../projects/deepseek-harness/docs/experimental-persistence-catalog.zh.md)、[术语表](../projects/deepseek-harness/docs/glossary.zh.md)。
- **特定工程面**：[开发环境](../projects/deepseek-harness/docs/development.zh.md)、[API Gateway](../projects/deepseek-harness/docs/api-gateway.zh.md)、[DeepSeek 请求扩展](../projects/deepseek-harness/docs/deepseek-llm-api-wire-extensions.zh.md)、[依赖重命名](../projects/deepseek-harness/docs/rescope.zh.md)、[Web 样式](../projects/deepseek-harness/docs/web-styling.zh.md)、[圆角规范](../projects/deepseek-harness/docs/ui-radius.zh.md)。安装、接入或修改对应能力时再读。

`.i18n.yaml`、schema / catalog JSON 和图片不是新的阅读章节。[docs/AGENTS.md](../projects/deepseek-harness/docs/AGENTS.md)用于维护上游文档；本次只在学习记录中建立导航，不改上游文档结构。

## 4. 最小路径：12 篇选读，7 站完成

“完成产物”是读完后的自检标准，不要求现在写作业。已理解的站可以直接跳过；不必因为换成官方文档再读一遍同样的内容。

### 第 1 站：系统全貌与最少框架知识

- **01 [架构](../projects/deepseek-harness/docs/architecture.zh.md)**
  - 首读：`Profile 与组合包`、`应用启动`、`核心包`；先确认系统由谁组成。
  - 读完下一篇，再回看：`事件`、`轮次流程`、`会话日志`、`新行为的归属位置`。
  - 跳过：桌面应用细节、复杂 prompt 更新和迁移分支；先不追生成目录。
- **02 [Cordis 入门](../projects/deepseek-harness/docs/cordis-primer.zh.md)**
  - 读：`五个核心概念`、`分发模式`、`Cordis Waterfall 语义`、`实践规则`。
  - 暂跳：Loader 表达式细节。只要能区分服务方法、事件和 effect，就不展开完整教程。
- **完成产物**：一张“入口、Agent、LLM、Tools、Session”职责图，能说明插件与 Driver 的区别。

### 第 2 站：先看数据，再看执行

- **03 [LLM 流式输出](../projects/deepseek-harness/docs/subsystems/llm-streaming.zh.md)**
  - 首读：`内容块与消息`、`模型请求`、`StreamChunk：原始协议`、`适配器约定`。
  - 重点只看：文本、tool-call、tool result、callId、请求配置、流的完成 / 错误；图片、签名 replay、官方扩展暂跳。
- **04 [Session](../projects/deepseek-harness/docs/subsystems/session.zh.md)**
  - 读：`SessionEvent<T>：一条日志条目`、`派生历史`、`轮次的结束原因`、`持久性约定`。
  - 在 `SessionEventMap` 里只查 user / assistant message、assistant attempt、tool call / result、turn / step 边界及 request header，不通读全部变体和 Cordis API。
- **完成产物**：一组“用户消息 → assistant tool-call → tool result”的字段草图，并说明哪些事件不进入模型历史。

### 第 3 站：追通执行与所有权

- **05 [核心](../projects/deepseek-harness/docs/subsystems/core.zh.md)**
  - 读：`主干逐包速览`、`创建与所有权`、`Agent 句柄`、`拦截决策`。
  - 重点：create / dispose、followup / steer / inject、cancel / whenIdle、pre-step / request-error；其他维护方法先查含义，不追实现。
- **06 [Agent 生命周期图](../projects/deepseek-harness/docs/agent-lifecycle.zh.md)**
  - 第一遍只跟正常路径：入队 → claim → 请求 → 模型响应 → 工具 → 下一步 / 结束。
  - 第二遍跟拒绝接纳、请求失败与 retry；不逐条背事件名。
- **完成产物**：能区分 turn、step、attempt，并指出输入何时接纳、响应何时结算、谁有权释放 Agent。

### 第 4 站：让模型建议变成受控操作

- **07 [工具](../projects/deepseek-harness/docs/subsystems/tools.zh.md)**
  - 读：`ToolDefinition`、`执行：可扩展的 waterfall 加单调策略`。
  - schema 只看当前参数所需类型；暂跳完整 JSON Schema 子集、工具 UI 词汇和生成 API。
- **08 [工具执行流水线](../projects/deepseek-harness/docs/tool-execution-pipeline.zh.md)**
  - 跟三条路径：允许执行、拒绝 / 审批不可用、执行失败 → 归一化结果。
  - 找到权限、不可被后续监听撤销的 guard、超时包装、结果观察各自的位置；PTC、UI 卡片先跳。
- **完成产物**：一张 read 工具任务卡，写明参数校验、允许目录、callId、错误结果、取消信号和“不支持的操作直接拒绝”。

### 第 5 站：划清保存与恢复的保证

- **09 [会话持久化](../projects/deepseek-harness/docs/subsystems/persistence.zh.md)**
  - 读：`SessionHandle`、`flush 检查点`、`崩溃恢复保留被中断的轮次`、`格式拒绝`。
  - 重点：单写者、append / flush、关闭、恢复前缀与未知结果。暂跳完整 metadata、各代迁移和存储后端替换。
- **完成产物**：解释“工具已完成但结果未保存”为什么不能直接重做；列出保存失败和格式不支持时的动作。

### 第 6 站：补齐异常、预算与资源边界

- **回读 03 / 05 / 08，不增加整篇任务**
  - 03：`LlmFailure`、`ResolvedRetryPolicy`、`TokenUsage`，区分错误类别、重试和计量。
  - 05：取消和 dispose；08：拒绝、异常归一化与超时包装。
- **10 [通用防御性模式](../projects/deepseek-harness/docs/defensive-patterns/general.zh.md)**
  - 只读：`正交结果独立上报`、`异步状态不是同步状态`、`共享状态只有一个所有者`、`dispose 必须达到完全停稳`、`在分发器中隔离回调异常`。
  - 读清规则背后的失败场景即可，暂不读包迁移和性能专项。
- **预算设计不能从这些文档自动推出来**
  - `maxTokens`、usage、retry policy 和工具超时，分别约束或描述局部行为，不等于一个总任务预算。
  - 用已有[预算下钻 X2](2026-10-17_01_deepseek_harness_architecture_index.md)明确：额度作用域、attempt / 辅助请求计费、在途预留、耗尽动作及未知 usage。
  - 若要核对实际超时与循环提醒，还需两处 **docs 之外的短补充**：[工具超时 README](../projects/deepseek-harness/packages/guard/timeout-policy/README.zh.md)、[重复调用提醒 README](../projects/deepseek-harness/packages/guard/repeat-tool-reminder/README.zh.md)。前者是协作取消，后者是建议，不保证强制终止。
- **完成产物**：一张“异常 / 取消 / 预算耗尽 → 谁处理 → 是否继续 → 怎样报告”的决策表；对照验收 D04～D06、D11～D13、D16。

### 第 7 站：从理解转成可实施、可验收的任务

- **11 [工具编写参考](../projects/deepseek-harness/docs/cookbook/adding-a-tool.zh.md)**
  - 读：`最小形态`、`execute() 约定的规则`、`执行策略与观测`、`验证`。
  - 暂跳：后台工作、PTC 和 Web 展示。示例演示注册，不是可直接照搬的目录授权方案。
- **12 [测试策略](../projects/deepseek-harness/docs/testing.zh.md)**
  - 读：`层级`、`优先使用真实实现而非 mock`、`验证外部世界，而非自我报告`、`测试真实入口路径`。
  - 当前先理解证据类型，不运行命令，也不把上游覆盖率与 CI 全套要求照搬成 MVP 门槛。
- **完成产物**：按[实施方法](2026-10-17_02_harness_architect_to_coding_agent.md)形成 P0～P5 的任务拆分，引用[验收清单](2026-10-17_03_harness_acceptance_checklist.md)；模型自述成功不能作为验收结果。

**停止条件：**能解释数据、推进者、授权点、失败 / 取消、保存点与验收方式，就已完成本路径的阅读目标。接下来是选择实现范围与验证设计，不是自动去读剩余数百篇。

## 5. 不在主路径，但什么时候必须补读

| 触发问题 / 新范围 | 追加文档 | 为什么不是现在全读 |
|---|---|---|
| prompt / tool schema 怎样组装仍不清楚 | [系统提示词组装](../projects/deepseek-harness/docs/subsystems/system-prompt.zh.md) | 主线先知道职责与输入输出；需要修改组装时再追精确类型 |
| ctx、inject、effect 看不懂 | [Cordis 教程](../projects/deepseek-harness/docs/cordis-tutorial/index.zh.md)第 1～4 章；配置不懂再读第 5 章 | 用具体练习补前置，而非把整个框架当成另一个项目精读 |
| 要在现有 Harness 中写首个插件 | [第一个插件](../projects/deepseek-harness/docs/user/develop/basic/index.zh.md) → [构建工具](../projects/deepseek-harness/docs/user/develop/basic/tool.zh.md) | 这是 Web 加载教程；自建 headless 小系统不必先搭 Web |
| 开放写入、shell、真实工作区或服务化 | [文件系统](../projects/deepseek-harness/docs/subsystems/filesystem.zh.md)、[审批](../projects/deepseek-harness/docs/subsystems/approval.zh.md)、[sandbox](../projects/deepseek-harness/docs/subsystems/sandbox.zh.md)、[凭据](../projects/deepseek-harness/docs/subsystems/credentials.zh.md)、[公开部署](../projects/deepseek-harness/docs/user/guide/public-deployments.zh.md) | 安全要求不能省；只是当前不开放这些能力，新增能力前必须补 |
| 长对话 / 上下文压力 | [Token 计量](../projects/deepseek-harness/docs/subsystems/token-meter.zh.md) → [压缩](../projects/deepseek-harness/docs/subsystems/compaction.zh.md)、[spill](../projects/deepseek-harness/docs/subsystems/spill.zh.md) | 先建立输入 / 输出 / 总任务预算，再加入有损整理与存储外移 |
| 需要多步计划或委派 | [plan](../projects/deepseek-harness/docs/subsystems/plan.zh.md)、[todo](../projects/deepseek-harness/docs/subsystems/todo.zh.md)、[subagent](../projects/deepseek-harness/docs/subsystems/subagent.zh.md)、[jobs](../projects/deepseek-harness/docs/subsystems/jobs.zh.md) | 不是“单 Agent 可靠闭环”的前置；增加后要补所有权与共享预算 |
| 要换 provider / 做用户界面 | [添加模型适配器](../projects/deepseek-harness/docs/cookbook/adding-an-llm-adapter.zh.md)；或 [Web Client](../projects/deepseek-harness/docs/subsystems/web-client.zh.md) → [Conversation](../projects/deepseek-harness/docs/subsystems/conversation.zh.md) | 两条独立能力线，不因都属于产品而一起加入 MVP |
| 要外发遥测或升级已有会话 | [Session 遥测](../projects/deepseek-harness/docs/subsystems/session-telemetry.zh.md)；或 [格式状态](../projects/deepseek-harness/docs/session-format-status.zh.md) → 对应升级 / 持久化变更记录 | 涉及真实数据和兼容承诺时才展开；没读没测不能宣称支持 |

正式运行或改代码前另读[项目安全说明](../projects/deepseek-harness/SAFETY.md)、[仓库规则](../projects/deepseek-harness/AGENTS.md)和[开发准备](../projects/deepseek-harness/docs/development.zh.md)。这些是执行门槛，不代表现在授权安装、运行或调用模型。

## 6. 如何与已有教材配合

- 本文决定**官方文档先读哪篇、只读哪一节**。
- [架构索引](2026-10-17_01_deepseek_harness_architecture_index.md)用于对照 M0～M6 模块和 X1～X8 跨模块主题，不另开一套重复课程。
- [原主链教材](2026-10-16_01_deepseek_harness_task_lifecycle.md)和[失败教材](2026-10-16_02_deepseek_harness_failures_retries_and_cancellation.md)只在官方文档不易理解时查对应小节，不要求同步从头重读。
- 你已经问过的 followup / steer / inject、Driver / StreamAttempt 等内容，可以直接用于第 3、6 站自检，把时间留给尚不清楚的恢复、预算和验收设计。

**建议现在从[专题第 1 站](deepseek_harness/2026-10-17_01_system_and_plugins.md)开始；若职责图已清楚，直接进入[专题第 2 站](deepseek_harness/2026-10-17_02_messages_events_requests.md)。需要核对原文时再使用本文的章节索引。阅读顺序是建议，不代表你已完成或掌握这些站点。**
