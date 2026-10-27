# 第 2 站：消息、事件与请求

[专题入口](README.md) · 上一站：[系统架构](2026-10-17_01_system_and_plugins.md) · 下一站：[执行循环](2026-10-17_03_loop_and_lifecycle.md)

相邻可选扩展：[2a：Skills 与渐进式披露](2026-10-17_02a_skills_and_progressive_disclosure.md)。从本课的消息与请求继续看技能目录、正文怎样进入 payload；不改变七站主线。

## 1. 是什么：消息、事件、请求和流是四种对象

| 数据 | 回答什么 | 典型字段 / 内容 |
|---|---|---|
| Message | 模型历史中的一条内容是什么？ | id、role、content、source；工具结果另带 toolCallId |
| SessionEvent | 系统发生过什么？ | seq、time、type、data，部分事件还带消息视图操作 |
| GenerateOptions | 这一次怎样请求模型？ | provider、model、messages，以及 tools、maxTokens、signal 等 |
| StreamChunk | 模型这一次陆续返回了什么？ | block-start、delta、block-end、usage、finish |

**Session 事件包含的事实比模型输入多；内部请求也不等于 provider 的 HTTP JSON。**

同一份记录可以服务模型历史、界面、恢复、统计，但它们不是同一种视图。

### 1.1 消息中的四个关键点

- **role 表示消息角色**：system、developer、user、assistant、tool。
- **content 是块数组**：一条 assistant 消息可以包含文字和多个 tool-call，不一定是最终回答。
- **source 表示生产者**：模型、用户、工具或具名插件；程序注入的内容可以是 user 角色，但来源不是用户。
- **callId 表示调用关联**：结果必须回答对应调用，不能只靠“上一条消息”配对。

source 提供来源信息，不自动授予权限。文件内容被放进消息，不代表它能改写执行策略。

## 2. 为什么要做：既要让模型继续，也要解释系统发生过什么

- 模型只需要当前相关内容；排障还需要失败尝试、时间和边界，不能共用一份未区分用途的数据。
- 多个工具可能同时返回；没有 callId，就难以可靠确认结果回答的是哪个调用。
- 只有最终文本，无法解释当时的模型配置、工具声明和上下文变化，也无法可靠恢复。
- UI 希望立即显示片段，持久记录却必须表达完整结算；不区分会把“看见过”误当“已完成”。

## 3. 怎么做：从消息往返到请求与流的结算

### 3.1 一个完整工具往返

下面是**自拟的内部 Message 数据示例**，不是原始 SessionEvent、HTTP 请求或真实运行日志。工具名 `read_doc` 是教学工具，不声称是上游内置工具；ID 是示意值。

```json
[
  {
    "id": "u1",
    "role": "user",
    "content": [{"type": "text", "text": "项目要求哪个 Node.js 版本？"}],
    "source": {"kind": "user"}
  },
  {
    "id": "a1",
    "role": "assistant",
    "content": [
      {
        "type": "tool-call",
        "id": "c1",
        "name": "read_doc",
        "arguments": "{\"path\":\"C:\\\\mvp-fixture\\\\README.md\"}"
      }
    ],
    "source": {"kind": "model", "provider": "mock", "model": "scripted"}
  },
  {
    "id": "t1",
    "role": "tool",
    "toolCallId": "c1",
    "content": [{"type": "text", "text": "This fixture requires Node.js 24."}],
    "source": {"kind": "tool", "callId": "c1"}
  },
  {
    "id": "a2",
    "role": "assistant",
    "content": [{"type": "text", "text": "根据 README.md，这个示例项目要求 Node.js 24。"}],
    "source": {"kind": "model", "provider": "mock", "model": "scripted"}
  }
]
```

这里的版本要求是**测试文件的虚构内容**，不是 DeepSeek 的真实环境要求。

关键变化：

| 时点 | 模型请求中的消息片段 |
|---|---|
| 请求 1 发出前 | u1 |
| 请求 1 成功返回 | 得到 a1；尚无工具结果 |
| read_doc 完成后，请求 2 发出前 | u1、a1、t1 |
| 请求 2 成功返回后 | 记录 a2，形成这次回答 |

表中省略系统提示和其他请求配置。注意 `arguments` 是 **JSON 字符串**；工具结果是独立 tool 消息，不是另一个 tool-call 块。

### 3.2 从事实到请求

- **记录事实**
  - 用户被接纳、模型响应结算、工具调用 / 结果及 turn / step 边界进入 Session。
  - 关键设计点：事件的 seq 是日志位置，不是 Message ID，也不是 callId。
- **派生模型历史**
  - 消息类事件按视图规则进入 `deriveMessages()`；边界和失败 attempt 等用于记录，不变成普通对话消息。
  - 关键设计点：不是对原始日志做一次角色筛选就够了；系统提示更新、替换和插件投影也影响最终历史。
- **构造并发送请求**
  - Driver（此处指 ReactLoopAgent 实例内部的一次异步执行活动，不是独立类实例）结合历史、模型配置和工具声明形成内部请求；adapter 再转换为 provider 格式。
  - 关键设计点：API key、URL、取消 signal 属于执行 / 传输控制，不是要塞进模型正文的内容。

### 3.3 哪些记录进入模型历史

| 记录 | 对模型历史的作用 |
|---|---|
| 有效 system / user / assistant 消息、tool result | 按视图规则形成消息 |
| turn / step 边界 | 不形成消息 |
| assistant/attempt、llm/retry | 记录失败 / 恢复事实，不形成普通消息 |
| 空 content 的 assistant/message | 保留结算证据，但不生成空的模型历史消息 |

完整请求可追溯还需要 request header、工具历史及相同的投影定义；仅保存聊天文字不够。

### 3.4 流式接收与消息结算

- **接收**：多个块的 delta 可以交错，index 标识它属于哪个块。
- **组装**：block-end 提供完整内容块；usage 在 finish 之前，finish 后不再有 chunk。
- **结算**：汇总为 assistant message，或无消息表面的失败 attempt；实时 chunk 与已提交结果分开。

直观例子：界面已经显示“根据文件……”，随后连接断开。这只能证明看过片段，不能证明得到完整回答，也不能证明片段已持久保存。

## 4. 为什么这么做：一个事实来源，多种受约束的视图

**原文明确**：[Session 说明](../../projects/deepseek-harness/docs/subsystems/session.md)规定派生历史的缓存与冻结，使消费者不能借投影修改历史；[总架构](../../projects/deepseek-harness/docs/architecture.md)要求模型可见输入能从日志重建。

**分析**：这样可以解释“当时模型到底看到了什么”，代价是事件、投影和兼容逻辑都要维护，不能只维护一个可随意修改的 messages 数组。

替代方案是单独保存聊天 messages 和运行日志。它更容易起步，但两边可能不同步；若采用它，就要另外证明某次请求与当时日志、配置的一致性。

## 5. MVP 与验收

- 第一版先支持 text、tool-call、tool result；复杂多模态与私有 replay 后置。
- 保留事件与消息分离、稳定 ID、显式配置和失败记录；是否采用完整事件溯源框架另议。
- 验收 F02：第二次请求包含正确的 c1 结果；D02：请求可追溯；D07：失败片段不冒充已完成消息。见[统一清单](../2026-10-17_03_harness_acceptance_checklist.md)。

<details>
<summary>原文与源码入口</summary>

- [LLM 流式输出](../../projects/deepseek-harness/docs/subsystems/llm-streaming.zh.md)：内容块 / 消息、StreamChunk、模型请求、适配器约定。
- [Session](../../projects/deepseek-harness/docs/subsystems/session.zh.md)：事件条目、派生历史、结束原因和持久性。
- [消息类型源码](../../projects/deepseek-harness/packages/llm/llm/src/message.ts)、[流与请求类型](../../projects/deepseek-harness/packages/llm/llm/src/types.ts)、[消息视图](../../projects/deepseek-harness/packages/core/session/src/surface.ts)。
- 本示例只验证数据格式与关联，不是 Harness 的运行证据。

</details>
