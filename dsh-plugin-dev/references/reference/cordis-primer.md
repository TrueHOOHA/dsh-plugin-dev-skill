# Cordis 入门

Cordis 是 DeepSeek Harness 底层内嵌（vendored）的插件框架。本入门讲解 harness 插件作者在阅读[子系统页面](https://deepseek-harness.github.io/deepseek-harness/reference/subsystems/core)上生成的服务/事件参考之前需要掌握的 Cordis 概念；[Cordis 教程](https://deepseek-harness.github.io/deepseek-harness/develop/cordis-tutorial/)则以动手方式讲解同样的内容。内嵌源码与同步流程见 [vendor/README.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/vendor/README.md)。

## Cordis 五个核心观念

- **插件是实现 Service 的对象。** 它可以是带有可选 `inject` 和 `apply(ctx)` 字段的函数，也可以是其生命周期由 Cordis 挂载进当前上下文的 `Service` 子类。
- **上下文是服务的仓库。** 服务从上下文申领一个稳定的 `ctx.<key>`（如 `ctx.tools` 、 `ctx.llm` 、 `ctx.sessions` ）；其他插件按键查找服务，而不是导入具体实现。
- **用 `inject` 声明服务依赖。** 列出所需服务的插件会等待这些服务存在，因此加载顺序由服务需求表达，而不是手工启动编排。
- **类型化事件负责通信。** 服务通过 TypeScript 声明合并声明事件名，再按监听器是观察、包装、并发、顺序执行还是首个 bail 值即停，分别以 `emit` 、 `waterfall` 、 `parallel` 、 `serial` 或 `bail` 分发。
- **注册都是可逆的 effect。** 提示词区块、工具 schema、适配器、提供方和监听器都通过 `ctx.effect()` 或 `ctx.on()` 安装，因此重载与卸载会以可预测的方式回卷。

## 分发模式

每个事件都有且仅有以下一种分发模式，也只能用对应的方法分发：

| 模式 | 是否等待 | 分发顺序 | 有返回值？ |
| --- | --- | --- | --- |
| `emit` | 否 | 监听器按注册顺序观察 | 否 |
| `waterfall` | 否 | 监听器按注册顺序观察 | 是 |
| `parallel` | 是 | 所有监听器并发观察 | 否 |
| `serial` | 是 | 监听器按注册顺序观察 | 是 |
| `bail` | 否 | 监听器按注册顺序观察，直到某个监听器 bail | 是 |

分发模式是事件公共契约的一部分。新的 harness 事件用 `@mode` 标签记录其模式，生成的目录会据此检查声明与分发点是否一致。

## Cordis waterfall 语义

`ctx.waterfall` 是环绕（around）中间件。监听器收到 `(...args, next)`；调用 `next()` 把（可能被包装过的）结果委托给下一个服务；不调用 `next()` 直接返回即短路。值通过 `next()` 的返回值传播。

协作型监听器通常修改一个共享的请求或决策对象，然后委托。监听器也可以选择完全替换结果，下游监听器只会看到替换后的结果。仅当监听器必须先于普通注册执行时才使用 `prepend: true`。

对单决策事件而言，短路就是设计本身：策略监听器在它拥有决策权时可以不调用 `next()` 直接返回，而只做标注或观察的监听器必须委托。

## Loader 配置

`@deepseek-ai/cordis-plugin-include` 把 `!!js` 解析成表达式节点。Loader 会在条目声明的注入激活之后、针对该插件上下文（ `ctx.serviceName` ）插值其 `config`；在每个挂载决策时、针对 loader 上下文插值其 `disabled` 字段；Include 会保留嵌套行中的表达式，直到目标激活。其余条目元数据保持字面量。当由环境决定启用哪些插件时，使用 overlay。

## 实践规则

把行为封装进插件：工具流水线事件属于 `ctx.tools`，模型流式属于 `ctx.llm`，实时 agent 协调属于 `ctx.agents`。拦截与策略优先用事件；直接能力调用优先用服务方法。

每个注册都应该有 disposer——要么从 `ctx.effect()` 返回一个，要么使用自带 disposer 的 Cordis 辅助方法。如果拆除顺序很重要，就把相关工作放进同一个 effect，让处置按预期顺序回卷。
