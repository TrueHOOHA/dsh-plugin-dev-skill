# DeepSeek Harness 架构

在改动 `packages/` 下的任何内容之前先阅读本页。它假设你已了解 Cordis；如果不了解，请从 [Cordis 入门](./cordis-primer.md) 或 [Cordis 教程](https://deepseek-harness.github.io/deepseek-harness/develop/cordis-tutorial/) 开始。

## Cordis

[Cordis](./cordis-primer.md) 是 dsh 底层的框架：插件向共享上下文贡献服务、类型化事件和可回卷的 effect。产品的每个部分都是插件——模型适配器、工具注册表、会话日志、agent loop（智能体循环）本身也不例外——因此每一部分都可以通过配置替换。

没有需要打补丁的特权内核：你通过在其它插件旁边挂载一个插件来扩展 dsh，注册都是 effect，会随所属插件卸载而回卷。

## Profile 与 bundle

运行中的 `dsh` 是一棵在启动时按有序层组合出来的插件树。

**profile** 是存储在 Harness home 中的一个命名组合。它列出所叠加的 bundle，保存自己安装的树外插件，并保留用户自己的 `cordis.patch.yml`。 `web` 、 `headless` 、 `sdk` 、 `sdk-minimal` 和 `acp` 以模板形式随附发布。

**bundle** 是 Cordis 配置行及其挂载代码的分发格式，它插入的内容因此始终可以被上层的 patch 覆盖。

二者都在各自的 `package.json` 中通过 `dsh` 字段声明自己： `dsh.profile` 列出 profile 的 bundle， `dsh.bundle` 指向 bundle 的 patch 文件。

[`dsh-base`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/bundle/base/README.md) 是 `web` 、 `headless` 、 `sdk` 和 `acp` profile 共享的第一层：模型适配器、工具、持久化、沙箱与审批策略、settings、凭据、遥测。 [`dsh-web-app`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/bundle/web-app/README.md) 添加浏览器应用， [`dsh-headless`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/bundle/headless/README.md) 添加无服务器的一次性运行器， [`dsh-sdk-app`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/bundle/sdk-app/README.md) 添加 SDK JSON-RPC 服务器， [`dsh-acp-app`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/bundle/acp-app/README.md) 添加仅自动化的 ACP 服务器。 [`dsh-sdk-minimal`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/bundle/sdk-minimal/README.md) 是刻意的例外：这一个 bundle 拥有完整、显式的 SDK 树，不套用 `dsh-base`。

各层依次应用于空条目列表：profile 的有序 bundle、它的 `cordis.patch.yml`、home patch，然后是 `--patch` overlay。patch 按 id 定位某一行并替换其整个 `config`，或插入新行。 `preset` patch 在该声明自己的 `config.plugins` 内执行这些操作，层优先级相同。

HMR 由 YAML 控制：base 默认启用仅配置的 `dsh-hmr`；headless、SDK 和 ACP 禁用它； `sdk-minimal` 完全省略。profile patch 可以覆盖这些默认值。HMR 负责协调监视与重载；启动器提供 profile 数据与就绪信号。

base 包含面向 Web 和 agent 的 [Plugin Manager](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/boot/plugin-manager/README.md)。

查看你机器实际启动的树：

```sh
dsh --profile web --dump-config
```

它打印的任何行都可以被你自己的 patch 替换。

组合机制见 [app-boot](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/boot/app-boot/README.md#profiles)；配置字段见生成的 [config catalog](https://deepseek-harness.github.io/deepseek-harness/reference/config-catalog)。

## 应用启动

受支持的 Node 应用通过命名的 `dsh` profile 启动。随附 profile 为 `web` 、 `headless` 、 `sdk` 、 `sdk-minimal` 和 `acp`，用 `dsh --profile <name>` 或 `dsh <name>` 选择。 `plugin` 是管理命令的名字；名为 `plugin` 的 profile 需要写 `--profile plugin`。TypeScript SDK 解析其同版本 `dsh` 依赖并选择 `sdk`；自定义插件组合始终是"一份 profile 加有序 patch 文件"，而不是另一个可执行文件或内联应用树。 `sdk-minimal` 是仓库自有的独立 bundle，走同一个启动器，不是调用方提供的 Cordis 树。

vender 出来的 CLI、仅构建和仅测试的可执行文件、进程内直接插件挂载、私有的浏览器 WebWorker 预览都不是 Harness 应用启动器。 [`verify-application-entrypoints`](https://github.com/deepseek-ai/deepseek-harness/blob/master/scripts/verify-application-entrypoints.ts) 把每个包 bin、可执行源文件、根 demo 和根目录的 `start:web` / `dev:web` 脚本归入明确的类别，并拒绝绕过 `dsh` 的 Node 应用路径。

Python SDK 遵循同样的应用架构。它的运行时 wheel 把常规 `dsh` CLI 打包为 `deepseek-harness-sdk-runtime-<platform>-<arch>`，客户端默认以显式 Harness home 启动 `dsh --profile sdk`。最小示例选择随附的 `sdk-minimal` profile。Python 暴露 profile 选择和有序 patch 文件，而不是完整的 Cordis 树；持久的外部插件通过 `dsh plugin` 安装。已移除的私有直连配置载体没有兼容 bin 或回退解析器。

## 桌面应用

[Electron 桌面应用](https://github.com/deepseek-ai/deepseek-harness/blob/master/apps/desktop/README.md) 把与生产完全一致的 dsh 运行时以签名资源随附，并拥有保留的 `$DSH_HOME/profiles/desktop`。共享的辅助逻辑负责初始化 profile 文件、对账 bundle、在不替换 pnpm 管理的包的前提下解析安装与 bundle 依赖。桌面端与 npm CLI 共享产品数据，但包、激活与锁文件相互独立。桌面端自带的 CLI 管理其初始化好的插件。

Electron 以 Electron Node 模式启动私有的 Desktop Host。Host 调用共享的 CLI profile 运行器和完整的 Web 应用。窗口立即加载打包的 Web 资源，并等待 boot 注入后在同一文档中激活客户端插件。Web 负责 RPC 与流；桌面载体把本地页面接入经过认证的 Host。Node IPC 承载 boot 注入、就绪、致命错误与关闭。桌面端默认监听操作系统分配的端口；profile 配置可以覆盖。Shell 自有的 UI 通过随附的 pnpm 执行插件事务，使用正常的用户与 profile 配置。

## 核心包

| 包 | 职责 | `ctx` 键 |
| --- | --- | --- |
| [`core/session`](https://deepseek-harness.github.io/deepseek-harness/reference/subsystems/session) | 追加式 `SessionEvent` 日志与内存存储 | `ctx.sessions` |
| [`core/system-prompt`](https://deepseek-harness.github.io/deepseek-harness/reference/subsystems/system-prompt) | 提示词区块与工具 schema 组装 | `ctx.systemPrompt` |
| [`core/tools`](https://deepseek-harness.github.io/deepseek-harness/reference/subsystems/tools) | 带作用域的工具注册表与受护栏的执行流水线 | `ctx.tools` |
| [`core/agent`](https://deepseek-harness.github.io/deepseek-harness/reference/subsystems/core) | `Agent` 接口、实时注册表和 `agent/*` 事件 | `ctx.agents` |
| [`core/agent-loop`](https://deepseek-harness.github.io/deepseek-harness/reference/subsystems/core) | 实现该接口的默认驱动 | `ctx.agentLoop` |
| [`core/scope`](https://deepseek-harness.github.io/deepseek-harness/reference/subsystems/scope) | 按 agent 作用域化注册的原语 | 库，无键 |
| [`llm/llm`](https://deepseek-harness.github.io/deepseek-harness/reference/subsystems/llm-streaming) | 消息与流式词汇表以及适配器 seam | `ctx.llm` |

## 事件

事件就是扩展点，为改动选择正确的事件域通常是第一步决策。

- **会话事件**是追加进日志、并经 `session/event` 广播的持久事实。当事实必须在重载后仍然存在时使用它。
- **Agent 事件**（ `agent/*` ）携带活的 `Agent`：inbox、step、status、request、validation、continuation。观察或拦截进行中的工作时使用它们。
- **能力事件**把策略与适配器挂到 seam 上（ `fs/*` 、 `tools/*` 、 `telemetry/*` ），无需导入 loop。

AgentLoop 在处理排队工作前会等待串行的 `agent/created`，并对失败的初始化做回滚。卸载期间资源会保留到 Agent 拆除；顺序由 [agent-loop](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/agent-loop/README.md#understand-the-implementation) 定义。

[事件图谱](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/event-producer-consumer.md)列出每个事件的生产者与消费者。

## Turn 流程

一个 **step** 是一次模型请求加上它调用的工具。一个 **turn** 是零个或多个 step：在它的第一个输入被认领之前开启，在没有欠账时关闭。

```text
turn/start
  claim next-step input plus one queued message
  assemble prompt sections + tool schemas
  -> agent/pre-step; project runtime context at fallback
                                     reject | enter(messages, startsRequestSeries?)
     reject, or a first enter rewritten empty -> close the turn with no step
     step/start
     agent/request -> prepareCall (cancellation commits neither system nor users)
     reconcile system/message using the prepared call capability
     append entered messages as user/message; reconcile retained runtime context
     log request/header and request/context as needed
     derive and freeze model history from the log
     stream the bound prepared call -> llm/stream -> agent/assistant-stream start
       agent/assistant-stream chunk*
       assistant/message | assistant/attempt -> agent/assistant-stream end
     tool/call* -> tools/pre-execute -> tools/execute -> tools/post-execute -> tool/result*
     step/end
     tools owe another request, or next-step input arrived -> claim -> next step
  -> agent/turn-stopping
turn/end
```

`turn/*` 、 `step/*` 、 `system/message` 、 `user/message` 、 `assistant/message` 、 `assistant/attempt` 和 `tool/*` 是持久的会话事件；其余是横跨三个域的实时扩展点。 `agent/assistant-stream` 发布进程本地的 start、瞬态 chunk 与 end 帧。loop 会在提交 end 帧之前，把完整的压缩流作为一条消息或仅记录日志的 attempt 提交；Web 的 Session-follow 适配器是该实时事件唯一的远端消费者。 `agent/pre-step` 、 `agent/request` 、 `llm/stream` 和三个 `tools/*` 事件是 waterfall，其监听器必须调用 `next()` 委托； `agent/turn-stopping` 是 serial，没有 `next()`。

一个 inbox 供给驱动器；注入的上下文等待唤醒消息。AgentLoop 的持久 `inbox` 投影在不依赖活的 Agent 的情况下暴露待处理输入。

`agent/pre-step` 决定被接受的输入。监听器可以改写或拒绝被认领的消息；被拒绝或首个被改写为空的认领会关闭一个没有 step 的持久 turn。enter 决策可以设置 `startsRequestSeries`：loop 记录一条新的 `request/header`（原因 `series`，或当信封也变化时为 `change` 且 `startsSeries: true`）。包装型监听器用 `{ ...decision, messages }` 保留该声明。组装与 `step/start` 之后， `agent/request` 和 `prepareCall()` 在系统提示词与被接受的用户消息提交之前解析实际路由；这两个异步阶段中的取消两者都不提交。prepared call 能力治理提示词准入，而不是先前的 `request/context`。每次尝试都会刷新已注册的运行时事实、reconcile 绑定的提示词、一次性准入用户消息、恢复被移除的运行时上下文、按需记录 header/context，并在流式传输绑定调用前冻结请求。重试不会重复组装或 `agent/pre-step`。附件开始之后的 surface 替换与图片卸载决策会开启新的 request series，包括首次恢复的 pre-step；未变化的 resume 延续原 series。首个被准入的 step 即使提示词为空（无 wire 消息）也会在 user 消息之前保留 system 头部。提示词只作为 `system/message` 历史传输：空渲染会清空所有活跃的 system 节点，不会留下旧提示词对模型可见；有能力的路由在缓存历史之后追加非空更新（包括受支持的工具更新）；无能力的路由和新的 request series 会把非空提示词文本合并到第一个 system 节点，并对后续非空 system 节点记录空的替换（[决策](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-09-02-system-prompt-as-surface-node.md)；[决策规则](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/agent-loop/README.md#understand-the-implementation)）。

loop 发送带实时取消的不可变请求，并复用 freeze 证据作为冻结身份；构造与取消原因由 [agent-loop](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/agent-loop/README.md) 负责。

失败的 step 会[记录缺失的工具结果](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/agent-loop/README.md#understand-the-implementation)。

细节见[时序图](https://deepseek-harness.github.io/deepseek-harness/reference/agent-lifecycle)、[工具流水线](https://deepseek-harness.github.io/deepseek-harness/reference/tool-execution-pipeline)与[取消与错误恢复](https://deepseek-harness.github.io/deepseek-harness/reference/subsystems/core)。

## 会话日志

会话日志是模型所见上下文的源头。 `deriveMessages()` 从日志投影模型历史。每条 `assistant/message` 都内嵌产生其组装内容的完整压缩定时流； `assistant/attempt` 保留已定案的失败、重试、取消与流错误尝试，不增加模型历史。fork、resume、transcript、遥测和持久化都从这些持久定案派生，而实时 UI 的增量来自 `agent/assistant-stream`；在定案前发生的硬性进程损失不会留下持久的 attempt 流（[决策](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-09-01-v2-embedded-assistant-streams.md)）。

会话消费者只认识当前的逻辑格式。仅头部（header-only）的 `stat` 和 `list` 重扫每个 Session 目录，选择其数值最大的规范 generation，并在不加载事件、不发布后继的情况下翻译受支持的历史头部。存储会话的 `open` 选择同一个 generation、拒绝未来版本，或在返回经过校验的当前逻辑事件之前一次性解码并组合静态的相邻迁移链。读打开直接使用内存中的结果而不发布后继；写打开先编码、校验并独占发布以版本命名的最终后继，紧挨着未变的源文件。对未密封中断尾部的常规修复仍是 handle 消费方的责任；迁移只为后来已被 `turn/start` 密封的有界 released restart 插入缺失的中断 `turn/end`。JSONL v0 使用 `session.jsonl[.zstd]`，v1 及之后使用小写的 `session.vN.jsonl[.zstd]`，已提交的 generation 路径永远不会被重命名、替换或删除。JSONL provider 拥有物理分帧、压缩、generation 选择和独占发布，而每个相邻迁移包恰好拥有一个 `vN -> vN+1` 步骤（[决策](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-08-31-released-session-format-migrations.md)）。

**对模型可见即已入日志。** 每次模型请求都必须能从日志重建。新的模型可见输入需要会话事件。修改既有消息内容的插件要注册[纯消息投影](https://deepseek-harness.github.io/deepseek-harness/reference/subsystems/session)；分离的读取方显式提供同样的定义。工具变更与能力无关；[Session 工具历史](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/session/README.md)提供 provider 声明。

**投影 seam。** `dsh-session-projection` 拥有 `ctx.sessionProjections`：注册的单元对已提交事件做增量折叠，宿主消费方用 `stateOf()` 读取一个带类型的状态，载体用 `snapshot()` 批量裁剪客户端视图。宿主读取方要么在激活期间要求该服务，要么在注册表或所需键缺失时显式失败。贡献方可以保留 `ctx.inject(['sessionProjections'], ...)` 注册，而不是静默默认缺失的宿主值。agent loop 为其读取方注册共享的 `turnBoundary` 状态（[决策](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-08-19-session-projection-mandatory-seam.md)）。

[工作目录](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/working-directory.md)提供用户上下文与执行路径，而不改变原始项目身份、沙箱写根或既有进程目录。

## 能力 seam

**seam** 是一个可替换的能力，具有三种角色：声明接口的 **Service Definition**、实现它的 **Service Provider**，以及使用它的 **Consumer**（常见为面向模型的工具）。一个包可以合并多个角色，但单一角色不构成 seam；新增能力意味着设计全部三者（[能力图谱](https://deepseek-harness.github.io/deepseek-harness/reference/capability-seams)）。

seam 是"换一个提供方就改变整个产品"的原因。文件系统与子进程提供方共享同一个执行世界，把它们指向远程沙箱时，Bash、PTY 和 LSP 会随之迁移，无需分叉提供方。[Subagent 提供方](https://deepseek-harness.github.io/deepseek-harness/reference/subsystems/subagent)在同一个接口背后的差异同样巨大，从全新的子 agent 到另一个产品中的受托 turn 都有。

[实验性的 Agent Teams](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/agent-team.md) 通过可续接的 subagent 暴露可选的 `ctx.agentTeams` 协调，具备持久的成员与任务状态以及直接的 inbox 消息传递。

## 新行为放在哪里

新行为挂到有文档记载的扩展点上。改动 loop 本身时更新本表。

| 目标 | 机制 |
| --- | --- |
| 添加模型提供方 | 在 `ctx.llm` 上注册其适配器 |
| 添加面向模型的能力 | 注册到 `ctx.tools`；其 schema 加入提示词组装 |
| 给单个会话不同的能力集合 | 组合一个 agent preset；其中的服务行需要 `isolate` realm |
| 添加 shell 执行 | 注册一个 `ctx.shell` 后端；本地实现经 `ctx.subprocess` 派生进程 |
| 添加持久终端执行 | 注册 `ctx.terminals` 后端并加上 `dsh-tool-terminal` |
| 添加人工命令 | 注册到 `ctx.commands`；分发不经过模型 turn |
| 管理后台任务 | 注册到 `ctx.jobs`； `job_*` 工具读取或停止任务 |
| 从外部 webhook 启动 Session | 在 `ctx.webhookRuntime` 注册受信任规则并挂载 provider 适配器 |
| 添加文件系统访问或策略 | 注册 `ctx.fs` provider 或监听 `fs/*` 事件 |
| 限制派生进程 | 使用 `ctx.sandbox` 后端；消费方在 spawn 前包装 argv |
| 拦截请求、工具或 turn | 使用对应的 `agent/*` 或 `tools/*` 事件； `agent/turn-stopping` 停止一个 turn |
| 添加面向模型的上下文 | 调用 `agent.inject()`；它落入下一个被准入的请求 |
| 添加 UI 或编辑器集成 | 驱动 `ctx.agents` 并从 `session/event` 渲染 |
| 添加 Web Client Chat 节点 | 注册 `ConversationNodeDefinition` + 带 key 的渲染器 |
| 添加持久会话状态 | 扩展 `SessionEventMap`；从日志渲染和回放 |
| 生成会话标题 | 注册唯一的 `ctx.sessionTitle` provider |
| 管理同会话目标 | 使用 `ctx.goals`；经 `agent/*` 继续 |
| 在 turn 边界 fork 会话 | `ctx.agents.create({ sessionId, seed, meta: { parentSession, seedLength } })` ——只有 agent loop 发布的会话会持久化 |
| 把会话存储到新后端 | 在共享的 handle 脚手架之上实现 `SessionPersistence`（ `create` / `open` / `stat` / `list` / `export` ） |
| 把注册限定到单个 agent | 使用该 agent 的 `agent.ctx` |

[扩展手册](https://deepseek-harness.github.io/deepseek-harness/reference/cookbook/extension-cookbook)把特性映射到能力，并索引 [包](https://deepseek-harness.github.io/deepseek-harness/reference/cookbook/adding-a-package)、[工具](https://deepseek-harness.github.io/deepseek-harness/reference/cookbook/adding-a-tool)、[LLM 适配器](https://deepseek-harness.github.io/deepseek-harness/reference/cookbook/adding-an-llm-adapter)和 [settings 页面](https://deepseek-harness.github.io/deepseek-harness/reference/cookbook/adding-a-settings-card)的分步指南。[Conversation 子系统](https://deepseek-harness.github.io/deepseek-harness/reference/subsystems/conversation)负责 Chat 节点组装。
