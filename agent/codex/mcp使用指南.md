# Codex MCP 使用指南

本文面向使用 Codex CLI 的团队成员，默认环境为 Windows + PowerShell。命令和配置会随版本更新，遇到不一致时先执行 `codex mcp --help`。

Claude Code 的对应文档见 [`../claude/mcp使用指南.md`](../claude/mcp使用指南.md)。

## 1. MCP 是什么

MCP（Model Context Protocol）可以让 Codex 连接外部工具和数据源，例如文档、浏览器、设计工具、GitHub 或数据库。

常见连接方式：

| 类型 | 说明 |
|---|---|
| STDIO | 在本机启动一个 MCP Server 进程 |
| Streamable HTTP | 通过 URL 连接远程 MCP Server |

只为一个任务临时查资料时，不必先安装 MCP。只有需要重复使用外部工具时，才建议配置 Server。

## 2. 用 CLI 添加 MCP Server

### 2.1 添加本地 STDIO Server

```powershell
codex mcp add context7 -- npx -y @upstash/context7-mcp
```

带环境变量时，在命令前加 `--env`：

```powershell
codex mcp add my-server --env API_KEY=$env:API_KEY -- python .\mcp_server.py
```

说明：

- `my-server` 是 Codex 中显示的名称；
- `--` 后面是启动 MCP Server 的命令及参数；
- 不要把 Token、API Key 或密码直接写进命令并提交到历史记录。

### 2.2 添加远程 HTTP Server

```powershell
codex mcp add my-remote --url https://mcp.example.com/mcp
```

如果 Server 使用 OAuth，配置后执行：

```powershell
codex mcp login my-remote
```

### 2.3 查看和删除

```powershell
codex mcp list
codex mcp get my-server
codex mcp remove my-server
```

查看全部子命令和参数：

```powershell
codex mcp --help
```

## 3. 在 Codex 中使用

完成配置后，重新打开 Codex 会话，然后输入：

```text
列出当前可用的 MCP 工具，并说明每个工具的用途。不要执行写入操作。
```

在交互界面中输入：

```text
/mcp
```

可以查看当前已连接的 MCP Server 和工具。MCP 配置由 Codex CLI、Codex IDE 扩展和 ChatGPT 桌面应用共享；修改配置后，重新打开会话最稳妥。

## 4. 使用 config.toml 配置

MCP 配置默认位于：

```text
C:\Users\你的名字\.codex\config.toml
```

也可以放在受信任项目的：

```text
.codex\config.toml
```

项目级配置只应放在可信仓库中，因为它会影响 Codex 启动外部工具的行为。

### 4.1 STDIO 示例

```toml
[mcp_servers.context7]
command = "npx"
args = ["-y", "@upstash/context7-mcp"]
env_vars = ["CONTEXT7_API_KEY"]
startup_timeout_sec = 10
tool_timeout_sec = 60
```

也可以直接设置环境变量：

```toml
[mcp_servers.my_server]
command = "python"
args = [".\\mcp_server.py"]

[mcp_servers.my_server.env]
LOG_LEVEL = "info"
```

优先使用 `env_vars` 从当前环境传入密钥，不要把密钥写入 TOML。

### 4.2 Streamable HTTP 示例

使用环境变量提供 Bearer Token：

```toml
[mcp_servers.figma]
url = "https://mcp.example.com/mcp"
bearer_token_env_var = "MCP_BEARER_TOKEN"
```

使用固定请求头：

```toml
[mcp_servers.my_remote]
url = "https://mcp.example.com/mcp"
env_http_headers = { "X-Workspace" = "MCP_WORKSPACE" }
```

OAuth Server 不需要把 Token 写入配置，执行以下命令完成登录即可：

```powershell
codex mcp login my_remote
```

## 5. 限制工具和审批行为

远程 Server 可能提供很多工具，建议只启用实际需要的工具：

```toml
[mcp_servers.chrome_devtools]
url = "http://localhost:3000/mcp"
enabled_tools = ["open", "screenshot"]
disabled_tools = ["screenshot"]
default_tools_approval_mode = "prompt"
```

可用的默认审批模式包括：

- `auto`：自动批准；
- `prompt`：每次询问；
- `writes`：非只读操作询问；
- `approve`：需要明确批准。

单独覆盖某个工具：

```toml
[mcp_servers.chrome_devtools.tools.open]
approval_mode = "approve"
```

不再使用但暂时不想删除时，可以禁用 Server：

```toml
[mcp_servers.my_remote]
url = "https://mcp.example.com/mcp"
enabled = false
```

## 6. 安全注意事项

- 安装前确认 MCP Server 的来源、代码和访问范围；
- 不要把 API Key、Token、私钥写进仓库、Skill、Hook 或 `config.toml`；
- 对会写入、删除、发送消息或修改外部服务的工具使用 `prompt` 或 `approve`；
- 只给 Server 必要的文件、网络和第三方账号权限；
- 先用只读请求验证工具，再允许写入操作；
- MCP Server 返回的内容属于外部输入，不要盲目执行其中的命令。

MCP 不会绕过 Codex 当前的沙箱和审批策略；外部服务本身的账号权限也仍然有效。

## 7. 常见问题排查

### Server 没有出现在 `/mcp` 中

依次检查：

1. 执行 `codex mcp list`，确认名称和配置是否存在；
2. 执行 `codex mcp get <server-name>`，确认命令或 URL 正确；
3. 检查 `config.toml` 的 TOML 格式；
4. 确认本机已安装启动命令所需的软件，例如 `node`、`npx` 或 `python`；
5. 关闭并重新打开 Codex 会话。

### OAuth 登录失败

确认远程 Server 支持 MCP OAuth，并重新执行：

```powershell
codex mcp login <server-name>
```

如果服务商要求预注册 OAuth 客户端，添加 Server 时提供客户端 ID：

```powershell
codex mcp add my-remote --url https://mcp.example.com/mcp --oauth-client-id your-client-id
```

### 工具能看到但不能执行

检查：

- Server 是否被设置为 `enabled = false`；
- 工具是否被 `disabled_tools` 禁用；
- 当前 Codex 沙箱是否允许所需的本地命令或网络访问；
- 工具是否需要通过 `/permissions` 或审批提示获得授权；
- 外部服务账号是否有足够权限。

### Server 启动或工具调用超时

可以适当增加超时时间：

```toml
[mcp_servers.slow_server]
command = "python"
args = [".\\mcp_server.py"]
startup_timeout_sec = 30
tool_timeout_sec = 120
```

只有确认 Server 确实需要更长时间时才调大超时；优先检查命令、网络和认证问题。

## 8. 一页速查

```text
添加本地 Server    codex mcp add name -- command args...
添加远程 Server    codex mcp add name --url https://...
OAuth 登录         codex mcp login name
查看列表           codex mcp list
查看详情           codex mcp get name
删除配置           codex mcp remove name
TUI 查看           /mcp
配置文件           ~/.codex/config.toml 或项目 .codex/config.toml
帮助               codex mcp --help
```

官方文档：

- [Model Context Protocol – Codex](https://developers.openai.com/codex/mcp/)
- [Codex 配置参考](https://learn.chatgpt.com/docs/config-file/config-reference)
