# Harness 实现阅读笔记：规划与上下文（延后阅读）

> 2026-10-15 从概念讲义中移出，避免在尚未建立项目全貌时引入内部术语。
> **不是概念课前置，也不是完整项目教程。** A＋B 概念讲完后，先建立项目总体架构，再用这里的固定版本入口核对局部机制。
> 以下保留此前已核对的公开文档 / 代码事实及限制，没有运行项目或验证完整调用链。

## 1. DeepSeek Harness：todo 工具保存什么？

固定 commit：`571902028a75c64c1d767f4ae5fd099485f0ddf0`。

已读 [todo 包说明](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/packages/todo/tool-todo/README.md)、[类型契约](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/docs/subsystems/todo.md)和 [src/index.ts](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/packages/todo/tool-todo/src/index.ts)的相关段落：

- 一个 todo 项是 content + status，状态为 pending / in_progress / completed。
- 模型每次提交整份列表，不是修改某个稳定 ID 的节点。
- 执行路径校验列表与调用身份，再向所属 Agent session 追加 `todo/write`。
- 是否允许多个 in_progress 由配置决定。这限制的是进度列表，不自动证明底层动作没有并行。

关键操作：

```typescript
exec.agent.session.append('todo/write', { todos })
```

它记录进度声明，不是执行任务、调度 DAG 或检查报表金额。

包说明强调列表随 session 保存；固定版本源码的 `todos` projection 则在 `todo/write` 时取新列表，在后续 `turn/start` 时返回 null。因此后续需要区分：

1. 历史 `todo/write` 事件是否保留。
2. 当前 projection / 界面是否仍显示上轮计划。
3. 下一轮模型是否通过某条上下文路径获得该计划。

本次没有追完 UI / 上下文路径；不能从事件持久化直接推出每轮自动沿用上一轮可见清单。正式阅读时用回放与跨 turn 测试核对。

## 2. DeepSeek plan mode 与执行权限

固定版本 [plan 包说明](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/packages/plan/README.md)说明：plan mode 引导先探索、设计并请用户审阅，但工具仍可用；sandbox 与 approval 限制单独配置。

“先规划”的交互模式，不等于技术上禁止文件写入或命令执行。批准前不改文件，需要执行端权限保证。

## 3. Deep Agents：组装层

固定 commit：`825acb3ca03bd103e4df6013b122eca8475d1982`。

[graph.py](https://github.com/langchain-ai/deepagents/blob/825acb3ca03bd103e4df6013b122eca8475d1982/libs/deepagents/deepagents/graph.py)定义 create_deep_agent，装配工具与中间件，最后调用底层 create_agent。文件说明包含 planning、filesystem、subagent、summarization 等能力。

后续追踪规划工具在哪层加入、计划怎样进入状态与模型上下文，以及谁负责依赖与完成检查。本次未追完路径，不把 planning 字样当作自动可执行 DAG。

## 4. DeepSeek：context、spill 与 compaction

固定 commit 同第 1 节。

| 入口 | 已核对职责 | 不能直接推断 |
|---|---|---|
| [packages/context](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/packages/context/README.md) | 指令文件、会话引用、文件提及等；区分默认挂载与 opt-in | 可发现文件自动全文进入模型 |
| [packages/spill](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/packages/spill/README.md) | 存储服务、本地后端与工具结果策略分开 | 所有结果默认 spill，或定位符包含全文 |
| [packages/compaction](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/packages/compaction/README.md) | 旧历史摘要、手动 compact、输出裁剪等；token 测量另由 LLM 家族服务负责 | 所有历史仍同样可见 |

context 文档说明，相关注入可作为 user-role 消息进入 session 历史；这是该项目的组织方式，不代表任意资料应获得可信应用规则权限。

spill 文档说明工具结果策略通过 `maxInlineTokens` opt-in，存储失败时保留原结果；其他消费者也可直接使用存储服务。配置不同，行为未必相同。

## 5. Deep Agents：存储压缩与模型上下文压缩

固定 commit 同第 3 节。

- [filesystem.py](https://github.com/langchain-ai/deepagents/blob/825acb3ca03bd103e4df6013b122eca8475d1982/libs/deepagents/deepagents/middleware/filesystem.py)：已读段落包含大工具结果落盘与分段读回提示、相关模块连接。
- [summarization.py](https://github.com/langchain-ai/deepagents/blob/825acb3ca03bd103e4df6013b122eca8475d1982/libs/deepagents/deepagents/middleware/summarization.py)：自动 / 主动历史压缩，旧消息摘要与 backend 历史回查；类型允许历史落盘失败时 file_path 为 None。
- [_blob_offload.py](https://github.com/langchain-ai/deepagents/blob/825acb3ca03bd103e4df6013b122eca8475d1982/libs/deepagents/deepagents/middleware/_blob_offload.py)：二进制内容按哈希保存，状态改用引用，模型请求又可从 backend 恢复内容。

所以 checkpoint 变小，不代表实际发送给模型的媒体内容变小。后续要检查真实请求、状态快照与读回流程，而非只看“压缩”功能名称。

## 6. 正式阅读时的顺序

项目定位与运行方式 → 核心对象与生命周期 → 一次完整 loop → 状态与工具 → 本笔记的规划 / 上下文模块 → 测试验证。

Rowboat / OpenClaw 另按办公 / 个人方向阅读，不假定其计划与上下文表示和这里相同。

关联：[学习路线](index-2026-10-08.md)、[产品阅读报告](agent_product_reading_workbuddy_muse_2026-10-15.md)。本笔记仅保留可回查资料，不要求现在阅读。
