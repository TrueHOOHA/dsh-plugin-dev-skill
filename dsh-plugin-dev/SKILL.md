---
name: dsh-plugin-dev
description: |
  Develop plugins for DeepSeek Harness (dsh), a Cordis-based agent framework.
  Use when the user asks to create, modify, debug, or package a dsh plugin;
  write a tool, service, LLM adapter, or event listener; configure cordis.yml
  patches; or bundle a plugin for distribution. Also triggers on mentions of
  Cordis lifecycle, Fiber states, inject, ctx.effect, defineTool, Service
  subclass, LlmAdapter, Schemastery config, cordis.yml, dsh plugin, or
  dsh.bundle. Do NOT use for general agent/LLM integration outside dsh.
---

# dsh Plugin Development

Develop plugins for [DeepSeek Harness](https://deepseek-harness.github.io/deepseek-harness/)
(dsh), a Cordis-based agent framework. Plugins are TypeScript modules exporting
`apply(ctx: Context)`, composed via `cordis.yml`.

## Quick Reference

### Plugin Structure

```ts
import type { Context } from '@deepseek-ai/cordis'
export const name = 'my-plugin'
export function apply(ctx: Context) { /* register capabilities */ }
```

**Three forms**: function (default), object (`{ name, inject, apply }`), class
(`extends Service` — use only when providing a service to other plugins).

**Dependencies**: `export const inject = ['tools', 'llm']` — framework keeps
plugin PENDING until all injected services are ready.

**Auto-cleanup**: Everything registered through `ctx` is an effect — unload
automatically cleans up. For custom resources (timers, connections), wrap in
`ctx.effect(() => { ... return () => cleanup })`.

**Child plugins**: `const fiber = ctx.plugin(childPlugin)` — children inherit
parent context; `await fiber.dispose()` unloads recursively.

### Fiber States

```
PENDING → LOADING → ACTIVE → UNLOADING → DISPOSED
                 ↘ FAILED
```

`PENDING` → missing service dependency. `FAILED` → `apply` threw or config
validation failed. Service disappearing at runtime auto-disposes dependents.

### Tools

```ts
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

`defineTool` converts parameters to JSON Schema, validates args, and auto-unregisters on unload.
See `references/basic/tool.md` for details.

### Configuration

```ts
import Schema from '@deepseek-ai/schemastery'
export interface Config { greeting: string; maxRetries: number }
export const Config: Schema<Config> = Schema.object({
  greeting: Schema.string().default('Hello'),
  maxRetries: Schema.number().default(3),
})
export function apply(ctx: Context, config: Config) { /* validated config */ }
```

Must export BOTH `interface Config` AND `const Config: Schema<Config>`.
Invalid config → FAILED state with clear error. Use `!!js` in YAML for dynamic
values: `config: { apiKey: !!js process.env.API_KEY }`.
See `references/basic/config.md`.

### Services

**Provide**: `class extends Service { constructor(ctx) { super(ctx, 'name') } }`
+ `declare module '@deepseek-ai/cordis' { interface Context { name: NameService } }`

**Consume**: `export const inject = ['name']` → `ctx.name.method()`.

**Optional**: `ctx.get('name')` → returns `undefined` when absent.

**Service isolation**: `group: true` + `isolate: { shell: true }` gives each group
its own provider instance. See `references/framework/service.md`.

### Events

| Mode | Call | Use |
| --- | --- | --- |
| `emit` | `ctx.emit(n, ...args)` | Sync broadcast, ignore returns |
| `parallel` | `await ctx.parallel(n, ...args)` | Concurrent listeners |
| `serial` | `await ctx.serial(n, ...args)` | Sequential, first non-null wins |
| `bail` | `ctx.bail(n, ...args)` | Sync serial |
| `waterfall` | `ctx.waterfall(n, ...args, next)` | Middleware chain |

Waterfall listeners **must** call `next()` unless intentionally short-circuiting.
Key harness events: `agent/step`, `tools/result`, `session/event`.
See `references/framework/events.md`.

### LLM Adapters

```ts
class MyAdapter extends LlmAdapter {
  async *stream(options: GenerateOptions): AsyncIterable<StreamChunk> {
    // Convert → provider API → yield StreamChunk
  }
}
ctx.llm.registerAdapter(['my-provider'], adapter)
```

StreamChunk order per block: `block-start` → `text-delta`/`tool-call-delta` →
`block-end` → `usage` → `finish`. Throw `LlmError` with stable codes.
See `references/practice/llm-adapter.md`.

### Three-Layer Capability Design

```
Definition (Service abstract class) → Provider (implements) → Consumer (tool)
```

All three share the same service name via `inject`. Only split into separate
packages when roles need independent evolution.
See `references/practice/index.md`.

### cordis.yml / Bundles

**Local dev**: `--patch ./cordis.patch.yml`

```yaml
- insert:
    - id: my-plugin
      name: '/absolute/path/to/src/my-plugin.ts'
      config: { greeting: 'Hi' }
```

**Distribution**: npm package with `dsh.bundle` manifest + `cordis.patch.yml`.
Install: `dsh plugin --profile demo add ./hello-plugin`.
Layer order: profile bundles → profile patch → home patch → `--patch` overlays.
Later layers win by row (no deep merge).
See `references/basic/publish.md`.

### HMR

Load `@deepseek-ai/cordis-plugin-hmr` + `@deepseek-ai/cordis-plugin-logger-console`
+ `@deepseek-ai/cordis-plugin-timer` to auto-reload on file save.

### Diagnosing PENDING

```ts
import { FiberState, type Context } from '@deepseek-ai/cordis'
for (const fiber of ctx.registry.values().flatMap(r => r.fibers))
  if (fiber.state === FiberState.PENDING)
    console.log(`${fiber.name} is PENDING`)
```

## Reference Docs

Detailed guides in the `references/` directory:

| Path | Covers |
| --- | --- |
| `references/basic/` | First plugin, tools, config, publishing |
| `references/framework/` | Lifecycle, services, events |
| `references/practice/` | Three-layer design, LLM adapters |
| `references/cordis-tutorial/` | Step-by-step Cordis framework tutorial |