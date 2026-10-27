# 第 3 站：执行循环与生命周期

[专题入口](README.md) · 上一站：[消息与事件](2026-10-17_02_messages_events_requests.md) · 下一站：[工具与策略](2026-10-17_04_tools_and_policy.md)

## 1. 是什么：turn、step、attempt 与生命周期

本篇的 **Driver 指 ReactLoopAgent 实例内部启动的一次异步执行活动**，不是独立类实例，也不等于长期存在的 Agent 对象。Agent 是对外接口；AgentLoop 是工厂与生命周期服务。实例可先后启动多次 Driver 活动，每次可包含多个 turn；生命周期与协作方式见[第 1 站](2026-10-17_01_system_and_plugins.md)。

| 单位 | 含义 | 为什么单独存在 |
|---|---|---|
| turn | 一段 Agent 执行，包含零个或多个 step | 表达一次执行区间的开始和结束 |
| step | 一个模型请求阶段及其工具执行 | 表达“生成 → 操作 → 再判断”的一次推进 |
| attempt | step 内的一次模型尝试 | 请求失败可重试，不必重新接纳输入 |

正常读文件任务：turn 1 / step 1 得到工具调用并执行；step 2 获得最终回答。如果 step 1 请求失败后重试，是 step 1 的新 attempt，不自动变成 turn 2。

图中的 step 是逻辑单位；有重试时，它不只对应一次网络请求。

## 2. 为什么要做：新输入、重试与退出会互相竞争

- 用户可能在模型请求或工具执行期间继续输入，必须定义何时生效，否则会丢失、重复或修改在途请求。
- 同一次推理可能需要重试，但不应重新接纳用户输入或重复已完成操作。
- 初始化、执行、取消和资源释放是异步过程；没有明确所有权，就可能在工作还没结束时拆掉依赖。

## 3. 怎么做：在明确边界接纳、推进和释放

### 3.1 输入投递与接纳

下表限定 Agent 已初始化、未处于取消 / 维护 / 销毁阶段。

| 输入方式 | Agent 空闲时 | 正在工作时 |
|---|---|---|
| followup | 唤醒，排入新的 turn | 排队，留待后续 turn |
| steer | 唤醒 | 最近可接纳的 step 边界加入 |
| inject | 仅排队，不唤醒 | 最近可接纳的 step 边界加入 |

**接纳边界是 claim，不是 HTTP 发出时刻。**

例子：

- 10:00:00，step 2 已 claim 输入，开始异步准备请求。
- 10:00:01，用户 steer：“只看安装说明。”
- 10:00:02，step 2 请求才发出。

这条 steer 仍来不及进入 step 2，需等后续可接纳边界。它不修改在途请求，也不自动打断正在执行的工具。

UI 可以选择投递意图，最终接纳时机由 Driver 裁决。取消后的重新投递等特殊分支需另查，不能直接套上表。

### 3.2 正常执行的五个子流程

- **打开 turn、认领输入**
  - Driver 建立执行边界，从 inbox 取本次候选输入。
  - 关键设计点：认领后这些项已离开队列；pre-step 没选中的项不会自动重新排队。
- **决定是否进入 step**
  - 组装 prompt / tools，经 pre-step 决定 reject 或 enter。
  - 关键设计点：首批被拒绝或为空，可结束一个没有 step 的 turn；工具续接允许没有新用户输入。
- **准备并执行模型请求**
  - 进入 step，经过 request 与 prepareCall；同步提交接纳内容和必要配置，再派生请求、发起流。
  - 关键设计点：在上述异步准备阶段取消，不应把尚未接纳的用户 / system 内容当成已发送事实。
- **结算响应与执行工具**
  - StreamAttempt 汇总本次尝试；成功消息提交后，再把 tool-call 交给调度器。
  - 关键设计点：实时片段不直接执行成工具；失败 attempt 的部分参数不能触发操作。
- **判断继续还是结束**
  - 工具结果通常需要下一次请求；新 next-step 输入也可能继续执行；无续接需求后经过停止检查点，结束 turn。
  - 关键设计点：不是“有 assistant 消息就结束”，也不是“没有新用户消息就结束”。

### 3.3 创建、失败回收与释放

- **创建**
  - 调用者通过 registry 创建，由默认 factory 构造 Session、Agent scope 并执行 setup。
  - 登记对象，保存 detach 责任，再发出创建通知并等待相关异步监听。
  - 关键设计点：`agent/created` 是初始化扩展点，不是宣告所有初始化已结束；登记成功也不是完全 ready。
- **创建失败**
  - 沿已获取资源进行回收，避免半初始化对象继续工作。
  - 关键设计点：外部已观察到的通知不能撤回，不能把这一过程理解成完全不可见的数据库事务。
- **释放**
  - handle 的 dispose 停止工作、等待实际退出，再移除注册并清理 scope。
  - 关键设计点：cancel 是提出停止；dispose 是带所有权的完整收尾。资源必须存活到最后的记录 / 清理完成。

**原文明确**：[核心文档](../../projects/deepseek-harness/docs/subsystems/core.md)把 AgentHandle 的 disposer 当作所有者能力；持有普通 Agent 引用不等于持有对等的销毁权。loop provider 卸载也承担它所创建对象的结构性所有权。

## 4. 为什么这么做：按生命周期划分状态与所有权

**原文明确**：[核心说明](../../projects/deepseek-harness/docs/subsystems/core.md)让消费者依赖 Agent 接口而非默认 loop，并规定拥有 handle 的一方承担完整释放；[防御性模式](../../projects/deepseek-harness/docs/defensive-patterns/general.md)反对用共享运行状态推断单条消息结果。

### 4.1 Driver 与 StreamAttempt 分开

| 层次 | 使用或保存的状态 | 生命周期 |
|---|---|---|
| ReactLoopAgent 实例 | Session、Inbox、scope、phase、activityDone | 创建到 dispose，可经历多次执行活动 |
| Driver 活动 | 使用同实例的 inbox / phase 推进 turn / step，传播取消 | 一次 kick 调用链，跨多次模型调用，结束后实例可仍存在 |
| StreamAttempt | 当前尝试的 chunks、内容块、usage、finish | 每次模型尝试新建 |

**分析**：失败重试时需要丢开失败尝试的临时组装状态，但保留当前 turn / step 和已接纳输入。StreamAttempt 是每次尝试的组装对象；Driver 活动由同一个 ReactLoopAgent 的方法和字段实现，不是另开线程或进程。

### 4.2 whenIdle 不充当单条消息的完成凭据

一个 running 区间可能处理多条 followup、steer 和注入上下文。idle 也可能因为错误、取消而到达。

所以入口必须说明它拥有哪个执行区间，再解释该区间的输出；不能仅等一次 idle 就断言“指定消息成功”。

### 4.3 核对设计时遇到的原文差异

固定版本的 [LLM 文档](../../projects/deepseek-harness/docs/subsystems/llm-streaming.md)有“恢复打开新 turn”的描述，[core 文档](../../projects/deepseek-harness/docs/subsystems/core.md)也有“失败 step 关闭后恢复”的表述；它们与[生命周期图](../../projects/deepseek-harness/docs/agent-lifecycle.md)及本次核对的实现不一致。

**本课程采用源码行为**：[Driver](../../projects/deepseek-harness/packages/core/agent-loop/src/agent.ts)的 `step()` 先固定 turn / step，再在内部循环处理 attempt；request-error 返回 retry 后执行 `continue`。因此这里是**同 step 重试**。这是定位到的文档差异，不推断整份文档都不可用，也不修改上游。

## 5. MVP 与验收

- 先有单 Driver、顺序工具执行和 followup 闭环，再增补 steer、重试和并发。
- 接纳点、取消点和退出点要显式；测试使用可控 barrier，不用 sleep 猜顺序。
- 验收 F03～F05、D01、D04～D06：输入不重复、重试不重接纳、取消不启动新工作、清理不早于退出。见[统一清单](../2026-10-17_03_harness_acceptance_checklist.md)。

<details>
<summary>原文与源码入口</summary>

- [核心中文](../../projects/deepseek-harness/docs/subsystems/core.zh.md)：创建与所有权、Agent 句柄、拦截决策。
- [生命周期图中文](../../projects/deepseek-harness/docs/agent-lifecycle.zh.md)：普通、拒绝、失败与重试分支。
- [Inbox](../../projects/deepseek-harness/packages/core/agent-loop/src/inbox.ts)、[Driver](../../projects/deepseek-harness/packages/core/agent-loop/src/agent.ts)、[StreamAttempt](../../projects/deepseek-harness/packages/core/agent-loop/src/assistant-stream.ts)。
- 这里只做了静态核对，没有执行并发或取消测试。

</details>
