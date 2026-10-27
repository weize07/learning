# DeepSeek Harness：读取失败、请求重试与用户取消

- 学习日期：2026-10-16。
- 阅读阶段：独立项目课，第 3 篇。
- 前置：[一次读文件任务的完整主链](2026-10-16_01_deepseek_harness_task_lifecycle.md)。
- 每日概览：[topics_by_date](topics_by_date.md)。
- 固定版本：`571902028a75c64c1d767f4ae5fd099485f0ddf0`。
- 证据：固定源码与官方组件文档的静态分析；图、消息和场景均为教学示例，未连接模型或执行故障注入，不是实测日志。

本篇继续同一个任务：“读取 README.md，解释超时参数的单位，不修改文件。”

## 1. 先看决策表：失败不是统一的 retry

以下假设使用默认 loop；LLM 自动重试还要求装配 `llm-retry`，且该次失败取得适用的 provider 策略。策略可被覆盖，表中不代表所有 profile 都相同。

| 状态 / 错误 | 程序层的普通选择 | 后续由谁处理？ |
|---|---|---|
| read 的参数、路径或实际读取失败，被工具管线转成错误结果 | 记录 `tool/result`，通常继续下一 step | 模型根据错误选择查找、修正或说明不足 |
| 工具策略拒绝、审批不允许 | 形成相应错误 / 取消结果，不执行被拒绝的工具体 | 后续路径还受 Agent 取消信号等影响；模型不能自行扩大权限 |
| 模型请求失败到达 `agent/request-error`，normal 策略允许且预算未耗尽 | 记录重试安排，退避等待，在同一个 step 内重试 | 重试插件作决定，driver 发起新 attempt |
| normal 策略不允许、次数耗尽或没有适用策略 | 交给后续恢复监听器；若都不处理，结束为 error | 程序显式报告，不伪装成正常回答 |
| 用户取消活跃 Agent | 默认清空待处理输入，触发活动取消信号，收束为 aborted | 各执行层协作停止，driver 记录边界 |
| 模型达到输出 token 上限 | 使用 max-tokens 结束原因，不走普通瞬时错误重试分支 | 不等于任务正常完成；有额外输入时还需按控制流收束 |
| 工具调度器或记录环节异常，不能完成普通结果路径 | 排空已开始的调用，按分支尝试补齐保守结果并终止 | 不应声称工具成功或默认重放外部操作 |

先记住这三个设计意图：

> **工具错误通常是可供模型使用的环境反馈；请求错误通常是程序的传输 / 服务恢复问题；用户取消是控制意图，不应当成失败后继续重试。**

这些是分层职责，不是“工具永远不会抛异常”或“所有网络异常都自动可重试”的承诺。

## 2. 先认识两种不同的错误数据

### 2.1 工具失败：对应一次调用的 tool 消息

假设模型已提出 `call_read_1`，但目标不存在。内部结果可以表达为：

```json
{
  "id": "M3",
  "role": "tool",
  "toolCallId": "call_read_1",
  "isError": true,
  "content": [
    {"type": "text", "text": "Error: the requested README.md was not found."}
  ],
  "source": {"kind": "tool", "callId": "call_read_1"}
}
```

短 ID 和错误文字为教学示例，不承诺真实后端逐字返回这句话。重要的是 role、`isError` 与调用关联，不是错误字符串的排版。

它回答模型：“你刚才要求的那次读取没有成功。”所以应该保留调用建议和失败反馈：

```text
assistant：read(...), id = call_read_1
tool     ：isError = true, toolCallId = call_read_1
```

不要把工具异常伪装成文件为空、正常成功，或改成“用户说文件不存在”。工具错误仍可能影响用户目标，但通常不需要立即杀掉整个 Agent。

### 2.2 模型请求失败：不是一个虚构的 tool 结果

如果模型请求超时，还没有正常接受的工具调用，就没有理由制造一条“read 超时”的 tool 消息。失败属于本次模型 attempt。

相关词只需认识四个：

| 名称 | 表达什么 |
|---|---|
| `assistant/attempt` | 记录未成为正常 assistant 消息的一次尝试；不自动进入模型可见历史 |
| `agent/request-error` | driver 交给恢复逻辑的扩展点，不是发给模型的消息 |
| `llm/retry` | 记录安排第几次重试、原因与等待时间 |
| `llm/retry-started` | 等待完成后，记录即将开始该次重试 |

`llm/retry` 等是 Session 中的非对话记录，不会仅因保存了就自动增加 prompt token。每次真正再次调用模型仍可能重复计费。

## 3. 分支 A：文件不存在，为什么通常继续 loop？

**设计问题：环境反馈失败，是否意味着控制器也必须失败？**

不一定。文件路径可能有误，模型可以用搜索工具定位，也可以明确说明缺少依据。让错误进入对话，可以继续完成目标；但是否真的能修正，需要执行证据，不能只凭“我会重试”。

![读取失败与模型修正](images/deepseek-harness-failure-tool.svg)

正常路径：

1. assistant 的调用建议已经记录。
2. 调度器记录 `tool/call`，工具执行进入 read。
3. 读取错误沿工具调用返回，由工具管线捕获并定型为 `isError: true` 的结果。
4. 调度器写入对应的 `tool/result`。
5. 普通非 concludesTurn 结果让 loop 继续下一 step。
6. 下次请求包含原问题、调用建议及失败结果；模型决定下一步。

**谁重试？** 在这条主路径中，不是 `llm-retry` 自动重复 read。若模型修正路径后再次调用 read，那是一次新的工具调用，通常有新的 call ID，也是一条新的决策记录。

例如：

| 模型读到的反馈 | 合理的后续方向，非本项目自动保证 |
|---|---|
| 文件不存在 | 在许可范围内查找正确路径，或请求用户提供文件 |
| 参数值非法 | 修正参数，避免重复同一非法调用 |
| 路径被禁止 | 说明权限不足，走授权机制或选择允许的方案；不绕过限制 |
| 已读取但没有单位定义 | 继续定位定义，或报告证据不足 |

如果模型每次都用同一错误路径，就需要无进展检测、工具预算等额外限制。本篇不能因为基础 loop 会继续，就宣称项目已经覆盖所有循环风险。

## 4. 分支 B：模型请求失败，为什么在同一 step 重试？

**设计问题：模型服务临时不可用，是否应该让模型通过一条消息来“决定重试自己”？**

不需要。driver 和重试策略已经掌握错误码、次数与取消信号，应该在程序层恢复，不额外消耗一次模型推理来解释网络超时。

![模型请求失败与同 step 重试](images/deepseek-harness-failure-retry.svg)

图展开“适配器提供终止错误、normal 策略接受、等待成功、第二次尝试正常返回”的单次恢复：

1. 本次 StreamAttempt 收到终止失败，被记录为 `assistant/attempt`，而非正常成功的 `assistant/message`。
2. driver 进入 `agent/request-error`，携带 failure、provider、捕获的 retryPolicy 和 signal。
3. 重试插件检查策略；normal 模式不匹配或耗尽时向后续恢复逻辑委托。
4. 决定重试后，先追加 `llm/retry`，再进行可取消的等待。
5. 等待完成，追加 `llm/retry-started`，返回 `{ kind: 'retry' }`。
6. driver 的同一个 `step()` 再试一次，但新建 StreamAttempt。

这是在重跑本步的模型请求尝试，**不是重新执行从任务开始以来的所有 step 和工具**。第一次接纳的用户消息也不会因每次重试重复追加。普通重试复用本步已绑定的 prompt / tools 组装结果，并按当时的会话视图重建请求；扩展恢复逻辑若有意改变视图，不能再假设内容逐字相同。

### 4.1 次数和等待时间：固定版本的 normal 默认值

| 配置 / 条件 | 默认行为 |
|---|---|
| 模式 | normal |
| 可重试错误码 | `EMPTY_RESPONSE`、`RATE_LIMIT`、`SERVER`、`TIMEOUT`、`TRANSPORT` |
| `maxRetries` | 5，指首次请求之后最多再试 5 次 |
| 初始本地退避 | 500 ms |
| 最大单次退避 | 10,000 ms |
| jitter | 0.1，即本地指数退避乘以约 0.9～1.1 的随机因子，并受上限约束 |
| 有效且不超过上限的 provider Retry-After | 使用服务端建议等待时间 |
| Retry-After 超过上限 | normal 向后续恢复逻辑委托，不擅自把它截短后继续请求 |

在 provider / policy 不变、没有额外恢复逻辑时，默认最多是 **1 次初始请求 + 5 次重试 = 6 次尝试**，不是“总共 5 次”。计数按当前 step 的 provider / policy 维护，而不是整个应用永远共用一个计数。

这些是 `retry-policy.ts` 的默认值，不是性能建议，也不表示本机已启用它。API key 错误等不在默认 eligible 集合内的失败，不能假定会被 normal 策略自动修复。

### 4.2 always 模式不是“更可靠的有限重试”

源码还支持 always：在这个模型请求恢复扩展点上，先等待下游恢复处理；若未得到重试决定，再对取得该策略的模型请求失败进行无次数上限重试。成功、取消或卸载等条件才使相应流程停止；卡住且不响应取消的扩展还可能妨碍收束。

因此要分别判断：

- 可恢复性：再试一次是否可能成功？
- 权限 / 配置：重试能不能修复？
- 成本：是否允许无上限调用？
- 生命周期：等待能否响应取消、已开始的工作能否结束？

不能把 always 套用为“所有工具和系统异常都永远重试”；该插件也不为绕过 agent loop 的裸 `ctx.llm.stream()` 调用提供同样的尝试恢复机制。

## 5. 为什么失败 attempt 不直接加入 messages？

假设一次响应只返回了半段工具参数，随后连接失败：

```text
attempt A：{"file_path":"READ...
attempt B：{"file_path":"README.md"}
```

两次尝试的参数不能拼接，A 也不能伪装成一次完整调用。上篇的 Driver / StreamAttempt 分离在这里发挥作用：

| 状态 | 保存到哪里 | 下一次模型通常看到什么？ |
|---|---|---|
| 正常完整响应 | `assistant/message` | 推导出的完整 assistant 内容 |
| 请求失败，进入重试 | `assistant/attempt` 及 retry 记录 | 不把失败片段当成普通历史消息发送 |
| 用户取消，已有可安全保留的输出前缀 | `assistant/message`，标记 interrupted | 后续继续时可保留已交付前缀 |
| 用户取消，没有可保留内容 | 记录 `assistant/attempt` | 没有凭空构造一段回答 |

“记录已提交”与“尝试成功”仍然不同：StreamAttempt 的 committed end 可以对应一次 `assistant/attempt`，不能仅看 committed 就显示“任务成功”。

重试策略事件不进入模型历史，是为了把传输恢复与任务内容分开；取消时保留输出前缀，是为了让后续历史尽量与用户已见内容一致。这是两类不同状态，不是相互矛盾的规则。实际流式 UI 如何展示失败尝试，仍由入口层决定。

## 6. 分支 C：用户取消，为什么不是马上 kill 然后忘掉？

**设计问题：如何停止后续工作，同时不把已经发生的输出或副作用抹掉？**

默认 loop 使用协作式取消。`agent.cancel(cause, options)` 不是一个“立刻销毁全部进程并回滚”的操作：

- 未指定 `keepInbox` 时清空待处理输入；这是取消时刻的队列清理，不是永久禁止后续新消息。
- 活跃时触发当前活动的 AbortController。
- 模型调用、退避等待和工具执行等路径据此停止或收束。
- 已进入 step 的路径按分支记录结果与 step / turn 结束。

![流式响应期间用户取消](images/deepseek-harness-failure-cancel.svg)

图只展开“模型正在流式输出，且已有可保留前缀”的路径。不同取消位置的行为要分开看：

| 取消发生在哪里 | 程序应该如何收束；固定源码中的相关行为 |
|---|---|
| 模型流式响应中 | 信号传给调用层；默认 driver 尝试保存 interrupted 前缀，再以 aborted 收束 |
| 重试退避等待中 | 取消计时等待，不再正常发出 retry-started / 启动该次重试 |
| 工具尚未派发 | 不启动工具体；调度器为跳过调用记录相应错误结果 |
| 工具体已开始 | 不遗弃已开始的 Promise；协作取消并等待其收束，再记录结果 |
| 已成功提交外部写操作之后 | 取消不能撤销；需要业务侧查询、补偿或其他恢复机制 |
| 尚未真正进入 step | 可能没有 step/start，自然不能要求每次都有一对 step 边界 |

这里等待已开始操作收束，是为了避免宣布“已经停止”后还在后台悄悄产生结果。但这也意味着：**取消速度受底层配合程度影响**，不响应取消的扩展可能拖住生命周期。

`keepInbox: true` 只表示不在该次取消里清空队列，不应据此推断“一定自动恢复执行”。新的唤醒输入、活动收束和销毁状态还会影响后续处理。

## 7. 哪些事情不是 retry / cancel 自动解决的？

### 7.1 超时不证明远端没有执行

当前例子是只读文件，重复读取通常不会造成重复业务写入，但内容仍可能已经变化。如果工具换成提交订单：

```text
服务端已提交订单
    ↓
响应在网络中丢失
    ↓
客户端看到 timeout
```

此时自动重新提交可能创建第二张订单。模型请求重试、工具重试与业务操作去重是不同问题；需要业务幂等键、状态查询等机制，tool call ID 不自动替代它们。

### 7.2 steer 不是 stop

| 用户意图 | 应选择的控制思路 |
|---|---|
| 下一步改看另一个文件 | steer，在后续输入接纳边界影响方向 |
| 先完成本轮，之后做另一个任务 | followup |
| 下次请求顺便知道目录改变 | inject |
| 现在不要继续危险执行 | 取消 / 授权控制，不能仅依赖模型之后读到文本 |

即使取消已发出，也必须核查外部操作结果，不能承诺所有正在发生的动作都被及时阻止。

### 7.3 idle 不是成功证明

`whenIdle()` 等待驱动收束，错误或取消后的 idle 仍然是 idle。headless 入口还会查看运行区间的结束原因；普通 completed 退出码为 0，error / aborted 等不是正常成功退出。任务是否有依据、文件是否被正确处理，仍需独立验收。

## 8. 源码定位与后续可执行检查

按问题定位，不要求顺序通读所有文件：

| 要确认什么 | 固定源码入口 |
|---|---|
| 普通工具失败怎样变成结果 | [工具管线](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/packages/core/tools/src/index.ts)：`dispatchToolBody()`、`toolErrorResult()` 调用点 |
| 调用 / 结果、取消跳过与排空 | [工具调度器](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/packages/core/agent-loop/src/tool-calls.ts) |
| 错误恢复、前缀保存与 turn/end | [默认 driver](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/packages/core/agent-loop/src/agent.ts)：`step()`、`turn()`、`cancel()` |
| default 次数、错误码、退避 | [retry-policy.ts](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/packages/llm/llm/src/retry-policy.ts) |
| 哪个扩展点决定重试、何时记事件 | [llm-retry 实现](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/packages/llm/llm-retry/src/index.ts)、[组件说明](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/packages/llm/llm-retry/README.md) |
| 实时 end 与 Session 提交的关系 | [AssistantStreamAttempt](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/packages/core/agent-loop/src/assistant-stream.ts) |

如果后续运行最小测试，应优先用可控的假模型 / 工具，而不靠真实付费 API 碰运气。以下是测试设计，不是已通过的结果：

| 注入条件 | 应核对的记录与可测行为 |
|---|---|
| read 明确返回错误 | tool 结果 `isError: true`，调用 ID 对应；不伪造文件内容 |
| 首次模型失败、策略允许、下次成功 | 同 turn / step 的不同 attempt；retry 事件存在，用户消息不重复接纳 |
| 固定 normal 策略，连续 eligible 失败 | 默认额外重试最多 5 次，耗尽后无下游处理则 error |
| 退避等待中取消 | 不启动等待中的下一次模型请求；记录与控制状态一致 |
| 输出前缀后取消 | 正常可保存路径包含 interrupted 标记，而不是虚构完整成功响应 |
| 工具已开始后取消 | 确认停止新派发、等待已开始操作收束，核查真实副作用 |

还应区分“流的终止失败到达恢复扩展点”和“前置组装 / 记录 / 插件抛出的其他异常”。后者不一定经过同一重试分支；本篇并未证明所有异常都自动可恢复。

## 9. 小结与进度

不要先问“失败了重试几次”，先问：

> **失败属于哪一层？已经发生了哪些事实？谁有权决定继续？重做会不会重复副作用？**

- 已提供：工具错误回填、normal / always 请求恢复、失败 attempt 与可见历史的区别、协作取消及三张局部时序图。
- 已核对：固定版本源代码与默认策略值；教材中的场景、消息、时序均为静态说明。
- 未执行：模型调用、故障注入、进程中止测试或持久化恢复实验。
- 下一篇：Session 的事件记录与模型消息视图，从“保存过什么”追到“本次实际看见什么”，再进入上下文整理与规划插件。
