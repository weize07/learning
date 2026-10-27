# DeepSeek Harness：沿一次读文件任务追完整执行链

- 学习日期：2026-10-16。
- 阅读阶段：独立项目课，第 2 篇。
- 前置：[项目全貌与核心概念](2026-10-15_09_deepseek_harness_overview.md)。
- 每日概览：[topics_by_date](topics_by_date.md)。
- 固定源码版本：`571902028a75c64c1d767f4ae5fd099485f0ddf0`。
- 证据：已阅读固定版本的入口、Agent 工厂、loop、工具调度与文件读取源码；以下是静态调用链分析与教学示例，不是实际运行日志。未安装项目、连接模型或执行任务。

## 1. 本篇的目标：先理解边界，再追主链

假设用户提出：

> 读取 README.md，说明请求超时配置的单位；请引用依据，不修改文件。

为了让路径最清楚，本篇选择 **headless 入口**，而不是从浏览器按钮和网络协议开始。它创建 Agent、提交任务、等待执行结束，最后把结果写到标准输出。

这不表示 Web 没有 loop，而是先分开两种复杂度：

- 本篇：应用入口如何使用公共 Agent 接口，以及默认 loop 怎样工作。
- 后续：Web / SDK 怎么把输入送到这些接口，怎样订阅状态和输出。

教学场景假设 `read` 已挂载、模型可用、目标文件可读，没有额外消息、子 Agent 或策略干预。真实运行必须确认这些条件；“用户说不修改”本身不是文件只读权限。

先抓住设计的中心问题：**一个不断接收输入、调用不确定模型、执行真实工具的程序，怎样避免把“打算做”误当成“已经做”，并让下一次请求基于明确的事实？**

用四条线组织理解，而不是背函数：

| 设计边界 | 要解决的问题 | 对应部分 |
|---|---|---|
| 待处理输入 / 已接纳历史 | 运行途中收到消息，什么时候进入模型请求？ | inbox、step 边界 |
| 模型建议 / 真实执行 | 谁能允许、拒绝或执行文件操作？ | 工具管线与文件后端 |
| 原始记录 / 本轮模型视图 | 什么保存下来，什么发给模型？ | Session、`deriveMessages()` |
| 执行引擎 / 结果交付 | 同一套 loop 怎样被 CLI、Web、SDK 使用？ | 公共 Agent 接口与入口 |

每节先回答“为什么需要这层”，再看时序和源码。下文的设计理由是根据职责与调用关系解释工程取舍，不声称是作者全部或唯一的动机。

## 2. 先认识 messages，再看完整路径

### 2.1 messages 是一次模型请求的有序对话视图

它不是字符串数组，也不是“只放用户发言的列表”，而是一组带角色、内容和关联信息的消息。

本项目有三层不同的格式：

```text
Session event log：完整的事件记录
        ↓ 推导当前可见历史
Harness Message[]：provider-neutral 的内部消息
        ↓ 适配器转换
Provider wire messages：具体模型 API 的请求格式
```

**本节先讲中间这一层。** 它不等同于某家 API 文档中的 `{role, content: string}`；不能把内部对象不经转换直接提交给所有模型。

### 2.2 一条内部 Message 的骨架

以下对象按固定版本的内部类型展示，ID 和内容均为教学占位，不是实际日志：

```json
{
  "id": "M1",
  "role": "user",
  "content": [
    {
      "type": "text",
      "text": "Read README.md and explain the timeout unit."
    }
  ],
  "source": {
    "kind": "user"
  }
}
```

| 字段 | 回答什么问题 | 注意 |
|---|---|---|
| `id` | 这是哪一条消息？ | 内部稳定身份；不等于工具调用 ID |
| `role` | 消息在对话协议中扮演什么角色？ | 如 system、user、assistant、tool；另有 developer |
| `content` | 这条消息包含什么？ | 内容块数组，可以不止一段纯文本 |
| `source` | 谁产生了这条内容？ | 用户、模型、工具或上下文生产者；不是显示昵称 |

`role` 与 `source` 不是重复字段。例如程序注入的运行上下文也可以使用 user 角色，但来源不是用户本人。`source` 是内部归因信息，不代表 provider 会原样接收这个字段，也不自动授予信任。

先只记两种内容块：

- 文本：`{ "type": "text", "text": "..." }`。
- 调用建议：`{ "type": "tool-call", "id": "...", "name": "...", "arguments": "..." }`。

后者的 `arguments` 在这个内部类型中是**原始 JSON 字符串**，不是已经校验好的 JavaScript 对象；实际执行前还要解析和验证。图片、推理和文件等其他块以后按需展开。

### 2.3 完整往返：用户 → 调用建议 → 工具结果 → 回答

下面是一份极简的最终对话视图。它省略真实运行中可能注入的上下文、动态声明等消息；假设测试文件中确实有示例所示文本，不代表本机文件内容。

```json
[
  {
    "id": "M0",
    "role": "system",
    "content": [{"type": "text", "text": "Use file evidence. Do not modify files."}],
    "source": {"kind": "system-prompt"}
  },
  {
    "id": "M1",
    "role": "user",
    "content": [{"type": "text", "text": "Read README.md and explain the timeout unit."}],
    "source": {"kind": "user"}
  },
  {
    "id": "M2",
    "role": "assistant",
    "content": [{
      "type": "tool-call",
      "id": "call_read_1",
      "name": "read",
      "arguments": "{\"file_path\":\"README.md\",\"offset\":1,\"limit\":40}"
    }],
    "source": {"kind": "model", "provider": "deepseek", "model": "deepseek-chat"}
  },
  {
    "id": "M3",
    "role": "tool",
    "toolCallId": "call_read_1",
    "isError": false,
    "content": [{"type": "text", "text": "README.md, line 12: request_timeout is measured in milliseconds."}],
    "source": {"kind": "tool", "callId": "call_read_1"}
  },
  {
    "id": "M4",
    "role": "assistant",
    "content": [{"type": "text", "text": "The unit is milliseconds, according to README.md line 12."}],
    "source": {"kind": "model", "provider": "deepseek", "model": "deepseek-chat"}
  }
]
```

这里最关键的是：

1. **工具调用仍是 assistant 的内容**：它是模型提出的动作建议，不是已经执行的结果。
2. **工具返回另起一条 tool 消息**：`toolCallId` 对应调用块的 `id`。工具结果的 `source.callId` 也保留这条关联。
3. **调用与结果不能靠“位置看起来相邻”来猜配对**：一条响应可能提出多个调用。
4. **工具错误仍然可以成为 tool 消息**：`isError: true` 表示失败，不应该改写成用户输入或成功事实。
5. `isError: false` 只说明工具结果被标记为非错误，不证明内容无误或业务验收成功。

### 2.4 两次请求分别看到了什么？

| 时刻 | 本例 messages 中有什么 | 尚未发生什么 |
|---|---|---|
| 第一次模型请求 | M0、M1 | 还没有调用建议与文件内容 |
| 第一次响应完成 | 新得到 M2 | 工具尚未执行 |
| 工具执行并记录结果后 | 新得到 M3 | 模型尚未据此生成解释 |
| 第二次模型请求 | M0、M1、M2、M3 | M4 是接下来要生成的结果 |
| 第二次响应完成 | 新得到 M4 | 若正常结束，不必再发第三次请求 |

因此不是“一开始把五条消息都发过去”，也不是模型第一次调用就已经看到文件。

此外，**工具声明 `tools` 与 `messages` 分开提供**：`tools` 告诉模型“可以提什么调用、参数是什么”，`messages` 记录“之前提出过什么调用、实际得到了什么结果”。真正的执行函数不在请求里。

**为什么不维护一个谁都能随便 push 的共享数组？** 因为还存在排队、取消、上下文更新和历史替换。项目选择从 Session 推导请求视图，避免入口、工具、UI 各维护一份互相漂移的历史；代价是需要明确事件、投影与接纳边界，而不只是操作数组。

类型依据：[message.ts](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/packages/llm/llm/src/message.ts)、[types.ts 中的内容块](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/packages/llm/llm/src/types.ts)。短 ID 为展示简化；生产代码通过相应构造 helper 创建消息。

### 2.5 完整的内部请求：不只有 messages 和 tools

固定版本的 `GenerateOptions` 共声明以下 13 个字段。`provider`、`model`、`messages` 为必填；可选字段是否实际出现，由调用来源、配置和适配器决定。

| 字段 | 类型 / 示例 | 职责 |
|---|---|---|
| `provider` | string | Harness 内部路由，选择注册的适配器 |
| `model` | string | 本次请求的模型 ID |
| `messages` | `RequestMessage[]` | 本次有序对话视图 |
| `tools` | `ToolSchema[]`，可选 | 模型可见的工具声明，不含执行函数 |
| `toolHistory` | `ToolHistory`，可选 | 初始工具声明及动态工具更新的历史，供适配器投影；不是工具返回历史 |
| `system` | string，可选 | 供手工 one-shot 调用使用的系统提示；loop 生成的请求不设置它 |
| `reasoningEffort` | 适配器定义的 effort ID，可选 | 本模型的推理强度控制，不保证所有路由支持相同值 |
| `temperature` | number，可选 | 采样控制，实际支持和生效方式取决于模型与协议 |
| `maxTokens` | number，可选 | 本次生成的输出 token 上限，不是整个任务预算，也不是上下文窗口 |
| `stop` | string[]，可选 | 生成终止字符串，不等于 turn / 任务的完成条件 |
| `signal` | `AbortSignal`，可选 | 本地取消控制，交给调用 / 传输层，不作为 JSON 提示内容 |
| `sessionId` | 会话身份，可选 | 路由、回放或传输元数据；不是让模型记忆会话的魔法参数 |
| `purpose` | `compaction` 或 `session-title`，可选 | 辅助模型调用的分类，普通对话不设置 |

语义上可以分成四组：**路由（调用谁）＋内容（看到什么）＋生成控制（怎样生成）＋运行控制 / 归因（怎样取消、属于哪次调用）**。

loop 的 `buildRequest()` 主路径组装的是下列结构。这里是 JavaScript 对象示意，变量代表前面已经构造的值，不是可直接执行的完整脚本：

```javascript
const request = {
  ...header.config,
  messages: session.deriveMessages(),
  toolHistory: session.toolHistory(),
  ...(header.tools === undefined ? {} : { tools: header.tools }),
  sessionId: session.id,
  signal
};
```

`header.config` 提供 `provider`、`model`，以及存在时的 `reasoningEffort`、`temperature`、`maxTokens`、`stop`。实际源码还处理冻结与请求归属标记，不能把这个示意当成可替换实现。

两点容易混淆：

- **loop 请求不设置顶层 `system`，不等于没有系统提示。** 系统提示已经通过 `messages` 中的 system 消息表达；下一层适配器仍可能把它转成 HTTP body 顶层的 `system`。
- `RequestMessage` 比持久化的 `Message` 稍宽：还允许仅用于一次请求、不带持久身份与 source 的 user 输入。前面介绍的 loop 主链仍从 Session 推导带身份的消息。

### 2.6 真正发出的 HTTP 请求：由适配器转换

不能把 `GenerateOptions` 直接 `JSON.stringify` 后当成通用模型 API 请求。这里展示固定版本的 DeepSeek Messages 适配器，不代表其他 DeepSeek 接口或 OpenAI Chat Completions 的字段规则。

以第一次请求、开启 high effort、简化工具声明为例，序列化后的基础 body 可以是：

```json
{
  "model": "deepseek-chat",
  "stream": true,
  "max_tokens": 1024,
  "thinking": {"type": "enabled"},
  "output_config": {"effort": "high"},
  "system": "Use file evidence. Do not modify files.",
  "messages": [
    {
      "role": "user",
      "content": [
        {"type": "text", "text": "Read README.md and explain the timeout unit."}
      ]
    }
  ],
  "tools": [
    {
      "name": "read",
      "description": "Read a UTF-8 text file.",
      "input_schema": {
        "type": "object",
        "properties": {
          "file_path": {"type": "string"},
          "offset": {"type": "number"},
          "limit": {"type": "number"}
        },
        "required": ["file_path"]
      }
    }
  ]
}
```

这是教学请求，不是实测抓包，工具声明也不是官方 read schema 的逐字复制；实际请求可能另有插件贡献的顶层扩展、图片和动态工具声明等内容。

关键映射：

| 内部字段 / 概念 | 该 DeepSeek Messages 适配器的处理 |
|---|---|
| `provider` | 用于选择路由，不作为同名基础 body 字段 |
| system 角色消息 | 提取初始系统提示；根据路由能力处理后续更新 |
| `maxTokens` | `max_tokens` |
| `stop` | `stop_sequences` |
| `reasoningEffort` | 非 off 值映射为启用 thinking 与 `output_config.effort`；off 则禁用 thinking |
| 工具声明的 `parameters` | `input_schema` |
| assistant 的 `tool-call` | `tool_use` 内容块，参数字符串解析为 input 对象 |
| 内部 tool 角色结果 | user 角色中的 `tool_result` 内容块，不保留内部 tool 角色的外形 |
| `signal` | 控制本地 fetch，不序列化进 body |
| `sessionId` | 此适配器写入会话相关请求头，也可供扩展逻辑使用 |

因此第二次请求的 wire messages 也不能原样照抄第 2.3 节的内部数组。内部 `toolCallId` 在该协议中用于生成 `tool_result.tool_use_id`。

HTTP 请求本身还有 URL、method 和 headers。该适配器向配置 API root 的 `/messages` 发 POST，附带鉴权、JSON / SSE 与协议版本等 headers；`stream: true` 由该序列化器写入 body。`GenerateOptions` 本身没有 `stream` 字段，因为调用接口已经选择了流式执行。

API 地址、凭据解析、代理、流空闲超时等来自适配器及连接配置，不要求每次都塞进 `GenerateOptions`，也不应放进 messages。凭据发送到服务端做鉴权，与成为模型提示内容是两回事。

支持边界：并非所有 provider 都支持这些控制；本版本内部通用请求没有声明 `top_p`、`top_k`、`tool_choice` 或 `response_format`，不能因为某家 API 支持就直接假定可在这里添加。DeepSeek 适配器文档还说明开启 thinking 时传入的 temperature 会被服务端忽略；不应默认所有数值控制都同时生效。

源码依据：[GenerateOptions](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/packages/llm/llm/src/types.ts)、[调用配置](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/packages/llm/llm/src/call-config.ts)、[DeepSeek 序列化](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/packages/llm/llm-deepseek/src/serialize.ts)、[HTTP 传输](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/packages/llm/llm-deepseek/src/adapter.ts)。

### 2.7 thinking 保存在哪里？先区分内容、计数与控制

内部请求没有顶层 `thinkingTokens` 字段。“thinking token 的存储”可能指三种不同需求：

| 需求 | 相关结构 | 方向 |
|---|---|---|
| 控制是否 / 多大程度推理 | 请求中的 `reasoningEffort`，以及适配器映射出的 thinking 配置 | 本次请求的控制参数 |
| 保存 provider 返回的推理内容 | assistant 消息的 `content` 中，`{ type: 'reasoning', text: string }` | 响应 → 会话 → 可能成为后续请求历史 |
| 统计推理用了多少 token | `TokenUsage.reasoningTokens?: number` | 响应 usage / 会话记录；不是对话内容 |

例如，下列内部 assistant 消息把推理块与调用建议保存在一起。推理文本是占位字符串，不是本次实际模型思考，也不表示 provider 必然暴露完整思考过程：

```json
{
  "id": "M2",
  "role": "assistant",
  "content": [
    {"type": "reasoning", "text": "<provider-returned reasoning text>"},
    {
      "type": "tool-call",
      "id": "call_read_1",
      "name": "read",
      "arguments": "{\"file_path\":\"README.md\",\"offset\":1,\"limit\":40}"
    }
  ],
  "source": {"kind": "model", "provider": "deepseek", "model": "deepseek-chat"}
}
```

在支持的路径上，流式 `reasoning-delta` 会组装为 reasoning 块，随后与完整响应一起记入 `assistant/message`。这是 **provider 返回的文本**，不是 token ID 数组、模型隐藏状态、KV Cache 或所有内部计算的完整保存。

下一次请求从 Session 推导历史时，保留下来的 assistant 消息可以包含这些块；适配器再决定如何转换。本版本的 DeepSeek Messages 序列化器把 `reasoning` 转成原生 `thinking` 块，把 `tool-call` 转成 `tool_use`。相应 `source.replayState` 可另外保存模型绑定的签名等回放元数据；这不是另一份推理正文，也不能假定跨模型通用。是否保留、发送、展示都要分别判断，其他 provider 也可能只返回摘要、不可读的状态或根本不返回推理正文。

`TokenUsage.reasoningTokens` 虽然在通用类型中存在，但不能假定每个适配器都填充。本版本已核对的 DeepSeek Messages `translate.ts` 只映射 input、output、cache read / write 等计数字段，没有单独填入 reasoningTokens；因此不能编造一个独立推理 token 数，也不能用 reasoning 文本长度替代 provider 权威计数。usage 位于响应事件的数据中，不是 assistant 的文本块，也不自动作为下一次请求的生成参数发送。

最后，“记入 Session”仍不等于默认落盘，持久化依赖后端。即使落盘，也只是保存已获得的内容与元数据，不能据此恢复服务器内部推理状态。

依据：[块与 usage 类型](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/packages/llm/llm/src/types.ts)、[DeepSeek 流式转换](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/packages/llm/llm-deepseek/src/translate.ts)、[原生回放元数据](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/packages/llm/llm-deepseek/src/replay.ts)。

### 2.8 再看总体时序

![一次读文件任务的主链](images/deepseek-harness-task-lifecycle.svg)

图中箭头标签主要用英文方法名、事件名和结果字段，便于对照源码；标题与解释保留中文。总图用于看阶段，细节以第 3～9 节的局部时序为准。部分总图标签概括同一阶段的动作，不表示箭头上的所有动作都由接收方完成，例如 inbox 的认领由驱动处理，不是 Session 的方法。

局部图中实线表示调用或通知，虚线表示返回、完成或数据流；同一生命线上的折返箭头表示本对象内部动作。`[if ...]` / `[optional]` 是条件说明，不是需要原样执行的代码。图中为可读性折叠了部分中间层，不是可以照抄执行的调用脚本。

主链可压缩为：

```text
入口创建 Agent，并提交用户消息
    ↓
消息进入 inbox，唤醒 driver
    ↓
打开 turn，认领输入，组装 prompt 与可见工具
    ↓
准备模型路由，提交本步消息，推导请求历史
    ↓
模型流式返回：需要 read 工具
    ↓
记录 assistant 消息，经工具策略与执行流程读取文件
    ↓
记录关联的工具结果
    ↓
下一步重新构造模型请求
    ↓
模型给出解释，本次没有工具调用
    ↓
收束 turn，Agent 回到 idle，入口读取结果并输出
```

不要把“下一步”理解成模型在自己调用自己。**循环由 Harness 程序驱动；模型每次返回一次响应。**

## 3. 入口先创建执行对象，再提交任务

**设计问题：为什么要先创建 Agent，而不是入口直接调用模型？**

一次任务需要明确的会话身份、工作目录、模型选择和资源生命周期。若 CLI、Web、SDK 各自临时拼一套执行对象，创建和清理规则很容易分叉。因此入口依赖公共 Agent 接口，具体工厂负责完整准备与发布。

设计重点是**先准备好，再对外可用**：监听器可能需要连接工具或配置作用域，若尚未就绪就允许模型执行，首次请求的能力集合可能缺失。相应代价是创建过程需要异步等待、处理失败和清理，而不是一次简单构造。

![3：Agent 创建与发布](images/deepseek-harness-detail-03-create.svg)

### 3.1 headless 入口做什么？

入口代码在 `packages/bundle/headless/src/index.ts` 的 `run()`。

正常的新任务路径是：

1. 等待应用插件装配完成，避免使用只加载了一半的工具或适配器。
2. 读取任务文本；空任务明确报错。
3. 取得默认模型选择和工作目录。
4. 调用 `agents.create(...)`，取得 handle 中的 `agent`。
5. 安装本次运行所需的输出和错误观察。
6. 用 `agent.followup(...)` 提交用户消息。
7. 等待执行收束，再输出结果。

有指定 session 身份时还存在恢复 / 采用已有会话的分支，本篇只追新建路径。这里的“入口”是程序代码，不是模型的一部分。

### 3.2 谁真正创建 Agent？

`ctx.agents` 是公共入口和活跃对象注册服务；默认 loop 插件把自己注册为工厂。公共调用再委托到默认实现，工厂的 `createAgent()` 准备 Session，随后 `setupAndPublish()` 完成初始化和发布。

工厂最终构造具体对象 `ReactLoopAgent`。这个名字在本篇只需理解为：

> 默认的“调用模型 → 执行工具 → 继续”的 Agent 实现。

创建并不是只做一次 `new`。源码还会准备作用域、执行 setup、登记 Session 与 Agent、通知创建监听器，然后才放行初始化期间排队的工作。这样，依赖创建通知完成连接的能力有机会在首次任务前就绪。

两点边界：

- 创建 Agent 不等于已经发起模型请求；还需要输入及相应驱动条件。
- Session 在内存中准备；是否取得持久化写句柄，取决于是否装配持久化后端。不能把 `append` 一概解释为已同步写盘。

### 3.3 为什么 enter 之后还要 announce？

`enter` 与 `announce` 不是重复注册，分别处理“登记状态”和“触发生命周期”。工厂发布阶段的核心顺序是：

```javascript
detachSession = agent.ctx.sessions.enter(session);
detachAgent = loopCtx.agents.enter(agent, parentAgent);
agent.ctx.sessions.announce(session);
await loopCtx.agents.announce(agent, source, abort.signal);
```

这是源码发布片段的顺序示意，`detachSession` / `detachAgent` 属于外围已准备的清理流程，不是完整可运行脚本。

| 操作 | 改变什么 | 为什么需要 |
|---|---|---|
| `sessions.enter(session)` | 登记活跃 Session、接入发布机制、返回 detach | 让会话可查，并取得移除该精确对象的清理能力 |
| `agents.enter(agent, parentAgent)` | 登记活跃 Agent 与运行时归属、返回 detach | 让执行对象可查；检查身份与重复注册 |
| `sessions.announce(session)` | 发出一次 `session/created` 通知 | 让关心会话的模块响应；不是写一条聊天消息 |
| `await agents.announce(...)` | 串行等待 `agent/created` 监听器 | 让需要异步准备的能力在放行输入前完成初始化 |

**为什么不在 enter 里面立刻发事件？** 事件监听器能够立即运行、查找其他对象、注册资源甚至失败。如果登记 Session 时就通知，监听器可能查不到尚未登记的 Agent；如果登记 Agent 时就通知，调用方可能尚未来得及保存 enter 返回的 detach。工厂选择先把两个注册关系与清理能力准备好，再开始对外通知，减少半初始化状态和失败时无法撤销登记的问题。

这里“先登记、后通知”不等于 enter 后对象仍完全不可见：注册服务的查询已经能取得它。它也不是跨所有插件的原子事务。announce 用于明确发布事件的时机和等待边界；通知已经到达部分监听器后发生失败，不能假装通知从未发生。

**为什么通知两次？** 因为“会话存在”和“执行对象创建”是不同的生命周期事件，关注者可以不同。Session 侧关心记录容器，Agent 侧关心可执行对象及其作用域；例如已核对的浏览器 MCP 集成会在 `agent/created` 阶段等待连接与工具发现，而不要求 Session 核心理解浏览器。

两种 announce 的等待语义也不同：

- `sessions.announce()` 是同步通知。同步抛错可以使发布失败；监听器若返回 Promise，该 Promise 的拒绝会被记录，但这里不等待它，也不把它当作异步就绪屏障。
- `agents.announce()` 等待串行创建监听器，监听器失败会拒绝该调用。调用方等待它，不等于等待 Agent 执行完用户任务。

创建失败时，由外围生命周期流程调用 detach 等清理。已开始宣布的对象会配对 disposal 通知；仅登记但尚未宣布的对象可以撤销登记而不虚构一次创建 / 销毁通知。清理仍不等于撤销插件曾产生的外部副作用。

可以把设计概括为：**先建立可查的对象关系与可回收的所有权，再通知相关模块，最后等待需要的初始化屏障。**

依据：[Agent 注册与宣布](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/packages/core/agent/src/index.ts)、[Session 登记与宣布](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/packages/core/session/src/index.ts)、[工厂发布顺序](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/packages/core/agent-loop/src/index.ts)。

### 3.4 图里的 await serial agent/created listeners 是什么？

这是三个概念的组合，不是一条完整代码：

- `agent/created`：事件名，表示本次 Agent 创建的生命周期通知。
- listeners：事先订阅该事件、且匹配本次作用域的回调函数；不是额外的 Agent 或线程。
- `await serial`：按注册顺序等待这些回调，前一个完成后才进入下一个；当前 announce 等待这一过程，而不是发出通知就立刻返回。

概念示例，`initializeBrowser` 和 `initializeOtherCapability` 是说明用的函数，不是这里的实际仓库符号：

```javascript
ctx.on('agent/created', async ({ agent }) => {
  await initializeBrowser(agent);
});
ctx.on('agent/created', async ({ agent }) => {
  await initializeOtherCapability(agent);
});
```

插件加载时执行 `ctx.on(...)` 只是登记回调，不会立即执行函数体中的初始化。创建 Agent 并宣布事件时，正常顺序才是：

```text
agents.announce(agent)
    → start listener A
    → await A completes
    → start listener B
    → await B completes
    → announce resolves
    → creation continues; queued work may be released
```

这里的等待是“执行前的准备完成”，不是等待模型回答。若 A 抛错或拒绝，正常顺序中不会继续执行 B，announce 失败，由外围创建流程处理清理。已经完成的外部操作不会自动回滚。

`await` 暂停的是这条异步创建流程的后续步骤，不是把整个 Node.js 进程阻塞住。其他独立异步工作仍可运行；listener 内部也可以按自身设计并发执行子任务，serial 约束的是这些监听回调之间的调度。

## 4. 输入不是直接拼到一个 messages 数组里

**设计问题：模型正在生成时，新消息属于正在执行的请求，还是下一次请求？**

已经发出的请求不能靠修改本地数组就被追溯改变；如果每次输入都另开 loop，又会竞争同一会话和工具。这里用 inbox 表达“待处理”，由单个 driver 在明确边界认领，随后才接纳进模型历史。

设计重点是**把“已收到”和“已进入请求”分开**。代价是用户的新输入可能需要等待边界，必须通过状态或交互明确告知，而不是假装模型已经看到。

![4：消息入队与边界认领](images/deepseek-harness-detail-04-inbox.svg)

### 4.1 `followup()`：放入下一轮队列并唤醒

默认实现中的调用关系是：

```text
followup(message)
    → send(message, 'next-turn', true)
    → inbox.splice(...)
    → wakeDriver(...)
    → kick()
    → turn()
```

这里 **inbox 是待接纳的输入队列**，不是模型已经看到的历史。

`wakeDriver()` 在空闲时预留运行状态并启动驱动；已有活跃驱动时不会为每条消息再开一个并行 loop。`kick()` 负责持续处理需要继续的 turn，最后收束状态。

输入入队本身有会话事件记录，但这与“已经作为 `user/message` 纳入本次模型历史”是两回事。

### 4.2 三种输入方式，先认识差别

| 方法 | 目标 | 是否主动唤醒 |
|---|---|---|
| `followup()` | 排入下一 turn，把这条普通输入作为后续独立一轮处理 | 是 |
| `steer()` | 排入最近可接纳的 step 边界，适合影响正在进行的任务 | 是；空闲时也可启动一轮 |
| `inject()` | 排入下一 step 边界，补充模型可见上下文，不单独启动工作 | 否；空闲时保留待处理 |

这不表示输入能插进已经发出去的模型请求中间。边界接纳是程序动作；取消后唤醒等特殊情况还会重新归类到下一 turn，本篇不展开。

### 4.3 三种方法的设计目的，不是三种文本分类

它们本质上是 `send(message, target, wakeup)` 的三种组合：

```javascript
followup(message); // send(message, 'next-turn', true)
steer(message);    // send(message, 'next-step', true)
inject(message);   // send(message, 'next-step', false)
```

这里的代码是对应关系示意，不是让调用方连续提交三次相同消息。

- **followup：任务排队。** 当前在解释 README，用户又要求“之后检查配置示例”，希望保留独立交互轮次，而不是把它当成当前 step 的修正。
- **steer：尽早转向。** 当前正在查错文件，用户说“改看 client README，不要继续看 server README”，希望最近一次可接纳输入的请求使用这个方向。它不取消已经发出的模型请求，也不撤销已执行的工具。
- **inject：附带上下文，不主动制造任务。** 插件通知“工作目录已经改变”，希望下次请求带上这个事实，但不希望仅因目录改变就在空闲时启动一次模型调用。

inject 不主动唤醒，不等于绝不会增加请求：若 Agent 已在运行，这条待处理上下文可在后续边界被认领，也可能让本轮继续。反之，若本次 pre-step 已经认领完输入，刚注入的消息可能赶不上该次请求。取消、拒绝或卸载也可能影响尚未处理的消息，不能承诺立即或必然送达。

三种方法都接收带身份和来源的 `UserMessage`。这里的 user 角色不等于一定来自真人；inject 也不等于 system prompt、永久 Memory 或更高权限。

### 4.4 用户按发送时，谁选择？

**选择由调用 Agent 接口的程序层完成，不是 loop 先让 LLM 阅读文本，再让模型决定用哪种方法。** 不同入口可以有不同规则，三个方法也不是必须同时展示给用户的按钮。

已核对本版本 Web / Desktop 共用的输入策略与 Host 命令：

```text
用户提交手势 + 当前 running 状态 + 设置 + 传输能力
    ↓
前端 resolveSubmitMode(...) 选择 queue / steer
    ↓
提交 request.mode
    ↓
Host SessionCommands.prompt(...)
    ├─ mode === 'steer' → agent.steer(message)
    └─ otherwise       → agent.followup(message)
```

普通输入框这条路径没有第三种 inject 模式。具体规则是：

| 状态 / 手势 | 前端选择 |
|---|---|
| Agent 不在 running，或传输不支持 steering | queue |
| running 且支持 steering，普通 Enter / 主发送按钮 | 按 `busyEnter` 设置；默认 queue，也可设为 steer |
| running 且支持 steering，Cmd / Ctrl 加速提交手势 | 与当前 `busyEnter` 设置相反 |

这不是根据“稍后”“马上”等自然语言关键词分类。用户表达的任务语义与发送模式可能不一致，产品需要让模式可理解、可选择；代码作者也可以设计另一种交互，但不能把它当成本版本事实。

前端判断的是当时看到的状态快照，发送到 Host 时状态可能已经变化。最近实际接纳的边界仍由 Agent driver 决定，不能由 UI 保证。SDK 也不必复制 GUI 的规则：该版本 JSON-RPC server 的普通 `prompt()` 路径直接调用 `followup()`，headless 提交任务同样使用 followup。

### 4.5 inject 通常由谁触发？怎样选择才合适？

已核对的具体调用方是 working-directory 插件：提交目录变化事件后，它构造来源为 `working-directory` 的通知，再调用 `agent.inject(notice)`，把“目录已变化”的事实排入后续上下文，而不凭这条通知单独唤醒空闲 Agent。

对于自建入口或插件，可以先按下面的问题选择：

1. **这是需要独立处理的后续用户请求吗？** 通常用 followup。
2. **这是要尽早影响当前任务、并希望空闲时也启动处理的输入吗？** 通常用 steer。
3. **这只是下次模型运行时应该知道的上下文，单独出现不应触发生成吗？** 通常用 inject。
4. **需要立刻停止危险操作吗？** 另走取消 / 授权控制，不能依赖 steer 在之后某个 step 才被读到。

以上是选择框架，不是自动内容分类器。后台消息若要求立即唤醒处理，单用 inject 就不够；也不能因为调用了 inject 就绕过来源、权限或提示注入防护。

固定版本依据：[公共输入契约](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/packages/core/agent/src/runtime-types.ts)、[前端发送策略](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/packages/client/ui-conversation/src/client/input/submission-policy.ts)、[busyEnter 默认值](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/packages/client/ui-conversation/src/submission-settings.ts)、[Host 路由](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/packages/api/session-controller/src/commands.ts)、[SDK prompt](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/packages/sdk/server/src/server.ts)、[目录变化通知](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/packages/session/working-directory/src/index.ts)。

### 4.6 “最近可接纳的边界”和“下一 step 边界”有区别吗？

这里不是两个不同机制，二者都在描述 `next-step` 队列。更精确的说法是：**消息由之后第一次能认领到它的 `preStep()` 处理。** “最近可接纳”用于避免把“下一 step”误解为“从界面上看，下一次尚未发出的模型请求一定能收到它”。

关键时刻是 inbox 的 `claim()`，不是模型请求发出的时刻，也不是一个笼统的 `step/end` 通知：

```text
消息到达
    → preStep(k+1) 执行 claim
    → 本次可认领该消息，正常路径进入 request(k+1)

preStep(k+1) 已执行 claim
    → 消息才到达，但 request(k+1) 可能尚未发出
    → 消息留在 next-step，等待后续认领，正常连续路径为 request(k+2)
```

因此，同一个 k+1 step 可以处于“尚未发模型请求，但已错过普通队列接纳”的准备阶段。`preStep()` 在认领后还要组装提示、经过前置决策，之后才到 `step/start` 和模型路由准备；不能仅凭“模型还没开始输出”判断能插入当前输入。

以上是默认 loop 正常继续路径的解释，不保证永远存在 k+2：取消、拒绝或停止会影响后续处理。空闲时的 steer 可以启动新 turn，而 inject 不会主动唤醒。它们不代表随时中断当前 step，也不修改已经构造或发出的请求。

## 5. 一个 step 的模型输入怎样组成？

**设计问题：谁决定本次模型实际看到的输入与工具？**

不是让入口把所有东西直接交给 provider，而是让 step 形成一个明确的请求快照：认领输入、组装可见能力、处理前置决策、确定适配器，再按该适配器的能力接纳 prompt 与消息，最后推导请求。

设计重点是**构造一次自洽的请求**，并让扩展发生在命名边界，而不是插件在发送途中随意改共享数组。代价是组装阶段较多，且“组装好了”不等于“消息已经提交”。

![5：从组装到请求快照](images/deepseek-harness-detail-05-request.svg)

图从 `preStep()` 已取得 claimed messages 开始；`agent/pre-step` 可以拒绝，图只展开正常接纳分支。消息提交发生在模型路由准备之后，不应把两步交换理解。

### 5.1 先打开 turn，再准备 step

`turn()` 先追加 `turn/start`，随后调用 `preStep()`：

1. 从 inbox 认领相应输入。
2. 调用 `systemPrompt.assemble(...)`，取得提示组装结果及可见工具。
3. 经过 `agent/pre-step` 扩展点，决定是否进入 step，并组织运行上下文。

“认领”可以理解为驱动取得这批输入的处理权，不是把它们立即发送给模型。若前置决策拒绝，或首次接纳的消息为空，可能结束 turn 而不发起模型调用。

正常接纳后才追加 `step/start`，进入 `step()`。

### 5.2 绑定模型，再提交本步消息

`step()` 先调用 `prepareRequest()`：

- 取得本次 provider / model 和请求参数。
- 经过 `agent/request` 扩展点。
- 调用 `llm.prepareCall(...)`，解析适配器及其默认参数和能力。

随后，loop 按适配器能力处理 system prompt，把本步接纳的用户消息、运行上下文写入会话，再由 `buildRequest()` 构造请求。

主路径顺序是：

```text
准备模型路由
    → 处理 prompt 与输入接纳
    → 记录必要的请求配置
    → session.deriveMessages()
    → 加上本步可见 tools、模型参数与取消信号
```

源码的内部请求不是固定的 `{system, messages}` 双字段结构：system prompt 也通过会话中的消息推导处理。具体 provider 的线协议转换属于适配器层。

### 5.3 工具定义与工具实现分在哪里？

- 发给模型：工具名称、描述、参数 schema 等声明。
- 留在程序：工具执行函数、文件后端和策略检查。

因此，模型知道存在 `read`，不等于模型自己能访问文件。真正读取发生在后面的执行链。

## 6. 模型返回工具调用，并没有执行文件读取

**设计问题：流式片段、完整模型响应与真实操作，为什么不混在一起？**

流里可能逐步出现文本和调用参数，单个片段不一定构成完整调用。项目用 `AssistantStreamAttempt` 汇集片段、推送实时帧，再把响应定型成带内容块的消息；正常路径随后才调度工具。

设计重点是**实时展示不等于执行承诺**。这样能够响应用户，又不因看见半段参数就启动动作；代价是要管理 start / chunk / end、成功提交与放弃等状态。实时帧和 Session 中的最终消息是不同对象，也不等于每个片段已独立写盘。

![6：流式片段与完整响应](images/deepseek-harness-detail-06-response.svg)

### 6.1 为什么 Driver 与 StreamAttempt 分开？

二者是同一执行过程中的不同职责，不是两个线程、进程或模型。

| 对象 | 负责的状态 / 动作 | 生命周期 |
|---|---|---|
| Driver，即 `ReactLoopAgent` | inbox、turn / step、模型请求、工具调度、继续 / 停止 / 取消 | 随一个活跃 Agent 存在，可处理多个 turn |
| `AssistantStreamAttempt` | 一次模型尝试的 chunk 累积、内容块组装、usage / finish / replayState，以及 start / chunk / end 帧 | 每次模型调用尝试新建；同一 step 重试也会新建 |

它们分别回答：“整个任务下一步做什么？”与“这一次模型尝试已经返回什么、怎样定型？”例如，一次请求失败后在同一 step 重试，driver 仍然拥有相同的任务与控制状态，但旧尝试的半段文本、调用参数和终止帧不能混入新尝试。每个 attempt 的身份与索引帮助观察者区分这些输出。

这也不代表把循环“交给 StreamAttempt”：driver 创建它、把 chunk 交给 `push()`，再读取 `blocks()` / `finish` 判断后续动作。正常提交时，driver 向 `settle()` 提供追加 Session 消息的回调；StreamAttempt 在追加成功后发 committed end，追加失败则发 abandoned end。它不自己决定是否调工具、重试或开启下一 step，也不自己选择模型或保证数据落盘。

工程取舍是把频繁变化的流式协议状态与任务控制状态分开，便于独立检查中断、重试和结果提交；并非理论上必须拆成两个类。图中两个生命线表示职责与生命周期边界，不表示网络边界。

### 6.2 一次响应怎样从片段变成消息？

`step()` 通过已准备的调用对象发起流式请求；代码也保留 `llm.stream(...)` 的分派路径。

`AssistantStreamAttempt` 收集流式内容，并发出 `agent/assistant-stream` 帧。成功完成后，loop 把组装好的响应追加成 `assistant/message`。

假设响应中有一个 `read` 调用，参数是：

```json
{
  "file_path": "README.md",
  "offset": 1,
  "limit": 40
}
```

这是该工具参数形状的教学示例，不是一次真实模型输出，也不是完整 provider 响应格式。

此时可以认为：

> 模型已经提出“读取这个文件的前 40 行”，但还没有证据证明文件存在、调用获准或读取成功。

loop 从响应内容中取出 `tool-call` 块，交给 `executeToolCalls()`。

## 7. 从调用建议到真实文件内容

**设计问题：为什么不直接执行模型给出的 `read(args)`？**

模型提供的是不可信输入，工具执行会触及真实资源。中间需要一个稳定的执行边界，统一处理策略、审批、不可覆盖的拒绝检查、取消及结果定型；工具体再负责文件领域的参数与读取语义。

设计重点是**授权与执行由程序掌握，模型不能靠写一段“已获准”越过边界**。工具只依赖文件服务接口，也使本地与其他后端能够替换。代价是错误来源不止一层，需要区分“策略拒绝”“工具参数错误”和“后端读取失败”。

![7：调度、策略与真实读取](images/deepseek-harness-detail-07-tool.svg)

图中 `prepare / dispatch / finalize` 是工具服务向调度器暴露的内部阶段，省略了访问该内部接口的符号写法；不是额外的模型工具。图展开 allow 分支；若策略返回 ask，会先等待审批，deny / cancel 分支不会继续进入 read 工具体。

### 7.1 调度与策略：不直接跳到工具函数

`agent-loop/src/tool-calls.ts` 负责调度，核心工具服务负责执行管线。简化顺序为：

```text
记录 tool/call
    → 准备执行身份和参数
    → tools/pre-execute 策略
    → 必要时请求审批
    → guards 检查
    → tools/execute 包装与实际工具函数
    → 后处理 / 结果定型
    → 记录 tool/result
```

上面是正常执行路径；拒绝或取消可能在中途形成错误结果，不再执行工具体。也不是每次只读调用都要求人工确认。

`tools/pre-execute` 是执行前策略扩展点；guard 可继续拒绝，不能用一个“允许”覆盖其他 guard 的拒绝。输入结构和取值校验也属于实际执行链，不能只信模型遵守 schema。

### 7.2 `read` 工具到底做什么？

文件工具插件在 `packages/fs/tool-fs/src/index.ts` 注册所选工具；具体读取在 `read.ts` 的 `applyReadTool()`。

其执行函数会：

1. 校验路径非空，`offset` / `limit` 为正整数，且不超过配置的行数上限。
2. 解析并检查读取目标。
3. 根据文件大小选择 `ctx.fs.readText(...)` 或 `ctx.fs.streamText(...)`。
4. 构造有行数、行长、字节限制的窗口。
5. 返回结构化内容，并渲染为模型可读结果。

其中 `ctx.fs` 是实际文件服务接口，后端决定在哪个文件系统中读取。**工具层不把本地磁盘实现写死。**

本篇追到文件服务调用边界，没有继续审查所有本地、远程文件后端的系统调用。读取本身还可能失败；前面讨论过的写入沙箱也不自动提供工作区读取隔离。

### 7.3 工具结果如何与调用对应？

调度器把模型的调用 ID 写入 `tool/call`。结果消息继续携带同一个 `callId`，并通过会话事件序号关联对应调用记录。

两种标识各有用途：

- `callId`：关联模型协议中的调用和结果。
- 会话事件序号：关联本地记录中的事实与来源。

它们都不是付款、发邮件等业务操作的幂等键。

如果模型一次提出多个调用，可并行安全的调用使用有界并发，exclusive 调用形成顺序屏障。派发可以重叠，但策略和结果提交仍维护模型调用顺序；本篇只有一个 `read`，暂不展开调度算法。

## 8. 为什么读取后模型能继续回答？

**设计问题：工具拿到了文件，为什么不由工具顺便生成最后回答？**

工具知道怎样读取文件，但不应该同时拥有用户目标、模型推理和循环控制。它把可关联的事实交回 Session；loop 再请求模型，让模型在“原问题 + 调用建议 + 实际结果”的上下文中解释。

设计重点是**工具返回证据，loop 决定继续，模型负责基于证据生成**。这样读取工具可以被不同任务复用；代价是多一次模型请求，以及相应的时延、token 和失败机会。确定性工作流不一定需要这次生成，使用 Agent 时也应评估它是否值得。

![8：工具结果进入下一次请求](images/deepseek-harness-detail-08-next-step.svg)

普通工具调用执行后，`step()` 返回“还需要继续”的结果，`turn()` 进入下一 step。

本例第二次请求的可见历史，在语义上包含：

```text
用户：读取 README，解释超时单位
助手：提出 read 调用
工具：对应调用的真实读取结果
```

下一 step 再次通过会话推导当前消息，并结合当时的 prompt、工具声明及模型参数构造请求。**不是简单地把工具文本拼到上一条 prompt 末尾，也不是把全部原始事件直接交给模型。**

如果文件结果明确写了单位，模型可以据此回答；如果只显示模糊配置名、窗口没覆盖定义或读取失败，就应继续找证据或说明不足。一次 `read` 返回成功不保证问题已经解决。

本例假设第二次模型响应给出有依据的解释，不含新的工具调用。

## 9. 怎样结束？结果又怎样返回调用方？

**设计问题：模型说完了，为什么不能立即把文本当成功结果返回并退出？**

可能还有刚进入的下一步输入、停止监听器引入的工作或待收束的子任务；记录也需要完成同步边界。与此同时，loop 不应自己决定是往 stdout 写文本，还是向网页推送事件。

设计重点是**计算的收束与结果的交付分开**：引擎提供状态和事实，入口按自己的协议交付。代价是 idle、turn completed、进程退出和业务成功成为不同状态，不能用一个 success 布尔值混写。

![9：收束、同步与入口交付](images/deepseek-harness-detail-09-delivery.svg)

图只画无额外输入、无子 Agent 的普通 headless 结果；实际入口还检查子 Agent 等待条件。`whenIdle()` 的返回箭头表示等待它的调用方得以继续，不是 loop 直接回调入口。

### 9.1 模型不再调用工具，是普通收束条件之一

在正常响应路径中，`step()` 没有找到工具调用，就返回 `{ kind: 'completed' }`。但 `turn()` 还要检查待处理的下一步输入，并经过停止阶段的监听器；若还有新工作，仍可能继续。

工具结果也可通过 `concludesTurn` 请求收束，输出 token 上限、阻止、错误和取消则有其他结束原因。**“模型没调工具”不是系统唯一的停止机制。**

本例没有额外工作，于是追加 `step/end`、`turn/end`。没有其他待处理输入时，driver 回到 idle；`whenIdle()` 等待这一收束边界。

### 9.2 headless 入口负责交付，不是 loop 直接操作 UI

该入口实际还会检查子 Agent 等待条件，本例没有子 Agent。随后它：

1. 调用 `sessions.flush(...)`。
2. 检查运行错误。
3. 从本次会话区间汇总最后的非空 assistant 文本和 turn 结束原因。
4. 普通模式把文本写到 stdout；错误信息按分支写到 stderr。
5. 根据结束原因选择退出码，`completed` 为 0，其他为 1。

`--json` 有单独的事件投影输出方式，不应把普通模式理解成所有流式文本都实时写到 stdout。

`flush` 是向已配置存储机制请求同步的边界，不自动证明任意部署都获得断电级持久性。退出码 0 也只说明程序采用了 completed 结果，不证明“单位解释正确”。

对 Web / SDK，结果交付会经过它们自己的桥接与协议层。本篇没有追这些层，因此不编造具体 UI 函数与网络调用。

## 10. 把控制流、事件与真实完成分开

| 看到的事实 | 可以说明 | 不能说明 |
|---|---|---|
| 输入已进入 inbox | 已排队 | 模型已读到它 |
| `assistant/message` 含工具调用 | 模型提出调用 | 工具执行成功 |
| `tool/call` | 有一次调用记录 | 外部动作已完成 |
| `tool/result` | 有结果，包括可能的错误 | 业务目标必然成功 |
| `turn/end`，原因 completed | 本轮按正常条件收束 | 回答有依据、任务验收通过 |
| Agent idle | 当前驱动空闲 | 进程已退出或外部操作已回滚 |

取消同样不是回滚。源码通过取消信号协作停止，已交付的流式文本会尽量以 interrupted 消息保存；工具已产生的外部副作用不会因为 `cancel()` 自动撤销。

事件顺序可作为阅读辅助，但不要套用固定模板判断所有失败分支。例如取消可能发生在输入提交之前，某个 turn 也可能没有实际 step。

## 11. 去哪里看源码？按主链读，不按目录全量翻

以下都指固定提交：

| 顺序 | 文件 / 符号 | 本轮只看什么 |
|---|---|---|
| 0 | [消息类型与构造](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/packages/llm/llm/src/message.ts)、[内容块类型](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/packages/llm/llm/src/types.ts) | role / source / content、tool-call 与 tool 结果的关联 |
| 1 | [headless 入口](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/packages/bundle/headless/src/index.ts)：`run()`、`summarize()` | create → followup → whenIdle → flush → 输出 |
| 2 | [Agent 公共服务](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/packages/core/agent/src/index.ts) | 公共接口与默认工厂分离 |
| 3 | [默认工厂](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/packages/core/agent-loop/src/index.ts)：`createAgent()`、`setupAndPublish()` | Session、作用域、ReactLoopAgent 与发布 |
| 4 | [默认驱动](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/packages/core/agent-loop/src/agent.ts)：`followup()`、`turn()`、`step()`、`buildRequest()` | 控制权、请求构造、继续与停止 |
| 5 | [调用调度器](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/packages/core/agent-loop/src/tool-calls.ts) | 工具派发与 call / result 关联 |
| 6 | [工具执行管线](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/packages/core/tools/src/index.ts)：`prepareExecution()`、`dispatchToolBody()` | 策略、审批、guards 与实际函数分离 |
| 7 | [read 工具](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/packages/fs/tool-fs/src/read.ts)：`applyReadTool()` | 参数校验、文件服务调用、有界返回 |

不是要求立刻通读这些文件。先按上面的任务故事定位关键函数，遇到恢复、请求重试、复杂并发等分支先标记；主链理解后再单独展开。

## 12. 本篇小结与下一步

现在可以将项目里的几个概念接起来：

> profile / bundle 装配出插件树；入口经公共 Agent 服务创建默认执行对象，消息通过 inbox 在边界接纳。loop 从 Session 推导模型请求，模型只提出调用，工具管线和后端完成实际操作；结果记入 Session，供下一 step 使用。入口再把运行结果交付给调用方。

- 已提供：内部 messages 格式、两次请求的数据变化，以及新建 Agent、只读文件、双 step、普通完成的静态主链；第 3～9 节各有设计问题、取舍说明与局部时序图。
- 尚未验证：实际模型行为、文件后端及持久化效果、UI 传输、故障恢复和平台兼容性。
- 下一篇：[读取失败、模型请求重试与用户取消](2026-10-16_02_deepseek_harness_failures_retries_and_cancellation.md)，沿同一主链理解错误如何回填、哪里重试、哪里必须停止；之后再深入 Session 视图及上下文、规划插件。
