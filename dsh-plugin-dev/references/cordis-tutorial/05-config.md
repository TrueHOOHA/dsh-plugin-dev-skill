## 5\. 配置

`cordis.yml` 中的每个 Cordis 配置项都可以携带 `config` 块，插件则声明一个 schema，在运行 `apply` 前验证该块。错误配置会导致加载失败，并给出准确的错误：插件绝不会在配置不完整时启动。

## 可配置插件

创建 `config-demo.ts` ，并将其放在 `tmp/cordis-tutorial` 中：

```ts
import type { Context } from '@deepseek-ai/cordis'
import Schema from '@deepseek-ai/schemastery'

export const name = 'config-demo'

export interface Config {
  greeting: string
  targets: string[]
}

export const Config: Schema<Config> = Schema.object({
  greeting: Schema.string().default('Hello'),
  targets: Schema.array(String).default(['world']),
})

export function apply(ctx: Context, config: Config) {
  for (const target of config.targets) {
    console.log(\`${config.greeting}, ${target}!\`)
  }
}
```

导出的 `Config` 既是 TypeScript 接口，也是同名的运行时 schema：消费方获得类型，Cordis 获得验证器。本仓库使用 [Schemastery](https://github.com/shigma/schemastery) 定义 schema；Cordis 本身接受任意 [Standard Schema](https://standardschema.dev/) 验证器，因此将普通对象导出为 `Config` 无法工作。

对其进行配置：

```yaml
- name: './config-demo.ts'
  config:
    targets: ['alpha', 'beta']
```

运行：

```
Hello, alpha!
Hello, beta!
```

未提供 `greeting` ，因此 schema 默认值会将其补齐： `apply` 始终会收到完整且经过验证的配置。

## 明确报错

现在向它传入无效内容：

```yaml
- name: './config-demo.ts'
  config:
    targets: 'not-an-array'
```
```
ValidationError: invalid config:
  - $.targets expected array but got not-an-array (at targets)
```

插件的 fiber 进入 FAILED 状态，本教程的启动器打印错误后以状态码 1 退出。如果某个插件的配置通过了 schema 验证，但其中指定的资源或提供方不可用，该插件也应当在能解析该引用时立即拒绝。

## 易变字段（volatile）

对于插件在每次操作中都会读取的字段，使用 `.volatile()` 标记：修改该字段只会更新其稳定引用，而不会重新挂载插件。用 `.get()` 读取：

```ts
import type { Context } from '@deepseek-ai/cordis'
import type {} from '@deepseek-ai/cordis-plugin-loader'
import Schema from '@deepseek-ai/schemastery'

export const Config = Schema.object({
  greeting: Schema.string().default('Hello').volatile(),
})

export function apply(ctx: Context, config: ReturnType<typeof Config>) {
  ctx.on('loader/volatile-update', () => {
    console.log(config.greeting.get())
  })
}
```

直接调用 `Config(raw)` 同样返回引用。没有默认值的可选字段也有引用；字段缺席时 `.get()` 返回 `undefined`。可以保留引用，也可以把某个值捕获下来供单次操作使用（包括跨 `await` ），但不要把它留存给后续操作。对象和数组值是普通数据的递归冻结快照（分离副本）；函数、类实例和循环引用会被拒绝。

Loader 比较原始配置时会忽略 schema 声明的 volatile 字段。仅 volatile 值变化时，会经由插件声明的 `internal/config` 钩子解析并校验；当所有普通生效值仍相同时，Loader 把新值提交进运行中的引用，并发出一次实例本地的 `loader/volatile-update` 事件（以键路径数组携带变化字段），绕过目标插件的 `internal/update`；值相等则不通知。校验失败的候选会被记录，运行中的引用保持不变，等待下一次激活。普通生效值（包括表达式求值结果）变化或与普通字段同时变化时，仍走既有的 update 流程，旧引用保持不变且不会收到通知。Group/Include 仍会向子项分发更新。直接 `fiber.update()`、代码替换和依赖替换保留各自的生命周期；`noSave` 仍然控制直接更新的持久化。通知使用普通 `emit`，不会等待资源重配置。

volatile 字段必须声明在固定的对象路径上，或者把整个对象或数组标记为 volatile。数组内部、字典、union/intersect 分支、lazy schema、transform 之中或另一个 volatile 值之内的独立引用会被拒绝。配置文件里存储的是普通值； `simplify()` 可以解包引用。序列化的 schema 保留字段类型、默认值、角色和 volatile 标记，供 schema 驱动的表单使用；既有的 settings 消费方继续使用各自的 settings 接口。

## 计算得到的配置值

本仓库使用的 loader 支持 `!!js` 标签，用于必须在加载时计算的配置值：

```yaml
- name: './config-demo.ts'
  config:
    greeting: !!js process.env.DEMO_GREETING ?? 'Hello'
```

`!!js` 仅在 `config` 与条目 `disabled` 字段内有效。 `disabled: !!js ...` 在每次挂载决策时基于 loader 上下文求值（本仓库的扩展），可以按平台或环境门控一行；其余元数据（ `name` 、 `id` 、 `inject` 等）保持静态，其中的表达式是普通真值数据。详见 [loader 配置](https://deepseek-harness.github.io/deepseek-harness/reference/cordis-primer#loader-configuration) 。

下一章： [组合与 HMR（热模块替换）](https://deepseek-harness.github.io/deepseek-harness/develop/cordis-tutorial/06-composition-and-hmr) ：将 `cordis.yml` 视为应用。