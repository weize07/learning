# DeepSeek Harness 入门：项目全貌、目录与核心概念

- 学习日期：2026-10-15。
- 阅读阶段：A＋B 概念之后的独立项目课，第 1 篇。
- 每日概览：[topics_by_date](topics_by_date.md)。
- 固定阅读版本：`571902028a75c64c1d767f4ae5fd099485f0ddf0`，不是自动跟随 master。
- 状态：根据官方 README、架构和组件文档整理；未克隆、安装、启动项目或验证运行行为。

本篇先建立全貌，不要求先读之前移出的零散源码笔记。先读第 1～4 节，再看术语和一轮任务。项目处于 developer preview，后续版本的目录、配置与行为可能变化。

## 1. 这是一个什么项目？

[DeepSeek Harness 官方仓库](https://github.com/deepseek-ai/deepseek-harness)提供模型之外的 Agent 应用支撑。官方命令名为 `dsh`，核心特点是“everything-is-a-plugin”，即功能通过插件装配。

先与熟悉的对象区别：

| 对象 | 在这里是什么 |
|---|---|
| 语言模型 | 接收输入，生成回答或工具调用建议；由配置的模型服务提供 |
| Harness | 组织模型、工具、会话、上下文与执行流程的程序 |
| Web / Desktop / SDK | 与 Harness 交互或集成的不同入口，不是新的基础模型 |
| Cordis | Harness 使用的插件框架，管理功能模块如何组合 |

它不是 DeepSeek 模型权重仓库，也不是“安装后模型在本机免费运行”。模型如何提供、是否联网或计费，要另看实际配置。

“由 DeepSeek 发布”也不等于所有模型请求必须使用同一款 DeepSeek 模型。架构将模型接口与适配器分开；具体 provider / model 支持必须查该版本配置。

### 1.1 可以把它想成什么？

把模型比作提出下一步的决策组件，Harness 则提供：

- 怎样把用户输入交给模型。
- 告诉模型有哪些工具，收到调用后怎样处理。
- 怎样保存交互，构造下一次请求。
- 怎样接文件、命令、确认机制与界面。

不是某个插件自己包办全部任务；各部分通过约定接口协作。

## 2. 一张图分开“启动装配”和“执行任务”

![DeepSeek Harness 的运行配置、插件装配与 Agent 执行职责图](images/deepseek-harness-overview.svg)

自绘简化图，依据本篇固定版本文档。箭头帮助理解数据流，不表示已验证的逐函数调用链。

**启动时：** 决定装哪些功能、怎样连接模型与工具。

**任务运行时：** 用户发请求，循环调用模型与工具，记录结果，再决定继续或结束。

这张图混合了两个视角：上方讲“程序如何装配”，下方讲“装配出来的程序如何协作”。**plugins 不是与工具服务并列的另一个业务子系统。** 工具服务、执行策略、Agent loop、模型适配和 Session 等职责，本身也由插件提供。它们被分别画出，是为了展开运行时分工，不是表示插件只在启动时存在、工具服务才在运行时存在。

例如，启动时装载工具相关插件，注册工具服务、具体工具及配置的策略；任务执行时，loop 使用这些已装配的服务处理调用，经相应校验与执行流程取得结果。插件及其服务会在运行期间继续发挥作用；具体哪些服务和工具被启用，仍取决于配置。图中的“工具服务与执行策略”也不是单个插件名，一个职责块可以概括多个插件的协作。

“图上有这个模块”不等于任意 profile 都启用它。权限、持久化、模型和工具能力都要看实际装配。

### 2.1 入口与运行过程是双向交互

图中向右的“任务 / 控制”表示调用方提交任务，以及入口所支持的取消、确认等控制请求；向左的“状态 / 回复”表示运行状态、流式输出、工具进度、需要确认的提示、最终回复或错误等反馈。具体可见内容与控制能力取决于入口、配置和协议，不代表每个 profile 都支持全部交互。

返回箭头是逻辑通道的简写，不意味着 Agent loop 直接操作浏览器或实现网络传输。应理解为“运行过程产生输出或事件 → 入口层通过相应协议暴露 → 界面展示或调用程序消费”；中间可能经过会话事件、订阅和协议适配，完整源码链路仍需后续追踪。运行结束、最终文本回复与业务任务成功也应分别判断，不能仅因收到结束通知就认定目标已完成。

### 2.2 沙箱限制什么？修改是否需要“搬回”工作区？

图中原来的“依配置落实”过于含糊，改为“后端限制写入；审批按策略执行”。意思是：配置选择限制模式和审批政策，实际调用时由文件后端、进程执行后端和审批流程执行，而不是仅在 prompt 中告诉模型“不要越界”。配置本身不构成隔离，具体后端及其限制范围才决定实际效果。

**沙箱不等于工作区副本，也不自动带有修改合并功能。** 需要区分：

| 执行方式 | 修改发生在哪里 | 如何进入原工作区 |
|---|---|---|
| 在原文件系统上限制权限 | 被允许写入的真实文件 | 已经直接生效，不需要搬回 |
| 容器 / VM 挂载原工作区为可写目录 | 挂载对应的真实文件 | 同样直接生效；容器不自动隔离这个目录的修改 |
| 独立副本、独立卷或远端工作区 | 与原工作区分开的文件 | 应用另行导出 diff / 产物，审查、检查版本并应用，或走提交与合并流程 |

独立 Git worktree 可分开工作文件，但共享仓库相关资源，不能仅凭 worktree 就宣称具有安全隔离。无论哪种方式，沙箱都不自动提供备份、回滚或事务。

**本篇固定版本的本地进程沙箱主要属于第一种。** 官方称其为 same-world confinement：与宿主共享内核和文件系统，对子进程施加文件操作限制，不是默认新建容器或文件副本。例如，真实项目文件允许写入后，工具修改它就会直接改变原工作区。远程执行、容器和 microVM 是另一类执行后端的设计，不能从“sandbox”一词自动推断它们已启用。

该项目的文件影响模式如下，具体执行范围与强度仍需看后端：

| 模式 | 文件写入边界 | 不应推断的保证 |
|---|---|---|
| `read-only` | 文件后端拒绝修改；进程后端按平台限制写入，存在必要系统例外和平台限制 | 不等于禁止读取敏感文件、禁止联网或无外部副作用 |
| `workspace-write` | 允许工作区及后端规定的临时区域写入，拒绝范围外写入 | 不保护工作区内文件免于错误修改或删除 |
| `danger-full-access` | 不施加这层文件沙箱限制，仍受操作系统权限等约束 | 不等于获得管理员权限，也不意味着安全 |

文件工具和命令执行有两条不同的防线：`fs-sandbox` 在受信任的文件操作代码中检查写入路径；进程沙箱通过平台后端限制运行的命令。只限制文件编辑工具，不代表 shell 命令自动受同样限制。官方文档还明确：文件后端的读取保持本地后端行为，不能把写入边界误当成读取隔离。

**网络、进程可见性和业务授权不是上述三个模式的含义。** 发 HTTP 请求、调用邮件或订单 API、访问数据库等，需要分别检查网络策略、可用工具、凭据权限、服务端授权与审批。即使本地文件只读，持有有效凭据的请求仍可能改变远端系统；反之，允许本地写代码也不应该自动授予生产部署或付款权限。

人工确认解决的是“是否授权某次敏感操作”，不是“把沙箱里的文件同步出去”。例如一次越界写入被拒绝后，该项目支持在获得相应批准后以更宽模式重试一次；并不是每次工作区写入都必须询问，也不是用户批准后永远解除所有限制。沙箱模式与审批政策是独立配置维度；有批准也不保证操作一定成功。对于真正的副本式执行，应用可以在将 diff 应用回原工作区之前增加单独的确认与冲突检查，但这不是所有沙箱的固有流程。

安全边界：官方说明该项目尚未经过安全审计。进程后端会报告完整或部分执行强度；Windows ACL 后端存在文档列出的部分限制，不能等同于强隔离 VM。要求受限执行却没有可用后端时，契约要求明确失败，不能悄悄改成无限制执行。本文依据文档说明，不表示已经验证本机运行效果。

固定版本依据：[进程沙箱](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/docs/subsystems/sandbox.md)、[文件写入限制后端](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/packages/fs/fs-sandbox/README.md)、[权限预设与审批维度](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/docs/subsystems/permission-presets.md)、[官方安全说明](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/SAFETY.md)。

## 3. 先看两条主线，不急着记所有目录

### 3.1 启动主线

```text
选择运行配置 profile
        ↓
按配置装配一组 bundle / plugins
        ↓
连接模型、工具、会话及所选入口
        ↓
等待或接收任务
```

这是“把程序组织起来”，不是“让模型制定业务计划”。

### 3.2 任务主线

```text
入口收到请求
        ↓
Agent loop 读取会话、准备模型输入和工具定义
        ↓
LLM 服务连接所选模型，取得响应
        ↓
如有工具调用：经工具执行流程取得真实结果
        ↓
结果记入会话，再构造后续请求
        ↓
继续、等待或结束
```

这条线对应我们学过的最小 loop。具体输入排队、取消、并行和恢复逻辑，下一篇再追。

## 4. 仓库怎么分层？

以下是该固定版本的目录地图，只选当前需要的部分。路径指上游仓库，不表示本机已有 checkout。

```text
deepseek-harness
├─ README.md / SAFETY.md       定位、运行入口与安全边界
├─ docs                       架构、开发与子系统说明
├─ packages                   主要能力包
│  ├─ boot / bundle           启动与功能组合
│  ├─ core                    会话、提示、工具、Agent 与循环等
│  ├─ llm                     模型接口与适配能力
│  ├─ fs / shell              文件与命令相关能力
│  ├─ web                     网页搜索与 HTTP(S) 抓取，不是交互式浏览器
│  ├─ browser-use             浏览器 provider 的独占注册接口
│  │  └─ browser-use          共享注册服务，不提供统一的页面操作方法
│  ├─ computer-use            桌面 provider 的独占注册接口
│  │  └─ computer-use         共享注册服务，具体操作由 provider 提供
│  ├─ storage                 存储能力家族，具体后端需另查
│  ├─ plan / todo             规划交互与进度清单
│  ├─ context / compaction
│  │  / spill / skill         上下文、摘要、落盘与技能等
│  └─ experimental            需显式启用的实验性能力；以下仅列部分
│     ├─ browser-use-playwright-mcp       通过 Playwright MCP 操作浏览器
│     ├─ browser-use-chrome-devtools-mcp  浏览器检查、调试与控制
│     ├─ browser-use-stagehand-native    带 AI 辅助的浏览器操作
│     ├─ browser-use-runtime             浏览器集成共享的会话资源管理
│     ├─ computer-use-cua-driver-mcp     通过 MCP 接入 Cua Driver
│     └─ computer-use-cua-driver-native  接入 Cua Driver 原生运行时
├─ apps                       桌面等应用入口相关代码
├─ python                     Python 集成相关代码
├─ vendor                     纳入仓库的依赖源码
└─ benchmarks / scripts       评测、构建与辅助脚本
```

当前优先顺序：

1. 根 README、SAFETY。
2. docs 的总体架构与 Cordis 入门。
3. core 的 Agent loop、session、tools，加上 llm。
4. 一条完整任务走通后，才回到 plan、todo、compaction 等局部机制。

core 是能力分组名，不代表一份凌驾插件机制的特殊单体程序；官方架构说明 loop、工具注册与模型适配等本身也由插件提供。

阅读浏览器与桌面相关目录时，注意三点：

- **读取网页与操作网页不是同一种能力。** `web` 提供搜索和抓取；填写表单、点击、提交等交互需要浏览器 provider。`apps` 中的 Web / Desktop 入口是 Harness 自身的界面或载体，不等于操作其他网页或桌面应用的能力。
- **公共注册接口不等于完整实现。** `browser-use` 和 `computer-use` 的共享服务主要负责唯一 provider 注册；具体工具、配置及平台要求由 provider 提供。此处列出的具体 provider 位于 `experimental`，需要显式装配，目录存在不代表当前 profile 已启用。桌面 provider 的唯一注册也不代表某个 Session 自动独占整个桌面。
- **E2E 测试不一定走 browser-use。** 对 Web 应用，可以让 Agent 编写并通过命令工具运行 Playwright 等确定性测试脚本，由测试框架启动浏览器；也可以通过浏览器工具做探索式验证。前者便于沉淀明确断言与可重复回归，后者便于观察和调试。运行条件和权限仍需另行配置，原生应用则需考虑对应测试框架或 computer-use。

固定版本入口：[web 能力目录](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/packages/web/README.md)、[Browser use 子系统](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/docs/subsystems/browser-use.md)、[Computer use 子系统](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/docs/subsystems/computer-use.md)。这里只补齐目录定位，不表示已安装或验证这些集成。

## 5. 启动配置的三个词：profile、bundle、plugin

用“配置一台工作站”的类比理解，但不要把它们当成模型概念：

| 术语 | 简单解释 | 在本项目中的作用 |
|---|---|---|
| profile | 选择哪种运行组合 | 命名的运行配置，列出要使用的组合及覆盖配置 |
| bundle | 预先整理的一组能力装配 | 提供配置行与对应代码，仍可由上层配置调整 |
| plugin | 一个可装载的功能模块 | 提供服务、工具、监听或其他行为 |

**例子：Web 入口。** 官方架构文档说明，web profile 以共享 base 组合为基础，再加入 Web 应用能力。base 包含模型适配、工具、持久化、策略等组成。

profile 不是用户账号，bundle 不是模型权重包，plugin 也不必是一个模型能直接调用的工具。某个插件可以只负责存储，没有任何模型可见函数。

该版本还提供 headless、sdk、sdk-minimal、acp 等 profile：

- headless 面向一次性运行，没有同样的 Web 服务入口。
- sdk 用于程序集成；Python 客户端也接入正常 Harness 运行时，不等于完整运行时用 Python 重新写了一遍。
- sdk-minimal 是有意设计的独立组合，并不套用共享 base。
- acp 是面向另一客户端协议的入口，本篇不要求掌握协议细节。

**它们不是按功能强弱从低到高的等级。** 更换 profile 可能改变实际能力，不能仅凭名字推断安全或持久性。

### 5.1 profile 与 bundle：运行方案与可复用装配层

两者都会影响最终配置，区别不是“profile 只管参数，bundle 只管功能”。

| 问题 | profile 的职责 | bundle 的职责 |
|---|---|---|
| 本次以什么运行方案启动？ | 提供一个可选择的命名组合，例如 web、sdk | 作为该组合使用的装配层，不代表一次运行 |
| 选择哪些能力、按什么顺序组合？ | 列出按序叠加的 bundles | 提供一组插件配置行及所挂载的代码 |
| 能否配置插件参数？ | 可通过 profile 自己的 patch 调整 | 自带插件配置，也可被上层 patch 调整 |
| 如何复用？ | 不同 profile 可选择同一个 bundle | 同一组装配可供多个 profile 复用 |

可以把 profile 理解为“这台应用的运行方案”，bundle 理解为“方案里选用的可复用装配层”，plugin 才是实际提供行为的模块。

以该版本的内置组合为例，以下是关系示意，不是可直接执行的配置语法：

```text
web profile
  ├─ dsh-base bundle       → 共享基础插件与配置
  └─ dsh-web-app bundle    → Web 应用插件与配置

sdk profile
  ├─ dsh-base bundle       → 复用同一基础装配层
  └─ dsh-sdk-app bundle    → SDK 服务插件与配置
```

因此，bundle 也可以包含 Web 或 SDK 等入口能力，并不只包含底层工具。profile 也不只选择 UI，它选择的是整个运行组合。

官方文档给出的装配顺序是：空配置列表 → profile 指定的有序 bundles → profile 自己的 patch → Harness home 的 patch → 命令行 `--patch`。加载器再用最终配置启动插件树；不是模型在任务执行中挑选 bundles。这里的覆盖按配置行规则执行，不应理解成任意字段的递归合并。

实用判断：

- 想把已有 Web 方案的某个插件配置改掉：通常先用配置覆盖，不必复制整个 bundle。
- 想让多个运行方案复用一组插件与默认配置：适合整理成 bundle。
- 想维护一套独立选择的运行方案：适合使用 profile，并选择相应 bundles 和覆盖配置。
- 想新增实际行为：实现或引入 plugin，再通过配置把它装入运行组合。

依据：[官方架构说明的 Profiles and bundles](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/docs/architecture.md#profiles-and-bundles)、[bundle 目录说明](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/packages/bundle/README.md)。这是固定版本的配置职责说明，不表示本机已运行验证。

## 6. Cordis：先只理解三个协作概念

### 6.1 Context / ctx：可取得共享服务的程序对象

在 Cordis 中，ctx 是插件查找服务、注册行为的上下文。它不是发给 LLM 的 token 上下文。

例如文档里的 `ctx.tools` 表示工具服务，`ctx.llm` 表示模型相关服务。一个插件通过服务接口合作，不必直接导入另一个具体实现。

> **程序 ctx ≠ 模型上下文窗口。**

这是本项目术语中最需要先分开的地方。

### 6.2 Service：对外提供一类能力

工具服务负责工具相关接口；模型服务负责模型调用接口。插件可以提供或使用服务。

如果插件需要某个服务，框架根据声明的依赖安排激活；不是让模型自己猜“先启动哪个模块”。

### 6.3 Event：模块之间传递发生了什么或介入某个阶段

例如有新会话事件、Agent 请求阶段或工具执行阶段，相关模块可按契约记录、观察或介入。

不是所有 event 都自动写磁盘，也不是所有监听器都只看不改。具体事件的执行顺序与返回语义后续按调用链讲，本篇不用背 emit、waterfall 等分发名。

Cordis 还管理插件加载、卸载和注册清理。卸载一个模块能清理程序注册，**不意味着自动撤销它曾执行的外部业务操作**。

## 7. 执行时的三个词：Agent、Session、Step / Turn

### 7.1 Agent：活跃的执行者；Agent loop：驱动它的方法

Agent 接口描述运行中的执行对象；默认 agent-loop 提供一种“请求模型、运行工具、继续”的驱动实现。它不是指模型文件，也不是 UI 中随便取的角色名字。

### 7.2 Session：交互记录，不只是聊天字符串

该版本 session 以追加事件记录事实，再从记录中推导给模型的历史。

可先理解为：

```text
记录：用户输入、模型响应、工具结果及其他事件
                         ↓
推导：当前模型应该看到的消息
```

并非每个记录事件都变成模型消息；历史摘要也可能改变当前可见历史，但不等于删除原记录。

重要边界：**基础 session 包在内存中保存实例，跨进程持久化依赖额外后端。** 不能因用了 Session 或 event log，就断言程序重启后一定可恢复。

### 7.3 Step 与 Turn：本项目有自己的定义

官方架构定义：

- step：一次模型请求，加上它触发的工具执行。
- turn：零个或多个 step；开始于认领输入之前，在没有待履行工作时结束。

教学例子：

```text
用户提出读文件任务
   step 1：模型建议列文件 → 执行
   step 2：模型建议读取某文件 → 执行
   step 3：模型给出回答
本次 turn 结束
```

这不是说所有 API 的 message、所有框架的 turn 都同义。一个用户任务可以跨多个 turn；turn 结束也不等于外部业务目标已验收。

## 8. 用一个只读任务认识各部分

请求：

> 读取授权工作区内的销售表和说明，告诉我金额单位是否明确。不要修改文件，也不要计算最终报表。

概念执行过程：

| 步骤 | Harness 中的角色 | 结果 |
|---|---|---|
| 启动并配置 | profile / bundles / plugins | 实际工具、模型与入口可用 |
| 接收请求 | 入口与 Agent | 开始处理本次输入 |
| 组织模型请求 | loop、提示与会话相关服务 | 模型看到必要消息与可用工具 |
| 模型建议查文件 | LLM 服务 / 适配器 | 返回结构化调用建议 |
| 校验和读取 | 工具服务与实际文件能力、执行策略 | 获准时返回真实文本或错误 |
| 保存并继续 | session 与 loop | 根据结果提出下一步或回复 |
| 回答单位是否明确 | 模型与应用完成检查 | 说明来源与不足，不越权修改 |

本表是根据组件职责组织的教学数据流，不是已运行轨迹或逐函数顺序证明。具体调用关联与事件写入见 [下一篇：一次读文件任务的主链](2026-10-16_01_deepseek_harness_task_lifecycle.md)；失败分支再单独展开。

如果工具未安装、未挂载或被策略禁止，模型不能凭“项目支持文件工具”就直接读文件。配置与执行记录比宣传功能表更具体。

## 9. 怎么运行？先了解，不立即执行

固定版本 README 展示 npm 启动和源码启动两条路径。源码路径需要 Node.js、pnpm 安装依赖并构建，再用 dsh 选择入口。

本篇只认识命令含义：

```text
dsh web            选择 Web 入口的运行配置
dsh --profile ...  显式选择运行配置
```

这不是本次要执行的安装步骤。使用 npx 拉到的最新发布包，不自动等于我们固定的 commit；实际实践时需选兼容稳定版本并记录差异。

启动 Web 服务不代表已经配置可用模型，更不代表本地已有模型权重。模型路由、凭据、文件范围与策略需要另行确认；密钥不放入教材、仓库或模型上下文。

项目 SAFETY 明确：开发者预览、未完成安全审计，不应被当成安全或生产就绪系统。后续运行用隔离 / 可丢弃工作区和最小权限，不直接挂载真实个人资料。

## 10. 本篇先不研究哪些东西？

暂时不读：

- 所有插件配置和事件名。
- todo 的完整更新与投影规则。
- compaction、spill 的具体阈值。
- UI 渲染、桌面打包与所有 provider 实现。

先能说清：

> profile 选运行组合，插件提供能力，Cordis 管理它们的协作；loop 连接模型与工具，session 保存并推导交互历史。

不要急着打开 [此前移出的局部实现笔记](../harness_implementation_notes_2026-10-15.md)。有了完整执行链后，再用它研究规划与上下文会更连贯。

## 11. 资料、复习与进度

本次 2026-10-15 重新读取的固定版本入口：

- [README](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/README.md)。
- [Architecture](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/docs/architecture.md)。
- [Cordis Primer](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/docs/cordis-primer.md)。
- [Packages map](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/packages/README.md)。
- [Agent loop](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/packages/core/agent-loop/README.md)、[Session](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/packages/core/session/README.md)。
- [SAFETY](https://github.com/deepseek-ai/deepseek-harness/blob/571902028a75c64c1d767f4ae5fd099485f0ddf0/SAFETY.md)。

可选复习：

1. plugin 是否必然是模型能直接调用的工具？
2. ctx 是否等于 LLM 的上下文窗口？
3. Session 是否默认保证重启恢复？
4. 一个 turn 结束，是否证明业务任务成功？

<details>
<summary>参考答案</summary>

1. 不是，插件也可只提供存储或其他服务。
2. 不是，它是程序中的服务协作对象。
3. 不保证，基础包在内存中，持久保存需要后端与正确配置。
4. 不证明，业务完成仍需独立验收。

</details>

进度：

- 已提供：项目定位、启动 / 执行双主线、架构图、目录与最小术语表。
- 阅读反馈 / 答疑：已补充装配与运行职责、profile / bundle、双向反馈、沙箱边界以及浏览器 / 桌面能力目录等说明。
- 未执行：克隆、安装、模型接入或任务运行；未确认本机运行兼容性。
- 下一篇：[固定版本下的一次读文件任务主链](2026-10-16_01_deepseek_harness_task_lifecycle.md)，从 Agent 创建到模型请求、工具结果与下一步，介绍实际经过的源码符号。
