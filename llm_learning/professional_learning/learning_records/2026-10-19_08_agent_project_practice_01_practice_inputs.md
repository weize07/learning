### 题 1：把需求变成接口

给出一张简图，说明运行循环、模型适配器和工具执行之间怎样连接，再写出关键数据的结构。

至少覆盖：

- 一次模型请求包含什么；工具的名称、说明和参数约束由谁提供。
- 如何明确区分“请求工具”和“最终回答”，而不是靠搜索自然语言里的关键词。
- 一个工具调用怎样与其结果对应。
- 谁持有历史消息和模型调用计数；谁检查输入并真正执行工具。
- 运行结果如何区分成功、执行错误和预算耗尽。

**交付形式：** 一张简图 + 一张接口 / 数据结构表即可。字段、类型和责任必须说清楚，不必拘泥于某家模型 API 的格式。

#### 题 1: 接口设计

##### 补充版：本轮可以交给实现者的最小设计

2026-10-20 根据评审补充。下方是本轮拟采用的接口契约，不是已经运行的 Python 代码，也不要求照搬 DeepSeek Harness 的完整架构。原始方案保留在折叠区；题 2、3 尚未修改。

###### 1. 最小架构与责任归属

```text
用户任务
   |
   v
Runner（运行循环，持有 RunState）
   |
   +-- 构造 ModelRequest --> ModelAdapter --> ModelResponse
   |                                            |
   |                         +------------------+-----------------+
   |                         |                                    |
   |                    tool_call                               final
   |                         |                                    |
   |                检查是否允许继续                            保存回答
   |                保存 assistant 调用记录                     返回成功
   |                         |
   +-- ToolExecutor.execute(call)
   |      校验名称和参数 --> 执行已登记工具 --> ToolResult
   |
   +-- 保存 tool 结果记录 --> 构造下一次模型请求

模型 / 工具失败或预算耗尽 --> 返回明确的失败结果，不伪造回答
```

| 组件 | 输入 / 输出 | 负责什么 |
|---|---|---|
| `Runner` | `run(task: str) -> RunResult` | 初始化并持有本次状态；组装请求、计数、分支处理、维护历史和返回运行结果 |
| `ModelAdapter` | `generate(request: ModelRequest) -> ModelResponse` | 本轮用脚本化模拟模型；检查实际收到的请求并返回结构化响应，不执行工具、不修改运行状态 |
| `ToolExecutor` | `definitions() -> list[ToolDefinition]`；`execute(call: ToolCall) -> ToolResult` | 提供工具定义；校验工具名、参数和资料 ID；只分派已登记工具，执行失败时抛出明确错误 |
| 内存资料库 | 按 ID 返回固定测试资料 | 仅供工具实现访问，不直接预填进模型请求 |

`ToolExecutor` 内部保存一份工具登记信息，包含定义与对应执行函数；对模型只暴露定义，不暴露函数对象。这样模型看到的工具名称和执行器可分派的名称来自同一处，不维护两套容易失配的列表。

历史和预算由 `Runner` 管理，不交给模型或工具执行器修改。本轮不需要独立的 AgentRegistry、Inbox、SessionManager、Connector 或流式读写对象。

###### 2. 关键数据结构

下面使用类型描述，不是特定模型厂商的 API 格式。`JSONValue` 表示合法 JSON 值；`JSONObject` 表示字符串键到 JSON 值的映射，不包含任意 Python 对象。

| 数据 | 字段与类型 | 约束 / 用途 |
|---|---|---|
| `ToolDefinition` | `name: str`；`description: str`；`parameters: JSONObject` | `parameters` 使用 JSON Schema 描述参数；同一执行器内名称唯一 |
| `ToolCall` | `call_id: str`；`name: str`；`arguments: JSONObject` | 调用 ID 非空、在一次运行内唯一；名称和参数需要执行器继续校验 |
| `ToolResult` | `call_id: str`；`content: JSONValue` | 成功结果的 ID 必须等于原调用 ID；失败不伪装成正常结果 |
| `TextMessage` | `role: "system" / "user"`；`content: str` | 初始指令与用户任务 |
| `AssistantToolCallMessage` | `role: "assistant"`；`call: ToolCall` | 保存模型请求执行的动作 |
| `ToolResultMessage` | `role: "tool"`；`result: ToolResult` | 保存程序实际执行后的结果 |
| `AssistantFinalMessage` | `role: "assistant"`；`content: str` | 保存最终回答，不作为工具执行 |
| `Message` | 上述四种消息结构之一 | 通过角色和字段结构区分；不靠在正文里搜索关键词 |
| `ModelRequest` | `messages: list[Message]`；`tools: list[ToolDefinition]` | 包含截至本次调用前的完整历史及可用工具定义；传递独立快照，不让后续历史追加改变已记录请求 |
| `ToolCallResponse` | `kind: "tool_call"`；`call: ToolCall` | 只包含一个调用，不包含最终回答 |
| `FinalResponse` | `kind: "final"`；`content: str` | 最终回答；没有工具名称，不分派给执行器 |
| `ModelResponse` | `ToolCallResponse` 或 `FinalResponse` | 用 `kind` 明确区分互斥分支；未知类型或不合法结构是模型响应错误 |
| `RunState` | `messages: list[Message]`；`model_calls: int`；`max_model_calls: int` | 每次运行新建；调用数从 0 开始，本轮上限为 5 |
| `RunError` | `code: str`；`message: str`；`call_id: str / None` | 保留可定位信息；调用相关错误附调用 ID |

运行结果采用互斥结构，避免“状态失败却带着成功答案”的歧义：

| 运行结果 | 字段与类型 | 含义 |
|---|---|---|
| `SuccessResult` | `status: "success"`；`answer: str`；`state: RunState` | 已接纳最终回答，不再调用模型或工具 |
| `ErrorResult` | `status: "error"`；`error: RunError`；`state: RunState` | 输入 / 响应不合法，或模型 / 工具执行失败；无成功答案 |
| `BudgetExhaustedResult` | `status: "budget_exhausted"`；`error: RunError`；`state: RunState` | 因模型调用预算停止；不是模型已经完成任务 |

`RunResult` 是这三种结果之一；返回的 `state` 是结束时快照。错误码至少区分 `invalid_model_response`、`model_error`、`unknown_tool`、`invalid_arguments`、`document_not_found`、`tool_execution_error` 和 `model_call_budget_exhausted`。错误信息不能只打印后丢弃，也不能替换为空资料继续运行。

###### 3. 两个工具的具体契约

以下是模型能看到的定义；执行器也必须落实相同校验，不能认为模型会自动遵守 schema。

```json
[
  {
    "name": "list_documents",
    "description": "列出可用资料的 ID 和标题，不返回正文。",
    "parameters": {
      "type": "object",
      "properties": {},
      "additionalProperties": false
    }
  },
  {
    "name": "read_document",
    "description": "按资料 ID 读取一份资料，不接受文件路径。",
    "parameters": {
      "type": "object",
      "properties": {
        "document_id": {"type": "string", "minLength": 1}
      },
      "required": ["document_id"],
      "additionalProperties": false
    }
  }
]
```

- `list_documents` 的调用参数只能是 `{}`；结果 `content` 为 `{"documents": [{"document_id": "...", "title": "..."}]}`。
- `read_document` 的参数示例为 `{"document_id": "dev-week"}`；结果 `content` 包含 `document_id: str`、`title: str` 和 `body: str`。
- ID 是否存在由资料库查询确认；schema 合法不等于资料一定存在。未知名称或参数不合法时，不执行工具主体。

###### 4. 一次调用与结果怎样关联

例如，模型适配器返回：

```json
{
  "kind": "tool_call",
  "call": {
    "call_id": "c2",
    "name": "read_document",
    "arguments": {"document_id": "dev-week"}
  }
}
```

在预算允许、实际执行成功后，历史中按顺序追加：

```json
[
  {
    "role": "assistant",
    "call": {
      "call_id": "c2",
      "name": "read_document",
      "arguments": {"document_id": "dev-week"}
    }
  },
  {
    "role": "tool",
    "result": {
      "call_id": "c2",
      "content": {
        "document_id": "dev-week",
        "title": "研发周记",
        "body": "本周关闭 8 个缺陷。数据库迁移仍在进行，尚未验收。"
      }
    }
  }
]
```

这两条记录进入下一次 `ModelRequest.messages`，不会追溯进入刚才已经结束的模型请求。不能仅靠两条消息相邻来关联，也不能把工具结果写成用户输入或模型自己生成的事实。

如果返回 `{"kind": "final", "content": "..."}`，`Runner` 保存最终回答并返回 `SuccessResult`，不调用名为 `respond` 的工具。

###### 5. 预算与异常由谁处理

- `Runner` 在调用模型前检查额度，在发起调用时计数；不是等模型返回后才决定这次调用是否算数。
- 收到最终回答与收到工具调用走不同分支；第 5 次的最终回答可成功，第 5 次的工具请求不得执行。具体检查顺序仍由题 3 展开。
- `ToolExecutor` 校验参数并执行工具；`Runner` 接收执行错误，保留错误码、信息和可用的调用 ID，结束本次运行。
- 模型调用失败也应明确返回错误，本轮不自动重试。未预期的异常不能静默吞掉或转换成成功结果；应保留诊断并显式暴露失败。
- 本轮使用普通请求 / 响应接口即可。真实 API 可以一次提交完整请求、选择流式接收响应，不需要默认设计一个“流式写入模型消息”的对象。

<details>
<summary>原始方案：多组件与完整会话生命周期（保留对照，本轮暂不实现）</summary>

主要的功能对象包括：
- AgentRegistry: 负责管理所有Agent的注册和查找，提供根据配置或标识获取Agent实例的能力
- Agent: 对应整个智能体，作为容器，绑定Driver，Inbox, Session等对象的生命周期，Agent通常在请求到来时被创建，并且在请求处理完成后可以被销毁，也可以在Session恢复时重新创建。多个Session对应多个Agent。
- Bundle: 负责定义Agent的配置和能力，包括可用工具、模型等
- Driver: 负责统一调度Inbox读取，Turn/Step流转，模型调用，工具执行，session写入等
- Connector: 负责与UI交互，传递用户输入和显示模型输出(API是不是一般也是通过Stream来实现？)
- SessionManager: 负责管理会话的生命周期，包括创建、更新和销毁
- SessionEvent: 代表会话中的事件，例如创建、更新、销毁等，不仅仅包括用户输入，还包括系统生成的事件，如Inbox消息出队，工具调用结果、模型响应等。
- Inbox: 负责存储和管理用户输入的消息
- Turn: 代表一次对话轮次
- Step: 代表一次模型调用的具体步骤
- ModelProvider: 负责提供模型调用接口
- ToolService: 负责提供工具的调用接口，协调ToolExecutor执行工具，并处理工具调用的结果。
- ToolExecutor: 负责执行工具调用
- Guard: 负责检查输入是否合法，是否超出预算等
- ToolRegistry: 负责管理可用工具的信息
- LLMStreamReader: 负责读取模型输出的流
- LLMStreamWriter: 负责写入模型输入的流
- LLMRequest: 代表一次模型请求，包含请求的内容、请求的工具信息、请求的参数等

1. 服务启动时
- 初始化AgentRegistry, SessionManager, ToolRegistry等公共组件。根据配置所定义的Bundle, 初始化对应的ModelProvider和ToolRegistry中定义的工具。
2. 用户发起请求时
- AgentRegistry根据请求中的配置或标识获取对应的Agent实例，如果不存在则创建新的Agent，并初始化其Driver、Connector、SessionManager、Inbox等组件。
- Connector根据请求中的用户输入，将其写入Inbox，唤醒Agent调度处理。
- Agent接收到请求后，通过Driver调度Inbox读取用户输入，创建新的Turn和Step，调用ModelProvider生成模型请求，并根据需要调用ToolService执行工具。执行结果返回后，Driver将结果写入Session，并更新Inbox和Turn的状态。
    - ToolService在执行工具前，会调用Guard进行输入合法性检查和预算检查，确保工具调用符合系统策略。
    - ToolService在执行工具后，会处理工具的返回结果，并将结果传递给Driver，Driver根据结果更新Turn的状态。
- 如果在执行过程中出现输入不合法、工具执行错误或预算耗尽等情况，Guard会进行检查并返回相应的错误信息，Driver根据错误类型更新Turn的状态。
- 一次用户请求通常对应一个Turn，Turn中可能包含多个Step，每个Step对应一次模型调用或工具执行。最后一个Step的结果通常作为Turn的最终输出通过Connector返回给用户，Driver会根据Turn的状态更新Session和Inbox，确保整个对话的上下文和历史记录得到正确维护。
3. 用户在会话中继续交互时
- Connector接收到用户的新输入后，将其写入Inbox。
- Driver根据当前的Session状态，
    - 如果上一个Turn尚未完成，Driver会继续处理该Turn的某个Step。
        - 如果被写入的Inbox的新消息是steer输入，那么在当前Step结束之后，Driver会根据steer输入调整后续Step的执行策略，确保对话按照用户的引导进行。
        - 如果被写入的Inbox的新消息是普通用户输入，Driver会继续处理当前Turn直至完成。完成后开启新的Turn处理新的用户输入。
    - 如果上一个Turn已经完成，Driver会根据新的用户输入创建新的Turn和Step，继续处理对话。
4. UI主动结束会话时,
- Connector接收到结束会话的指令后，将其传递给Agent。
- Agent通过Driver处理结束会话的逻辑，更新当前Turn和Step的状态，并通知SessionManager销毁或标记会话为结束。

</details>

### 题 2：走一遍真实的数据流

列出下列四次模型调用之间发生的事情。
测试安排：

1. 第一次模型调用：请求 `list_documents`。
2. 第二次：请求读取 `dev-week`。
3. 第三次：请求读取 `ops-week`。
4. 第四次：返回包含正确事实与来源的摘要。

#### 题 2：答案

| 时点 | 模型实际能看到什么 | 模型返回什么 | 程序执行什么 | 留给下一步的记录 |
|---|---|---|---|---|
| 第一次调用 | system_prompt, user_message, tool_list | suggest_call(list_documents) | call(list_documents) | list_documents的调用结果 |
| 第二次调用 | system_prompt, user_message, tool_list, list_documents的调用结果 | suggest_call(read(dev-week)) | call(read(dev-week)) | read(dev-week)的调用结果 |
| 第三次调用 | system_prompt, user_message, tool_list, suggest_call(list_documents), list_documents的调用结果, suggest_call(read(dev-week)), read(dev-week)的调用结果 | suggest_call(read(ops-week)) | call(read(ops-week)) | read(ops-week)的调用结果 |
| 第四次调用 | system_prompt, user_message, tool_list, suggest_call(list_documents), list_documents的调用结果, suggest_call(read(dev-week)), read(dev-week)的调用结果, suggest_call(read(ops-week)), read(ops-week)的调用结果 | suggest_call(respond) | respond | 包含正确事实与来源的摘要 |

### 题 3：修正一个会越界的循环

有人提出以下执行顺序：

> 先调用模型；如果模型请求工具就执行；最后检查“模型调用次数是否大于 5”，超过才结束。

请完成：

1. 指出这个顺序违反了哪些本轮约束，包括第 5 次和第 6 次附近的行为。
2. 给出你的检查顺序，保留“第 5 次返回最终回答仍算成功”的行为。
3. 写出至少两条边界测试的具体断言：不能只写“程序应该停止”，要说明模型调用数、工具执行数以及最终状态。

可以用“模型持续请求 `list_documents`”作为故障输入。计数都从一次全新的运行开始，不继承上一次测试的状态。

#### 题 3：答案
如果模型一直没有收束到respond, 且持续请求调用工具，那么模型调用次数会不断增加，可能超过系统设定的上限，导致循环越界。正确的检查顺序应在每次模型调用后立即检查调用次数是否超过限制，若超过则终止会话并返回错误信息。