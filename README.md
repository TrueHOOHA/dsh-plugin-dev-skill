# dsh-plugin-dev-skill

一个用于辅助开发 [DeepSeek Harness](https://deepseek-harness.github.io/deepseek-harness/)（dsh）插件的 AI Skill。

DeepSeek Harness 是基于 [Cordis](https://cordisjs.com/) 的 agent 框架，插件是导出 `apply(ctx: Context)` 的 TypeScript 模块，通过 `cordis.yml` 组合装配。本 skill 为 AI 助手提供创建、修改、调试、打包 dsh 插件所需的知识，覆盖工具（tool）、服务（service）、LLM 适配器（adapter）、事件监听（event listener）等能力。

## 使用方式

将 `dsh-plugin-dev/` 目录放入你的 skill 目录（如 `.claude/skills/`、`.config/opencode/skills/`），并开启插件开发模式。当用户提到以下内容时，本 skill 会被自动触发：

- 创建 / 修改 / 调试 / 打包 dsh 插件
- 编写 tool、service、LLM adapter、event listener
- 配置 `cordis.yml` 补丁
- 打包插件用于分发
- 提到 Cordis 生命周期、Fiber 状态、`inject`、`ctx.effect`、`defineTool`、`Service` 子类、`LlmAdapter`、Schemastery 配置、`dsh.bundle` 等

> 仅限 dsh 相关开发，不用于其他 agent / LLM 集成场景。

## 目录结构

```
dsh-plugin-dev/
├── SKILL.md            # Skill 主文件：核心概念速查 + 参考文档索引
├── agents/
│   └── openai.yaml     # Agent 元数据（显示名、默认提示词）
└── references/         # 离线文档库
    ├── basic/          # 入门教程：第一个插件、工具、配置、发布
    │   ├── index.md    # 第一个插件：编写 apply(ctx) 并通过 --patch 加载
    │   ├── tool.md     # 注册 tool：defineTool + ctx.tools.register
    │   ├── config.md   # 插件配置：Config interface + Schemastery schema
    │   └── publish.md  # 发布：dsh.bundle 清单、profile、dsh plugin add
    ├── cordis-tutorial/  # Cordis 框架分步教程（7 章，无需 API Key 即可运行）
    │   ├── 01-first-plugin.md        # 最小插件 hello.ts
    │   ├── 02-lifecycle-and-effects.md # 生命周期与 ctx.effect 清理
    │   ├── 03-services.md            # 服务：提供与注入（inject）
    │   ├── 04-events.md              # 事件：ctx.on / ctx.emit
    │   ├── 05-config.md              # 配置校验与加载失败
    │   ├── 06-composition-and-hmr.md # 插件树、group、isolate、HMR
    │   ├── 07-into-the-harness.md    # 注册模型可调用的工具
    │   └── index.md                  # 教程总览与环境准备
    ├── framework/      # 框架概念速查
    │   ├── index.md    # 插件模型与 Fiber 状态机
    │   ├── events.md   # 事件机制与 Harness 事件
    │   └── service.md  # 服务注入、提供与隔离
    └── practice/       # 进阶设计模式与集成
        ├── index.md    # 三层能力设计：定义 → 提供 → 消费
        └── llm-adapter.md  # 接入新模型提供商：LlmAdapter.stream()
```

## 内容概览

### SKILL.md 核心速查

| 主题 | 要点 |
| --- | --- |
| 插件结构 | 三种形式：函数 / 对象 `{ name, inject, apply }` / 类（继承 `Service`） |
| 依赖注入 | `export const inject = ['tools', 'llm']`，依赖未就绪时插件处于 `PENDING` |
| 自动清理 | 所有经 `ctx` 注册的内容都是 effect，卸载自动回收；自定义资源用 `ctx.effect()` 包裹 |
| Fiber 状态 | `PENDING → LOADING → ACTIVE → UNLOADING → DISPOSED`（可 `FAILED`） |
| 工具 | `defineTool` 自动生成 JSON Schema、校验参数、卸载时注销 |
| 配置 | 需同时导出 `interface Config` 与 `const Config: Schema<Config>` |
| 服务 | 提供方继承 `Service`，消费方声明 `inject`，`group: true` + `isolate` 实现隔离 |
| 事件 | `emit` / `parallel` / `serial` / `bail` / `waterfall` 五种模式 |
| LLM 适配器 | 继承 `LlmAdapter` 实现 `stream()`，按块顺序产出 StreamChunk |
| 发布 | 本地用 `--patch`，分发用 npm 包 + `dsh.bundle`，`dsh plugin add` 安装 |
| HMR | 加载 `cordis-plugin-hmr` + logger + timer 实现保存即重载 |

### 核心代码示例

```ts
// 最小插件
import type { Context } from '@deepseek-ai/cordis'
export const name = 'my-plugin'
export function apply(ctx: Context) { /* register capabilities */ }

// 注册一个工具
import { defineTool } from '@deepseek-ai/dsh-tools'
export const inject = ['tools']
export function apply(ctx: Context) {
  ctx.tools.register(defineTool({
    name: 'greet', description: '...',
    parameters: { name: { type: 'string', required: true } },
    output: { schema: { type: 'string' }, render: (_args, value) => [{ type: 'text', text: value }] },
    async execute(args) { return `Hello, ${args.name}!` },
  }))
}
```

## 许可证

[Apache License 2.0](./LICENSE)
