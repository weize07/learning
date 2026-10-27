# Agent 主干 11：MCP 与连接器

[主学习索引](../index-2026-10-08.md) · 上一课：[鲁棒性与成本预算](2026-10-19_02_agent_reliability_and_budgets.md)

下一课：[多 Agent 与委派](2026-10-19_04_multi_agent_and_delegation.md)。

**核心：MCP 标准化应用如何发现和调用外部能力，不替代 Agent 循环、业务授权或可靠执行。**

延续“读取项目资料、生成周报、批准发送”的场景。本课以 **MCP 2025-11-25 规范**为固定学习基线，2026-10-19 核对官方文档；不宣称它是当前最新版本。消息示例为自拟教学数据，不是实际调用记录。没有安装 SDK、启动 MCP server 或连接业务系统。

## 1. 是什么：应用与能力提供方之间的协议

### 1.1 Host、Client、Server 分别是什么

| 对象 | 职责 | 在周报例子中的位置 |
|---|---|---|
| Host | 承载 AI 应用，管理模型、连接、上下文与用户授权决策 | 周报 Agent 应用 |
| MCP Client | 在 Host 一侧维护到某个 server 的协议连接，处理协商与消息 | 文档连接、邮件连接各自的客户端 |
| MCP Server | 对外提供工具、资源或提示模板，连接实际系统 | 文档服务适配器、邮件服务适配器 |
| 业务后端 | 真正保存数据和产生业务副作用 | 文档库、邮件发送服务 |

这里的 Host 是 MCP 架构角色，不要求与你之前学习的某个框架里的 Host 类相同。

```text
Agent application / Host
  -> MCP Client A -> Document MCP Server -> Document store
  -> MCP Client B -> Mail MCP Server     -> Mail backend
```

一个 Host 可管理多个 Client；官方架构按每个 Client 与特定 Server 建立一对一关系描述。**MCP 连接会话不等于用户聊天会话或 Agent 的持久 Session**，不能仅靠一个连接 ID 推断业务授权和历史归属。

“连接器”不是必须一一对应的协议对象：它通常泛指接入某个系统的整合组件，可以包含鉴权、数据转换、MCP Client 或 Server。也可以不用 MCP，直接调用 REST API。

### 1.2 tools、resources、prompts 的区别

| 原语 | 表达什么 | 典型方法 | 例子 |
|---|---|---|---|
| tools | 可调用的操作 | tools/list、tools/call | search_docs、read_doc、send_report |
| resources | 可读取的上下文内容，以 URI 标识 | resources/list、resources/read | 项目说明、只读状态快照 |
| prompts | 可获取并填入参数的提示模板 | prompts/list、prompts/get | 周报撰写模板 |

三者是不同的接口组织方式，不是“工具有权限、资源不需要权限”。Server 不必全部实现；列出的资源也不会自动全部进入模型上下文，Host 仍需选择与读取。

### 1.3 MCP 与 Function Calling 不在同一层

| 机制 | 回答的问题 |
|---|---|
| 模型 Function Calling | 怎样告诉模型有哪些工具，模型怎样提出名称和参数 |
| MCP | 应用怎样发现 server 的能力，并通过协议请求执行、取得结果 |
| Agent runtime | 谁决定继续调用、什么时候停止、怎样保存状态与控制预算 |
| 业务后端 | 真实用户是否有权限，操作怎样产生并记录效果 |

常见组合：MCP 发现工具 → Host 筛选并转换成模型工具定义 → 模型提出调用 → Host 授权后用 MCP 调用 → 结果回填模型。

**模型通常不直接把 MCP JSON-RPC 消息发给 server；Host 也可以不经过模型，直接发起 MCP 调用。**

## 2. 为什么要做：减少重复接线，而不是消除业务差异

- **重复接入**：每个 AI 应用都为文档、代码库、业务 API 编写发现与调用封装，维护成本高。
- **能力发现**：不能只知道一个地址，还要知道工具名、输入结构和支持哪些功能。
- **连接生命周期**：需要协商版本、管理连接与关闭，而不是随意发一段 JSON。
- **组合扩展**：应用希望统一接入多个 server，不把每个后端的通信方式写进 Agent 主循环。

统一协议可以复用接入机制，但工具的语义、权限、数据模型和幂等保证仍不相同。**两个 server 都有 search 工具，不代表它们可无损互换。**

## 3. 怎么做：从握手到一次工具往返

### 3.1 先完成连接与能力协商

正常初始化顺序：

```text
Client -> Server: initialize
Server -> Client: result (protocolVersion, capabilities, serverInfo)
Client -> Server: notifications/initialized
Client -> Server: tools/list
```

- 协议版本用于判断双方能否兼容；不支持返回版本时应断开，而不是继续猜字段。
- capabilities 表示支持哪些协议功能，不是用户有权访问哪些文档的许可表。
- 工具支持列表变化通知时，Client 可据此重新发现；Host 仍需处理缓存、审核和作用域更新。
- 初始化不代表 server 已获准读取全部聊天记录，也不代表所有工具已对模型开放。

### 3.2 tools/list 返回什么

下面是 server 返回的一项工具定义；省略 JSON-RPC 外层和其他工具：

```json
{
  "name": "read_doc",
  "description": "Read one document available to the authenticated user.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "documentId": { "type": "string" }
    },
    "required": ["documentId"],
    "additionalProperties": false
  }
}
```

Host 不应无条件把发现的全部工具暴露给每个 Agent：

- 按用户、任务和风险选择工具；必要时要求审批。
- 多个 server 有同名工具时，维护本地别名到 server + 原工具名的明确映射。
- 限制描述与结果大小，不把远端描述、注解或正文当作可信系统指令。
- 工具宣称“只读”不能代替权限检查；特别是不可信 server 提供的注解。

### 3.3 模型调用与 MCP 调用如何衔接

假设模型提出读取 doc-17。Host 完成校验与授权后，Client 发出：

```json
{
  "jsonrpc": "2.0",
  "id": 17,
  "method": "tools/call",
  "params": {
    "name": "read_doc",
    "arguments": { "documentId": "doc-17" }
  }
}
```

Server 调用文档后端，返回：

```json
{
  "jsonrpc": "2.0",
  "id": 17,
  "result": {
    "content": [
      { "type": "text", "text": "本周完成接口迁移，待处理两项回归问题。" }
    ],
    "isError": false
  }
}
```

Host 再把结果转换为模型 API 对应的工具结果消息：

```text
Model tool call (callId = c1)
  -> Host validates and authorizes
  -> MCP tools/call (JSON-RPC id = 17)
  -> MCP result (JSON-RPC id = 17)
  -> Model tool result (callId = c1)
```

**JSON-RPC id 关联协议请求与响应；模型 callId 关联模型工具往返；operationId 关联业务幂等操作。** 三者职责不同，不可拿请求 id 自动当作幂等保证。

MCP 结果还可包含其他内容形式；支持结构化输出的工具可声明 outputSchema、返回 structuredContent。本课先使用纯文本，不假设所有 server 或模型 adapter 都支持所有内容类型。

### 3.4 stdio 与 Streamable HTTP

| 传输 | 怎么连接 | 需要注意什么 |
|---|---|---|
| stdio | Client 通常启动本地子进程，通过 stdin / stdout 交换协议消息 | 启动程序本身就是运行代码；stdout 只能输出有效 MCP 消息，诊断日志写 stderr |
| Streamable HTTP | Client 访问独立 server 的 MCP HTTP 端点 | 使用 HTTP POST / GET，可用 SSE 传递流；处理鉴权、Origin 校验、连接与会话管理 |

在固定规范中，Streamable HTTP 替代旧 HTTP+SSE 传输。**它仍可使用 SSE**，不能误解成“SSE 已被禁止”。读取老教程时先对齐规范与 SDK 版本。

stdio 本地进程不会天然被沙箱化，也不天然可信；HTTP 连接中断则不代表业务执行已取消。

### 3.5 两层授权：能连接不等于能执行

- **是什么**：访问受保护的 MCP server 与操作某份具体业务数据，是相关但不同的授权范围。
- **为什么要做**：登录成功的用户也不能读取其他租户的文件；允许读资料不代表允许发送邮件。
- **怎么做**：
  - 固定规范为 HTTP 传输定义了基于 OAuth 的授权机制；授权在规范中是可选能力，受保护部署应明确落实所需鉴权。
  - stdio 不套用该 HTTP OAuth 流程；凭据通常由启动环境等安全渠道提供，仍需最小化子进程获得的权限与秘密。
  - 服务端使用可信身份实施资源级授权；不相信模型参数中自报的用户身份。
  - 发送仍需上一课定义的内容批准；不能用“已连接 MCP”代替批准。
- **为什么这么做**：协议接入负责建立受控通路，执行后端负责约束真正的数据与副作用。秘密不能为了方便直接塞进提示词。

### 3.6 失败、重连和取消怎样处理

| 现象 | 含义与处理 |
---|---|
| JSON-RPC error | 请求 / 协议层处理失败，例如不支持的方法；保留具体错误，不变成成功空结果 |
| tools/call 返回 isError=true | 工具层执行失败；Host 要保留失败语义，再决定是否向模型提供可修正信息 |
| 连接断开或超时 | 结果可能未知；读取按策略有界重试，发送等副作用先查状态或依赖可靠幂等 |
| 取消通知已发出 | 只是请求停止，不证明后端已退出或撤销副作用 |
| 重新连接成功 | 恢复了通信，不代表原请求未执行，也不自动恢复用户任务 |
| 工具列表变化 | 重新获取并按策略更新映射；避免模型拿旧 schema 调用新版工具 |

沿用上一课：同一任务的 deadline、调用次数、费用与并发约束必须覆盖 MCP 调用；协议连接恢复不能重置预算。

## 4. 为什么这么做：统一通信，保留执行责任

| 方案 | 收益 | 代价与边界 |
|---|---|---|
| MCP Server 封装业务后端 | 多个 Host 可以复用同一协议接口 | Server 仍要维护业务映射、权限与依赖版本 |
| Host 控制工具暴露与结果回填 | 不把协议连接直接交给模型任意使用 | 需要工具映射、审批、schema 与内容转换 |
| 先协商再操作 | 可检测版本和功能不兼容 | 双方都要维护生命周期与能力支持 |
| 区分协议、模型和业务 ID | 更容易定位错误及防止重复副作用 | 需要显式关联和追踪，不是一根 requestId 包办所有 |
| 仍在后端强制授权与幂等 | 换 Host、重连或模型误判时仍有边界 | MCP 本身不能自动实现这些业务保证 |

只有一个稳定私有 API 时，直接接入可能更简单；当多个应用和能力提供方需要互通时，标准协议更有价值。不要为了使用 MCP 而把简单函数强行拆成远端服务。

## 5. MVP 与验收：先接一个受控只读工具

**建议范围**：本地测试资料、一个受控 read_doc 工具、固定 SDK / 协议版本，先不发送邮件、不连接真实企业数据。用现有受维护 SDK 实现协议，不手写整套传输。

| 场景 | 可观察的验收条件 |
---|---|
| 初始化与工具发现 | 实际完成 initialize / initialized，再发现预期工具；不兼容版本显式失败 |
| 模型提出读取 | 实际走一次 MCP tools/call；校验文档 ID、返回内容与模型 callId 回填 |
| 不同 server 同名工具 | 本地映射可区分，不把调用送错 server |
| 参数错误或越权文档 | 明确报错；敏感内容未进入模型上下文，不以空文本假装成功 |
| 工具 isError=true | 模型与任务状态保留错误事实，不显示“读取成功” |
| 只读调用超时并取消 | 有界停止等待并记录后端退出情况，不残留无管理请求 |
| 模拟发送已成功但连接断开 | 保留未知结果或按 operationId 确认，不能因重连无条件重发 |
| 工具描述或结果含改变权限的文本 | 不改变授权策略，不触发未批准操作 |
| 连接关闭 | 释放自己拥有的连接 / 子进程，不误杀其他客户端的共享 server |

以上为待实现的验收方案，不是已运行结果。下一课进入**通用多 Agent 与委派**：区分 router、supervisor、handoff 和并行分工，再用同预算对照判断是否值得增加协作。

### 官方规范与按需下钻

本课固定基线均为 2025-11-25，核对日期 2026-10-19：

- [架构](https://modelcontextprotocol.io/specification/2025-11-25/architecture)：Host / Client / Server、上下文隔离与能力协商。
- [生命周期](https://modelcontextprotocol.io/specification/2025-11-25/basic/lifecycle)：初始化、版本与能力。
- [Tools](https://modelcontextprotocol.io/specification/2025-11-25/server/tools)：发现、调用、schema 与结果形式。
- [传输](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports)：stdio、Streamable HTTP。
- [授权](https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization)：HTTP 授权与 stdio 的区别。

先读正文，遇到字段或互操作问题再回规范；不把规范支持的功能当成每个 server 都已实现。
