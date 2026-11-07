# 从提示词配置持久插件

Creator 模式提供 [Plugin Manager](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/boot/plugin-manager/README.md) 和只读的[运行时巡检](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/extensions/tool-cordis/README.md)。插件配置归属于当前 profile，作用于该 profile 的会话，并在进程重启后依然生效。

## 接入一个 MCP 服务器

启动 Web profile 并选择 Creator 模式。在一个可访问的、暴露 `ping` 工具的 Streamable HTTP MCP 服务器处，用其实际 endpoint 发送这条提示：

> Configure the MCP server at `<endpoint>` in this profile as `demo`. Make its tools available now, then call its ping tool and tell me the result.

agent 会编写一个仅含配置的组合包（bundle），其 patch 插入 `@deepseek-ai/dsh-mcp-client`，然后用 `plugin_manager install_bundle` 安装它。启用 HMR 时，工具会出现在同一个运行中的会话里。验证两件事：管理结果（ `application: applied` ）和一次成功的 `mcp__demo__ping` 调用。状态为 `restart-required` 的已保存条目尚未激活；失败的条目需要修复配置。

编辑配置前先读组合包的 patch。用 Plugin Manager 停用条目或移除组合包。MCP 客户端接受的配置与连接失败行为见 [MCP client reference](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/mcp/mcp-client/README.md)。
