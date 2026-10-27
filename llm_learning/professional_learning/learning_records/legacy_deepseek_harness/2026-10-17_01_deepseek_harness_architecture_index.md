# DeepSeek Harness：架构学习与源码索引

> 新的学习入口：先理解系统怎么分工、为什么这样设计，再按问题进入关键实现。不按目录通读，不继续扩充一篇包罗所有细节的教材。
>
> 阅读版本：`571902028a75c64c1d767f4ae5fd099485f0ddf0`。[本地源码](../projects/deepseek-harness/)已下载；本文依据项目文档、此前核对的主链及本次抽查的测试源码，未运行 Harness。

## 1. 学习方式与三份材料

直接学习请从[七站专题教材](deepseek_harness/README.md)进入：每站已重组原文、提炼概念与设计问题，并补必要例子。本文保留为按模块 / 跨模块问题查找的索引，不要求两套材料都从头阅读。

实际阅读官方文档时，按[文档层级与最小 MVP 路径](2026-10-17_04_deepseek_harness_docs_mvp_reading_path.md)选择先后与章节；本文继续作为模块、跨模块主题和源码下钻索引。

| 材料 | 用途 | 什么时候读 |
|---|---|---|
| 本文 | 系统全貌 → 模块职责 → 关键流程 → 源码 / 测试入口 | 默认入口，先看第 2～4 节 |
| [架构师如何指导 Coding Agent](2026-10-17_02_harness_architect_to_coding_agent.md) | 从需求、接口、约束到分批实现和审查 | 每读完一个模块，考虑怎样实现最小等价能力 |
| [功能与系统设计验收清单](2026-10-17_03_harness_acceptance_checklist.md) | 把设计要求变成可观察、可判定的测试 | 与模块同步读，不留到实现结束 |

每个主题只回答六件事：**解决什么问题 → 谁负责 → 关键数据 → 主流程 → 为什么这样选 / 有何代价 → 怎样验收**。同时保留“模块”和“跨模块设计主题”两个入口。下钻用分层 bullet：一级写子流程，二级写处理内容和关键设计点；源码链接放在流程之后。遇到分支选择，先给“状态 → 动作”表。先阅读，有疑问再提问，不主动测验。

证据分开记录：**文档声明、源码已核对、测试源码已核对、运行已验证**。前三者都不能写成“测试通过”；我们提出的实现要求另标为“拟定”，不归到上游能力名下。

## 2. L0：先看整个系统

Harness 的核心任务：把用户目标交给模型，把模型提出的操作交给受控工具，并管理执行、上下文、状态与交付。**模型负责生成，不负责可靠执行或证明任务成功。**

![DeepSeek Harness 系统全貌：启动装配、运行执行、记录与视图](images/deepseek-harness-system-layers.svg)

[单独打开 / 放大架构图](images/deepseek-harness-system-layers.svg)。从上到下看三个分区：**A 装配哪些能力 → B 谁推动执行 → C 记录怎样被使用**。蓝色实线是提交 / 调用，紫色虚线是返回 / 反馈，绿色实线是记录 / 派生；调用和返回分开画，避免长线交叉。

这是职责图，不表示所有箭头都是同步调用，也不表示每个 profile 都启动 Host。headless 没有服务器；持久化需要挂载相应实现；模型通常是外部 API，文件 / 进程 provider 则决定实际执行位置。

| 必须分清的对象 | 谁负责 | 不是什么 |
|---|---|---|
| 运行时装配 | profile / bundle / Cordis | 不是每次模型请求的上下文 |
| 活跃执行 | Agent 接口、Inbox、Driver | 不等于已保存的 Session |
| 可恢复记录 | Session 事件与持久化实现 | 不等于原样发给模型的 messages |
| 能力与约束 | 工具管线、能力 provider、权限策略 | 不等于模型“答应遵守” |
| 展示与交付 | 入口、Host / Client、事件消费者 | 显示 idle 不代表业务成功 |

**首要设计问题：谁拥有状态，谁有权推动下一步，谁承担失败后的处理？** 这比先记住多少个包更重要。

## 3. L1：模块职责与跨模块设计主题

### 3.1 按模块找负责人

下表是阅读顺序，不是完整包清单。验收编号来自[统一清单](2026-10-17_03_harness_acceptance_checklist.md)。

| 模块 | 拥有的数据 / 职责 | 关键设计问题 | 验收 |
|---|---|---|---|
| M0 装配与生命周期 | 配置、服务注册、Agent scope、资源释放 | 谁创建和释放资源？初始化未完成能否接单？ | D01、D05、D08 |
| M1 执行与输入调度 | inbox、turn / step / attempt、取消信号 | 忙时输入何时接纳？重试和下一步怎样区分？ | F03～F05、D04、D06、D11 |
| M2 Session 与视图 | 事件、messages 投影、保存 / 打开 | 哪些事实必须保存？怎样还原一次模型请求？ | F06、D02、D03 |
| M3 模型调用 | 通用消息 / stream、provider 转换 | 更换模型会影响哪些层？失败片段如何结算？ | F01、F04、D02、D07 |
| M4 工具与执行策略 | schema、callId、执行管线、能力 provider | 模型建议如何变成允许执行的操作？ | F02、F07、D03、D09 |
| M5 上下文与规划 | 提示组装、历史整理、规划辅助 | 改了模型可见内容后能否解释来源？计划是否真的控制执行？ | D02、D10 |
| M6 入口与交付 | 用户手势、命令、状态 / 流展示 | 前端状态过期时谁最终裁决？用户怎样辨别结果？ | F03、F08、D07 |

多 Agent、MCP、浏览器与桌面属于后续能力扩展。先能解释上述单 Agent 闭环，再读它们；不因包多而扩大第一版范围。

### 3.2 按设计主题检查遗漏

**模块表回答“谁做”，主题表回答“整个系统要保证什么”。** 下列主题不必各建一个模块，但必须有明确负责人、执行位置和验收证据；不能只写进 prompt。

| 主题 | 必须回答的设计问题 | 涉及模块 | 阅读入口 / 验收 |
|---|---|---|---|
| X1 异常处理与故障隔离 | 谁分类、谁重试、谁终止？局部失败会影响多大范围？错误如何到达用户？ | M0～M4、M6 | [失败教材](2026-10-16_02_deepseek_harness_failures_retries_and_cancellation.md)；F04～F05、D04、D16 |
| X2 预算与终止控制 | 限一次请求还是整个任务？时间、token、费用、步数如何累计？耗尽后谁阻止新工作？ | M1、M3～M5 | 下方预算分解；D10～D12 |
| X3 权限、人工介入与数据保护 | 哪些操作须批准？授权绑定什么？文件 / 工具内容能否越权？凭据、日志、会话怎样隔离和脱敏？ | M0、M2～M6 | [安全限制](../projects/deepseek-harness/SAFETY.md)、[工具管线](../projects/deepseek-harness/docs/tool-execution-pipeline.md)；F07、D09、D14 |
| X4 持久化、恢复与副作用一致性 | 何时算已保存？崩溃后从哪里恢复？未知结果怎样核对，怎样防止重复操作？ | M1、M2、M4 | [持久化](../projects/deepseek-harness/docs/subsystems/persistence.md)；F06、D02～D03、D15～D16 |
| X5 并发、资源生命周期与过载保护 | 队列 / 并发上限多少？慢消费者会不会拖垮执行？谁负责停止和释放资源？ | M0～M4、M6 | [AgentLoop](../projects/deepseek-harness/packages/core/agent-loop/README.md)；D01、D05～D06、D13 |
| X6 上下文正确性与规划约束 | 输入从哪里来？压缩会丢什么？哪些是建议，哪些由程序强制执行？ | M1～M5 | [上下文压缩](../projects/deepseek-harness/packages/compaction/README.md)；D02、D10 |
| X7 可观测性、评测与交付判定 | 能否定位到具体 attempt / tool call？成功看输出还是看产物？遥测失败会不会影响主任务？ | 全链路 | [测试策略](../projects/deepseek-harness/docs/testing.md)、[遥测](../projects/deepseek-harness/docs/subsystems/session-telemetry.md)；F08、D07、D14、D16 |
| X8 配置、兼容性与发布验证 | 错误配置何时拒绝？更换 provider 影响什么？旧记录怎样打开？正式入口与测试装配一致吗？ | M0、M2、M3、M6 | [总架构](../projects/deepseek-harness/docs/architecture.md)、[格式状态](../projects/deepseek-harness/docs/session-format-status.md)；D08、D15 |

这里是设计覆盖清单，不表示上游都提供了完整保证。多租户鉴权 / 配额、分布式调度、HA / 灾备在服务化部署时再专项展开；单机学习项目先明确“不支持”，不隐含承诺。

<details>
<summary>X1 下钻：发现异常 → 分类 → 恢复 / 停止 → 记录与通知</summary>

- **识别失败阶段与影响范围**
  - 区分配置 / 初始化、模型请求、工具执行、持久化、展示 / 遥测失败。
  - 关键设计点：工具找不到文件可以是一次调用失败；保存失败却可能使后续副作用失去可靠记录，不能套同一处理。
- **选择恢复动作**
  - 模型可重试错误：由请求恢复策略决定是否开始新 attempt；普通工具错误：形成关联结果，交给后续模型步骤处理。
  - 关键设计点：用户取消不是普通失败；重试模型请求不等于重做工具；外部操作结果未知时先核对，不盲目重试。
- **停止、结算与报告**
  - 追踪谁停止补充工作、谁等待已启动工作退出、谁记录结束原因、谁通知用户。
  - 关键设计点：清理异常不能被吞掉，也不能覆盖原始失败；非关键遥测故障与关键存储故障须分别定策略。
- **证据边界**
  - 模型重试、普通工具错误和取消已有[静态源码教材](2026-10-16_02_deepseek_harness_failures_retries_and_cancellation.md)。
  - 全链路错误优先级、存储故障与遥测故障组合仍需专题核对；D16 是自建目标，不是上游通过报告。

</details>

<details>
<summary>X2 下钻：定义预算 → 接纳前检查 → 执行中计量 → 耗尽后收束</summary>

- **定义预算对象与计量口径**
  - 单请求：输入窗口、最大输出、超时；单工具：执行时限与输出大小。
  - 整任务：总步数、attempt 数、累计 token / 费用、总时长；未来有子 Agent 时还要定义父子共享还是独立额度。
  - 关键设计点：记录 usage 只是计量，`maxTokens` 不是任务总额度；上下文占用和累计消耗也不是一回事。
- **决定在哪里执行限制**
  - 自建设计：新请求 / 工具启动前检查剩余额度；并发工作先预留，结束后结算；重试、压缩等辅助请求不得漏计。
  - 关键设计点：已发出的请求可能继续计费。usage 缺失时标记未知并采取明确策略，不能按零计费。
- **定义耗尽动作**
  - 区分提示模型节约的软提醒、阻止新工作的硬限制、对已启动工作的取消。
  - 关键设计点：停止时保留可用结果并说明原因；若还要让模型总结，必须预留这次总结的预算。
- **DeepSeek 已核对的局部机制**
  - [请求类型](../projects/deepseek-harness/packages/llm/llm/src/types.ts)有 `maxTokens` 与 `TokenUsage`；它们本身不证明存在全任务预算控制器。
  - [工具超时源码](../projects/deepseek-harness/packages/guard/timeout-policy/src/index.ts)只包装声明了 `timeoutMs` 的工具，发出取消信号后仍等待执行退出；不是强杀。
  - [重复调用提醒](../projects/deepseek-harness/packages/guard/repeat-tool-reminder/README.md)只劝模型改变做法，不阻止调用；[压缩机制](../projects/deepseek-harness/packages/compaction/README.md)管理上下文压力，不等于限制累计费用。
  - 全任务总额、额度预留及共享配额的完整实现尚未核对；不由这些局部机制推定“已有”或“没有”。拟定验收见 D11～D12。

</details>

## 4. L2：按主流程下钻

只展开当前问题对应的一项。每个一级 bullet 是子流程，二级说明“做什么”和“关键设计点”；到能解释输入、状态变化、输出和失败处理就停，不立即追完全部 provider。

<details>
<summary>M0：配置装配 → 创建 Agent → 初始化 → 释放</summary>

- **装配配置与服务**
  - 按 profile、bundles、patches 形成插件树，挂载 Agent registry、默认 factory 及所需能力。
  - 关键设计点：配置决定用什么实现；调用者面向 Agent 接口，不直接依赖 Driver。对应 X8。
- **创建与初始化**
  - 准备 Session / scope，执行 setup，登记对象，再通知并等待创建监听完成。
  - 关键设计点：先保存清理句柄，再调用监听；能查询对象不等于初始化完成，`created` 也不等于 ready。
- **失败回收与正常释放**
  - 创建失败沿已获取资源回收；dispose 停止并等待工作退出，再解绑注册与 scope。
  - 关键设计点：通知已被观察到就不能“撤回”；工作未退出时不能先拆掉其依赖。对应 X1、X5。
- **源码 / 证据**
  - [装配说明](../projects/deepseek-harness/docs/architecture.md)、[app-boot](../projects/deepseek-harness/packages/boot/app-boot/)、[registry](../projects/deepseek-harness/packages/core/agent/src/index.ts)、[factory](../projects/deepseek-harness/packages/core/agent-loop/src/index.ts)；Cordis 不熟时查[入门](../projects/deepseek-harness/docs/cordis-primer.md)。
  - [资源生命周期测试](../projects/deepseek-harness/packages/core/agent-loop/tests/resource-lifecycle.spec.ts)：已读工作退出前保留资源的断言，未执行。

</details>

<details>
<summary>M1：输入 → 接纳边界 → 请求 → 工具 → 下一步 / 结束</summary>

- **排队与接纳输入**
  - 投递方式决定队列目标和是否唤醒；Driver 在 claim 边界取输入，pre-step 决定实际接纳内容。
  - 关键设计点：到达时间不等于发送时间；已 claim 的请求不会因新 steer 被改写。对应 X5。
- **推进一次 step**
  - 组装 prompt / tools，准备模型调用，提交接纳事实，再从 Session 派生并冻结请求。
  - 关键设计点：同 step 重试是新 attempt，不重复接纳用户消息；预算设计必须覆盖每次 attempt。对应 X2、X6。
- **处理响应并决定是否继续**
  - 结算模型响应，调度工具并记录结果；仍欠工具反馈后的请求或有可接纳输入时继续，否则收束 turn。
  - 关键设计点：模型、工具和 Driver 分别承担生成、执行、调度；取消停止推进，但不回滚已有副作用。对应 X1、X4。
- **源码 / 证据**
  - [Inbox](../projects/deepseek-harness/packages/core/agent-loop/src/inbox.ts)、[Driver](../projects/deepseek-harness/packages/core/agent-loop/src/agent.ts)、[StreamAttempt](../projects/deepseek-harness/packages/core/agent-loop/src/assistant-stream.ts)；细节查[主链教材](2026-10-16_01_deepseek_harness_task_lifecycle.md)第 4～9 节。
  - [inbox](../projects/deepseek-harness/packages/core/agent-loop/tests/inbox.spec.ts)、[cancel](../projects/deepseek-harness/packages/core/agent-loop/tests/cancel.spec.ts)、[retry](../projects/deepseek-harness/packages/llm/llm-retry/tests/retry.spec.ts)是测试阅读入口，未执行。

</details>

<details>
<summary>M2：记录事实 → 投影 messages → 保存 → 打开 / 恢复</summary>

- **追加执行事实**
  - 消息、请求配置、调用 / 结果及执行边界进入事件记录，维持顺序与关联。
  - 关键设计点：Session 内存 append 与持久化 flush 不是同一个完成点。对应 X4。
- **派生不同视图**
  - `deriveMessages()` 得到模型历史；其他投影服务展示、统计等消费者。
  - 关键设计点：失败 attempt 可以用于排障但不进入普通模型历史；改变模型内容必须能追溯。对应 X6、X7。
- **保存、打开与恢复**
  - 沿持久化 handle 检查写入、flush、格式选择及打开后的状态重建；再进入 Agent resume。
  - 关键设计点：历史恢复不证明外部动作成功，也不自动给出安全重做位置；格式兼容与中断尾部另查。对应 X4、X8。
- **源码 / 证据**
  - [Session 说明](../projects/deepseek-harness/docs/subsystems/session.md)、[事件类型](../projects/deepseek-harness/packages/core/session/src/types.ts)、[Session](../projects/deepseek-harness/packages/core/session/src/index.ts)、[消息视图](../projects/deepseek-harness/packages/core/session/src/surface.ts)、[JSONL 后端](../projects/deepseek-harness/packages/session/session-persistence-jsonl/)。
  - [请求重建测试](../projects/deepseek-harness/packages/core/agent-loop/tests/request-reconstruction.spec.ts)：已读普通多步的历史前缀、冻结和 header 数量断言；完整恢复组合待核对，未执行。

</details>

<details>
<summary>M3：通用请求 → provider 请求 → 流 → 持久结算</summary>

- **绑定模型能力与转换请求**
  - 将内部 messages、工具声明与配置转换为 provider 请求；认证、URL、signal 属于传输控制。
  - 关键设计点：工具格式、reasoning replay、动态 prompt 有 provider 差异，不能靠通用字段假装全部兼容。对应 X8。
- **接收流并结算 attempt**
  - chunk 用于实时展示；StreamAttempt 汇总完整内容，结算为 assistant message 或失败 attempt。
  - 关键设计点：界面见过 chunk 不等于内容已持久化；usage 是计量数据，不是预算执行器。对应 X2、X7。
- **处理请求失败**
  - 请求错误进入恢复扩展点，策略决定是否退避后再试；取消须传到流和等待过程。
  - 关键设计点：失败片段不能自动执行成工具；重试次数、总耗时和其他恢复插件的组合要一起核对。对应 X1、X2。
- **源码 / 证据**
  - [LLM 说明](../projects/deepseek-harness/docs/subsystems/llm-streaming.md)、[通用类型](../projects/deepseek-harness/packages/llm/llm/src/types.ts)、[序列化](../projects/deepseek-harness/packages/llm/llm-deepseek/src/serialize.ts)、[响应转换](../projects/deepseek-harness/packages/llm/llm-deepseek/src/translate.ts)、[重试插件](../projects/deepseek-harness/packages/llm/llm-retry/src/index.ts)。
  - 已核对路径的细节见[主链教材](2026-10-16_01_deepseek_harness_task_lifecycle.md)第 2、5、6 节和[失败教材](2026-10-16_02_deepseek_harness_failures_retries_and_cancellation.md)，均非运行证据。

</details>

<details>
<summary>M4：tool-call → 参数 / 权限检查 → 执行 → tool-result</summary>

- **登记调用与安排顺序**
  - 从 assistant 的 tool-call 取得 callId，记录调用，按并行安全 / 独占规则调度。
  - 关键设计点：调度资格不是操作授权；callId 必须贯穿结果和错误。对应 X4、X5。
- **检查并执行操作**
  - 工具管线负责参数与策略处理；工具通过能力 provider 完成真实操作，声明的时限由超时插件包装。
  - 关键设计点：权限须在程序里落实；工作目录不等于授权目录；协作超时不保证强制终止。对应 X2、X3。
- **记录结果并反馈**
  - 将成功、普通错误或未启动的跳过结果关联回调用，供后续模型请求使用。
  - 关键设计点：工具错误不是整轮必然失败；进程崩溃时“操作成功但未记结果”也不是可自动重做。对应 X1、X4。
- **源码 / 证据**
  - [工具管线说明](../projects/deepseek-harness/docs/tool-execution-pipeline.md)、[能力接口](../projects/deepseek-harness/docs/capability-seams.md)、[调度器](../projects/deepseek-harness/packages/core/agent-loop/src/tool-calls.ts)、[工具管线](../projects/deepseek-harness/packages/core/tools/src/index.ts)、[read](../projects/deepseek-harness/packages/fs/tool-fs/src/read.ts)。
  - 普通路径已有静态分析；审批、sandbox、shell 的完整策略链留待专题，不从“有插件”推导完整隔离保证。

</details>

<details>
<summary>M5：上下文来源 → 整理 / 规划 → 下一次请求</summary>

- **追踪输入来源**
  - 选一个具体变化：工作目录通知、工具结果、系统提示更新或规划内容，找到对应事件 / 投影。
  - 关键设计点：记录来源不等于赋予指令权限；用户目标、外部资料和插件上下文不能混淆。对应 X3、X6。
- **判断是否需要整理**
  - 对照工具结果裁剪、历史压缩、图片 offload 各自的触发条件；另查规划数据何时更新。
  - 关键设计点：输出落盘不必然缩短模型输入；计划文本不等于可执行控制结构。对应 X2、X6。
- **验证变化后的请求**
  - 比较整理前后实际 messages、token 计量及必要任务约束，并检查能否从记录解释变化。
  - 关键设计点：压缩也可能调用模型、消耗预算；不能只验“变短”，还要验“留下了什么”。
- **源码 / 证据**
  - [system-prompt](../projects/deepseek-harness/packages/core/system-prompt/)、[context](../projects/deepseek-harness/packages/context/)、[compaction](../projects/deepseek-harness/packages/compaction/)、[plan](../projects/deepseek-harness/packages/plan/)。
  - 本项是下钻路线；已核对部分文档和既有主链，完整触发顺序与规划实现仍待专题核对。

</details>

<details>
<summary>M6：用户输入 → 投递选择 → 执行反馈 → 交付</summary>

- **选择投递意图**
  - UI 根据运行状态、steer 能力、用户偏好及手势选 queue / steer；Host 转为 Agent 调用。
  - 关键设计点：不是用模型猜输入语义；前端状态可能过期，实际接纳由 Driver 决定。对应 X5。
- **展示运行过程**
  - 分开追踪实时 stream / status 与已提交 Session 事件的消费者。
  - 关键设计点：暂态反馈不能冒充持久结果；慢界面、断连和重新订阅的处理另行核对。对应 X5、X7。
- **收束并交付**
  - headless 等待空闲、同步记录、汇总输出及退出码；Web 由对应展示路径表达结果。
  - 关键设计点：idle 可能来自完成、错误或取消；回答内容是否正确仍需独立验收。对应 X1、X7。
- **源码 / 证据**
  - [headless](../projects/deepseek-harness/packages/bundle/headless/src/index.ts)、[UI 提交策略](../projects/deepseek-harness/packages/client/ui-conversation/src/client/input/submission-policy.ts)、[Host 命令](../projects/deepseek-harness/packages/api/session-controller/src/commands.ts)。
  - 已核对 headless 主链与 UI 投递决策；Web 完整传输、断连和恢复不是已验证结论。

</details>

## 5. 不只记机制：从实现提炼设计取舍

下表“代价 / 替代方案”是学习分析，不声称是上游作者的完整动机。

| 设计问题 | DeepSeek 的选择 | 收益与代价 | 自建小系统时怎样取舍 |
|---|---|---|---|
| 如何增加能力而不反复修改循环？ | 插件、服务接口、typed events | 易组合替换；代价是装配、事件顺序与清理更复杂 | 先用显式模块组合；出现真实替换需求再扩展插件机制 |
| 如何追溯模型看到了什么？ | 从 Session 事件投影请求内容 | 能解释历史变化；代价是投影、格式演进和持久化一致性 | 保留“模型可见输入可追溯”的要求，不急着复制全套存储 |
| 为什么 Driver 和 StreamAttempt 分开？ | 长生命周期调度与单次请求组装分离 | 重试不污染调度职责；需定义 attempt 的提交与放弃 | 即使只用函数实现，也保持这两种状态不混用 |
| 为什么 UI 不直接改当前请求？ | UI 给投递意图，Inbox / Driver 在边界接纳 | 调度有单一负责人；代价是输入不会总是立即生效 | 写清 queue / steer 规则，并让界面说明等待状态 |
| 如何保证“任务完成”？ | 运行状态和 Session 提供执行证据；项目测试要求外部检查产物 | 执行结束可观察，但正确性仍需独立判定 | 把完成条件作为产品要求，不让模型自述替代测试 |

## 6. 后续学习顺序与停止条件

| 顺序 | 下一份专题回答什么 | 本轮做到什么就停 |
|---|---|---|
| 1 | M0 / M1 + X1 / X5：谁拥有生命周期，异常时谁收尾？ | 职责图、创建 / 结束路径、故障归属和并发规则 |
| 2 | M2 + X4 / X8：记录、视图、保存与恢复为什么分开？ | 请求来源、恢复失败例子及格式兼容边界 |
| 3 | M3 / M4 + X1 / X2 / X3：请求与工具怎样受控？ | 错误决策表、预算作用域、授权点与副作用边界 |
| 4 | M5 / M6 + X6 / X7：上下文变化后怎样解释和交付？ | 请求前后对照、投递表、观测证据和独立完成判定 |
| 5 | 自建最小系统的方案审查 | M0～M6 有负责人，X1～X8 有机制或明确不适用项；确认技术选择、任务卡和验收项后再编码 |

每轮交付一份短专题：关键结论放前面，源码与细节可展开；更新验收映射，不另建重复清单。[包与类交互图](../architecture_maps/deepseek_harness_explorer.html)保留作查找工具，不作为必读路线。
