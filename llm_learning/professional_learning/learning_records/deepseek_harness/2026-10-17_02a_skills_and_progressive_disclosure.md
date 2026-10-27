# 扩展 2a：Harness 中的 Skills 与渐进式披露

[专题入口](README.md) · 前置：[消息、事件与请求](2026-10-17_02_messages_events_requests.md) · 通用概念：[Skills](../2026-10-15_06a_skills_and_progressive_disclosure.md) · 下一主站：[执行循环](2026-10-17_03_loop_and_lifecycle.md)

实际补写及重写：2026-10-19。旧日期前缀只用于课程排序；本章不增加七站 MVP 范围。源码固定为 `571902028a75c64c1d767f4ae5fd099485f0ddf0`；只核对源码，未启动 Harness、调用模型或运行测试。

## 1. 先看完整过程：发现 → 告知 → 加载 → 做事

**Skill 是一份可带资源的操作说明，不是自动运行的程序。Harness 先把“有哪些说明”告诉 LLM；LLM 请求某一份，读完后再通过普通工具做事。**

本章只用一个教学任务：**“把销售 CSV 汇总成报告。”** 假设项目中已放好自编的 `sales-report` 技能和配套脚本；它们不是 Harness 内置技能，本章也不会创建或运行它们。

| 顺序 | 谁做 | 具体发生什么 | 此时 LLM 知道什么 |
|---|---|---|---|
| ① 发现 | Harness | 扫描配置的技能目录，解析名称和描述，保存文件位置 | 还不知道，尚未发请求 |
| ② 告知 | Harness → LLM | 请求中加入技能目录，以及 `skill` 工具的调用定义 | 知道有 `sales-report`，知道怎样请求正文 |
| ③ 选择 | LLM → Harness | 返回工具调用：`skill`，参数为 `{"name":"sales-report"}` | 选中了技能，还没读到正文 |
| ④ 加载 | Harness → LLM | 读取正文，作为工具结果放入下一次请求 | 现在知道具体操作步骤和资源位置 |
| ⑤ 做事 | LLM → Harness | 根据说明，再请求读文件、执行命令等普通工具 | 根据执行结果继续判断 |

**① 扫目录不需要 LLM 指挥；③ 选择哪份技能通常由 LLM 决定；④ 加载正文不会自动执行⑤。** 先走这条普通工具调用路径，其他入口放在文末。

## 2. Harness 怎么发现技能？

### 2.1 先有文件，再有扫描

假设当前项目根目录为 `C:\demo`，技能文件布局如下：

```text
C:\demo
  .agents
    skills
      sales-report
        SKILL.md
        scripts
          build-report.mjs
  sales.csv
```

其中 `SKILL.md` 的示意内容：

```markdown
---
name: sales-report
description: 读取销售 CSV，核验字段并生成销售汇总报告。
---

先确认输入文件、输出位置及统计口径。
检查 CSV 是否具备脚本要求的字段。
需要生成报告时，通过命令执行工具运行 scripts/build-report.mjs，
用 --input 指定 CSV，用 --output 指定报告路径。
执行后核验输出内容，不只看退出码。
```

- 开头两条横线之间的 **frontmatter** 提供名称、描述等元数据，用于建立目录。
- 后面的**正文**告诉 LLM 怎样完成任务。
- `scripts` 中的文件是可用资源；放在那里不代表已经运行。

### 2.2 谁扫描，什么时候扫描？

先认识三个实现名称，下面只沿调用顺序使用它们：

| 名称 | 本流程中的具体工作 |
|---|---|
| `skill-filesystem` | 文件来源插件：知道到哪些目录找技能、怎样读取文件 |
| `ctx.skills` | 技能服务：向来源查询，建立“名称 → 技能来源及位置”的对应关系 |
| `tool-skill` | 模型接口插件：把目录交给 LLM，并接收正文加载请求 |

执行顺序：

1. **插件装配时**，`skill-filesystem` 向 `ctx.skills` 注册一个文件来源；`tool-skill` 注册名为 `skill` 的工具。这不是已经把所有正文发给模型。
2. **准备一次模型 step 时**，`tool-skill` 的 pre-step 处理器调用 `ctx.skills.snapshot(...)`，查询当前 Agent、当前工作目录下有哪些可用技能。pre-step 就是“发起本步模型调用前的处理阶段”。
3. **目录没有可复用缓存时**，技能服务调用来源的 `list(...)`。文件来源定位搜索根目录，枚举直接子项，寻找 `sales-report\SKILL.md` 这样的文件，解析元数据。
4. **服务合并查询结果**，解决同名候选，返回技能摘要。文件位置留在 Harness 一侧，供以后按名称加载。

本例的关键对应关系是：

```text
sales-report
  -> filesystem provider
  -> C:\demo\.agents\skills\sales-report\SKILL.md
```

默认搜索位置包括项目的 `.dsh\skills`、`.agents\skills`，配置的自定义目录，以及用户级技能目录。项目根取最近的 `.git` 祖先，找不到则用 cwd。也支持根目录下直接放 `<name>.md`；**不会递归把项目里的所有 Markdown 都识别成技能**。

到这一步，**Harness 已知道有哪些技能，LLM 还不知道**。此外，“目录只返回摘要”不等于扫描时只读取文件开头：实现可读完整文件来解析，但不会因此把全文提供给 LLM。

## 3. Harness 怎样告诉 LLM？同一请求中的两部分

这是最容易混淆的地方：**技能清单放在 messages；加载技能的接口定义放在 tools。两者缺一不可。**

| 请求部分 | 作用 | 本例 |
|---|---|---|
| `messages` 中的目录消息 | 告知“有哪些技能、各自适用什么任务” | `sales-report`：读取销售 CSV 并生成报告 |
| `tools` 中的 `skill` 定义 | 告知“用什么接口、传什么参数来取得正文” | 调用 `skill`，传字符串参数 `name` |

**不是每个 Skill 都注册成一个工具。** 即使有 100 个技能，普通加载路径仍可共用一个 `skill` 工具，用 `name` 选择其中一项。

下面采用 **Chat Completions 风格**展示首次请求的相关字段，便于看清位置；文案与任务为自拟，省略模型配置、其他消息和其他工具定义（如 `pwsh`），不是实际 HTTP 抓包。Harness 内部的消息块由 adapter 转换成各 provider 的格式。

```json
{
  "messages": [
    {
      "role": "user",
      "content": "把销售 CSV 汇总成报告。"
    },
    {
      "role": "user",
      "content": "<system-reminder>\n<available_skills>\n- `sales-report`: 读取销售 CSV，核验字段并生成销售汇总报告。\n</available_skills>\n任务匹配时，先调用 skill 工具加载对应正文。\n</system-reminder>"
    }
  ],
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "skill",
        "description": "加载指定技能的完整指令。",
        "parameters": {
          "type": "object",
          "properties": {
            "name": { "type": "string" }
          },
          "required": ["name"]
        }
      }
    }
  ]
}
```

这份请求里**没有**“确认口径、检查字段、运行脚本”的完整正文，也没有脚本源码。

对应源码中，目录由 `tool-skill` 的 `renderCatalogMessage(...)` 生成，是 **user-role 消息**，内部来源为 `skill-catalog`。标签 `<system-reminder>` 只是文本包装，不会改变 role。Agent loop 将 pre-step 接纳的消息写入 Session，再构造模型请求。

因此，LLM 知道技能存在，靠的不是“自己看见磁盘”，而是 **Harness 把名称和描述写进了本轮输入**。正常模型选择路径中，也是 LLM 根据任务与描述选择，不是 Harness 用文件后缀自动选好正文。

## 4. LLM 怎样要求加载？加载后怎样执行？

### 4.1 第一次响应：LLM 返回结构化工具调用

LLM 如果选择 `sales-report`，会通过普通 Function Calling 返回如下调用，而不是仅仅说一句“我要使用该技能”：

```json
{
  "role": "assistant",
  "tool_calls": [
    {
      "id": "call_1",
      "type": "function",
      "function": {
        "name": "skill",
        "arguments": "{\"name\":\"sales-report\"}"
      }
    }
  ]
}
```

两个 `name` 不要混淆：

- 外层 `function.name = "skill"`：要执行哪个工具。
- 参数里的 `name = "sales-report"`：这个工具要加载哪一份技能。

此传输格式的 `arguments` 是 JSON 字符串；解码后就是 `{"name":"sales-report"}`。Harness 的 adapter 将响应转换成内部工具调用块，Agent loop 再交给工具执行系统。

### 4.2 Harness 执行 `skill` 工具：只是取得正文

`tool-skill` 的执行函数依次做：

1. 校验名称，并确认该技能在当前 Agent / 工作目录下可用且允许模型调用。
2. 调用 `ctx.skills.get("sales-report", ...)`。
3. 技能服务找到之前记录的来源与位置，调用该来源的 `get(...)`。
4. 文件来源重新读取对应 `SKILL.md`，返回正文及资源目录。
5. 工具将内容包装为 `<skill_content>`，作为 `call_1` 的工具结果。

工具结果示意，仍沿用前面的传输风格并缩短了正文：

```json
{
  "role": "tool",
  "tool_call_id": "call_1",
  "content": "<skill_content name=\"sales-report\">\n<skill_resources>资源目录：C:\\demo\\.agents\\skills\\sales-report</skill_resources>\n<skill_instructions>先确认口径并检查字段；需要生成报告时运行 scripts/build-report.mjs，完成后核验输出。</skill_instructions>\n</skill_content>"
}
```

`skill` 工具至此已经完成，但报告还没有生成。文件消失、名称无效或不允许调用时会走错误路径，而不是返回“加载成功”。

### 4.3 第二次请求：现在才把正文交给 LLM

正常续接时，Harness 再请求一次模型，相关历史按这个顺序出现：

```text
Request 2
  messages:
    user       -> Original task
    user       -> Available skill summaries
    assistant  -> skill(name="sales-report"), call_1
    tool       -> Full instructions and resource base, reply to call_1
  tools:
    skill, pwsh, and other visible tools
```

**LLM 在这次请求中读到操作方法，才继续规划具体动作。** 没有新建一个“技能 Agent”，也没有让 LLM 在第一次响应发出后直接修改那份已结束的请求。

### 4.4 第二类调用：请求普通工具真正做事

假设已确认输入、输出与统计口径，完成字段检查，环境已有 Node 和示例脚本；本配置还启用了 PowerShell 工具及执行器，允许这些文件操作。

LLM 接下来可以发起另一条工具调用：**工具名为 `pwsh`，不再是 `skill`**。为避免长字符串转义，下面只展示这次调用解码后的参数：

```json
{
  "description": "Generate the sales report from the validated CSV",
  "command": "node 'C:\\demo\\.agents\\skills\\sales-report\\scripts\\build-report.mjs' --input 'C:\\demo\\sales.csv' --output 'C:\\demo\\report.md'",
  "workdir": "C:\\demo"
}
```

它仍通过 `tool_calls` 返回，只是 `function.name` 换成 `pwsh`，并使用新的调用 ID。这里的工具名和参数来自 Harness 的 `tool-pwsh`；脚本、文件和命令用途是教学假设，不是已执行操作。

随后：

1. Harness 处理这条 **`pwsh` 工具调用**，按已装配的权限 / sandbox 策略决定是否允许执行。
2. 允许后，由实际 shell 执行器启动命令；不是模型自己运行程序。
3. Harness 将退出状态、stdout / stderr 等结果回填，交给后续模型请求。
4. Agent 根据结果检查产物；失败则处理错误，不能因为“加载过技能”就宣称报告完成。

**不存在一条自动的“加载技能 → 执行其中所有脚本”的链。** `SKILL.md` 不是入口程序；指令可以让 Agent 读取参考资料、调用文件工具、运行脚本，也可以只指导它组织答案。实际动作由后续普通工具调用落实，所需工具必须真实可用。

## 5. 为什么分成这些步骤？渐进式披露就在这里

| 阶段 | 新提供给 LLM 的内容 | 尚未自动发生的事 |
|---|---|---|
| 发现目录后 | 名称与描述 | 未把所有正文塞进上下文 |
| 选择并加载后 | 指定技能正文、资源位置 | 未读取所有参考资料、未执行脚本 |
| 某一步需要资源时 | 选定的资料内容，或工具执行结果 | 未自动遍历所有资源 |

**设计分析**：全量注入会让无关方法占据上下文；完全隐藏又会让模型不知道方法存在。因此先给目录，需要时再取正文和资源。

代价是多了加载往返，LLM 也可能选错或漏选；“请求中要求匹配时先加载”不等于所有模型一定遵守。正文进入普通历史后也可能继续被携带，**按需加载不等于用完自动忘掉**。

至此只需记住：**Harness 负责找和提供；LLM 负责选择和提出动作；工具系统负责真正执行。**

<details>
<summary>补充：用户显式调用、缓存与作用域、Office 实例（主流程清楚后再读）</summary>

### 用户能否跳过“LLM 选择”？

可以。若用户输入 `/sales-report 把这份 CSV 汇总成报告`，且技能允许用户调用，pre-step 可直接加载正文并注入 **user-role 上下文**。这样首次请求就能包含正文，不需要 LLM 先调用 `skill`。

这是另一条入口，不能与上面的“模型调用工具 → tool-role 结果”混成一条链。它只检查接纳消息中 `source.kind === 'user'` 的文本；实际手势按空白边界识别 `/name`，不局限于首行。用户调用与模型调用的允许条件分别控制。

### 会不会每个 step 都重新扫描、重新发目录？

- 查询可复用按 cwd、scope 链与 revision 区分的目录缓存，不是每次全量扫描，也不是固定 TTL。
- 目录不变时不重复追加；变化时发布替换目录。最初为空则不发，已有目录变空则发撤销信息；发现不完整时保留 last-good。
- watch 默认启用，相关文件变动会使目录失效；受控编辑路径也可通知失效。普通资源子树变化不等于目录变化。
- 正文加载会重新读取文件，但已经进入历史的旧正文不会被追溯替换。

### 同名技能和不可见技能怎样处理？

较近 scope 优先，同一层再按 rank / 注册顺序选择。filesystem 默认 rank 为：项目 `.dsh\skills` 100、项目 `.agents\skills` 200、自定义目录 300、用户 `.dsh\skills` 400、用户 `.agents\skills` 500、配置的 bundled 目录 600。

`tool-skill` 只发布允许模型调用的技能摘要，描述默认最多 500 **字符**。当前 Agent 看不到此插件注册的 `skill` 工具，或它被同名 scoped 工具遮蔽时，不发布这套加载指引。scope 可见性不等于文件访问授权。

无效元数据会告警并跳过，不是把所有 Markdown 都当成合法技能。文件来源也不是唯一来源，其他插件可以向技能服务提供内容；非文件来源的资源定位可能是 URL 或 opaque 描述。

### 实际 Office 技能怎样对应这条线？

`skill-office` 提供 `office`、`office-docx`、`office-pptx`、`office-xlsx`、`office-render`、`office-recalculate`、`office-export-pdf`。

一种选择顺序为：先加载 `office` 了解工作流，再加载 `office-docx` 取得编写方法，需要预览时再加载 `office-render`，最后请求普通工具执行渲染。纯渲染任务也可直接加载 operation；不是宿主看到文件后缀就自动注入所有正文。

本章正文采用 native tool calling。PTC 模式可通过 `run_code` 内的工具绑定调用同一个加载能力，外层格式不同，不改变“先取指令、后做动作”的区分。

</details>

<details>
<summary>源码与验证边界：按流程定位，不必逐个通读</summary>

| 本文步骤 | 核对位置 |
|---|---|
| 来源注册、根目录、扫描与文件读取 | [skill-filesystem](../../projects/deepseek-harness/packages/skill/skill-filesystem/src/index.ts)：`apply`、`list`、`discoverRoot`、`get`；[配置说明](../../projects/deepseek-harness/packages/skill/skill-filesystem/README.md) |
| 目录汇总、名称到来源的解析 | [技能服务](../../projects/deepseek-harness/packages/skill/skill/src/index.ts)：`snapshot`、`collect`、`get` |
| 目录消息、工具定义、正文加载 | [tool-skill](../../projects/deepseek-harness/packages/skill/tool-skill/src/index.ts)：`apply`、`renderCatalogMessage`、工具 `execute` |
| 消息进入历史，再发起模型请求 | [Agent loop](../../projects/deepseek-harness/packages/core/agent-loop/src/agent.ts)：`step`、`buildRequest` |
| 后续 PowerShell 调用 | [tool-pwsh](../../projects/deepseek-harness/packages/shell/tool-pwsh/src/index.ts)：`pwsh` 定义、参数与执行路径 |
| 真实分层技能 | [Office README](../../projects/deepseek-harness/packages/skill/skill-office/README.md)与[请求内容测试](../../projects/deepseek-harness/packages/skill/skill-office/tests/progressive-loading.spec.ts) |

Office 测试用 MockAdapter 检查“第一次只有摘要 → 加载后才有对应正文”，覆盖 native / PTC 与显式用户调用；还检查从保留历史构造新 Session 后指令仍在。后者不等于完整进程重启恢复测试，也不证明真实模型总能选对技能。本轮只读源码与测试，未执行测试。

后续实践应直接检查各次 request 的目录、正文与工具结果，并验证“只加载、不执行时没有生成报告”“执行被拒绝时不会绕过策略”。本章示例不代表已通过这些验收。

</details>
