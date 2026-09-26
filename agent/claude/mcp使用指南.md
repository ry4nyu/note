# Claude Code MCP 使用指南

本文面向本仓库团队成员，说明如何在 Claude Code 里接入、使用和排查 MCP Server。文中命令和默认值基于 Claude Code 2.1.x（Windows + PowerShell 环境），跟随版本变化，遇到不一致先跑 `claude mcp --help` 和 `/mcp`。

前置阅读：[`README.md`](./README.md)（安装、配置、权限模式）；Skill 与 MCP 的分工见 [`skill使用指南.md`](./skill使用指南.md) 第 2 节，需要独立上下文或并行时见 [`subagent使用指南.md`](./subagent使用指南.md)。Codex / Pi 的对应文档见 [`../codex/mcp使用指南.md`](../codex/mcp使用指南.md)。

> **关于本文的事实来源**：命令、作用域、默认值均在本机 Claude Code 2.1.x 上实测（`claude mcp --help`、`claude mcp get`、本机配置文件）。少数标注"官方文档口径"的内容来自官方文档描述，**没有本地复现**，按参考值看待。

---

## 1. MCP 是什么

MCP（Model Context Protocol）让 Claude Code 连接**外部工具和数据源**：文档检索、浏览器、设计稿、GitHub、内部 API、数据库等等。它是给模型**加工具**，不是教模型怎么干活。

常见的连接方式：

| 类型 | 说明 | 典型场景 |
|---|---|---|
| STDIO | 在本机起一个 Server 进程，用标准输入输出通信 | 本地工具、`npx` 起的包 |
| HTTP | 通过 URL 连远程 Server（含 `streamable-http`） | 官方托管的 SaaS 类 Server |
| SSE | 早期的远程传输方式，现在少见 | 老 Server 仍在用 |
| WebSocket | `ws://` 长连接 | 少部分自建 Server |

配置里 `type` 的合法取值实测为：`stdio`、`sse`、`http`、`streamable-http`、`ws`、`sdk`、`claudeai-proxy`（后两个是内部用途，手写配置用不到）。

**先想清楚要不要装。** Claude Code 里能扩展能力的地方不止一处：

| 你想要的效果 | 该用哪个 |
|---|---|
| 一类任务的流程、检查单、边界 | Skill（见 [`skill使用指南.md`](./skill使用指南.md)） |
| 每次都必须发生、不依赖模型判断 | Hook |
| 恒真的项目事实和约定 | `CLAUDE.md` |
| **接入新工具或数据源** | **MCP Server（本文）** |

只为一个任务临时查资料，不必先装 Server —— 用 WebSearch / WebFetch 或直接读文件通常更快。**当同一个外部工具你会反复用到时，才值得配 Server。**

---

## 2. 三个作用域

加 Server 时第一个要决定的是**装在哪个作用域**。实测（`claude mcp get <name>` 的输出）：

| 作用域 | 存哪 | 谁能用 | `claude mcp get` 显示 |
|---|---|---|---|
| `local`（**默认**） | `~/.claude.json` → `projects.<项目路径>.mcpServers` | 只有你，只在这个项目 | `Local config (private to you in this project)` |
| `project` | 项目根目录的 `.mcp.json` | 跟着仓库走，**能提交给全团队** | `Project config (shared via .mcp.json)` |
| `user` | `~/.claude.json` → 顶层 `mcpServers` | 只有你，**所有项目** | `User config (available in all your projects)` |

几条实测结论：

- **不写 `-s` 就是 `local`**，落在 `~/.claude.json` 的项目节点下，不进仓库、不影响同事。想先试试水就用默认值。
- **`~/.claude/settings.json` 不放 `mcpServers`** —— 那是配模型、权限、shell 的地方（见 [`README.md`](./README.md) 第 5 节）。本机实测 `settings.json` 里没有这个键，别往那儿写。
- 三个作用域**同时生效**，不是三选一。同名时优先级 **`local` > `project` > `user`**，且是**整体覆盖、不合并**（官方文档口径）—— 高优先级那份怎么写就怎么用，字段不会和低优先级拼起来。
- 想确认某个 Server 到底装在哪：`claude mcp get <name>` 第一行就是作用域。

---

## 3. 用 CLI 添加 Server

### 3.1 stdio（本地进程，最常见）

```powershell
claude mcp add context7 -- npx -y @upstash/context7-mcp
```

带环境变量时用 `-e` / `--env`：

```powershell
claude mcp add my-server -e API_KEY=xxx -- npx my-mcp-server
```

装到指定作用域：

```powershell
claude mcp add context7 -s user -- npx -y @upstash/context7-mcp
```

说明：

- `context7` 是你在 Claude Code 里看到的名字，之后 `remove` / `login` 都用它；
- **`--` 后面才是启动 Server 的命令和参数**，`--` 的作用是让 Claude Code 停止解析自己的选项，把后面的 `--some-flag` 原样交给 Server；
- `-e` 设的环境变量会注入 Server 进程，等于把密钥交给它，只给你信任的 Server。

> ⚠️ **不要把 Token / API Key / 密码直接写进命令行**：会留在 shell 历史里。要么用 `-e` 从环境变量取（`-e API_KEY=$env:API_KEY`），要么走第 4 节的配置文件。

### 3.2 HTTP / SSE（远程 Server）

```powershell
claude mcp add --transport http sentry https://mcp.sentry.dev/mcp
```

需要固定请求头时用 `-H` / `--header`（可以给多个）：

```powershell
claude mcp add --transport http my-remote https://mcp.example.com/mcp `
  -H "X-Api-Key: abc123" -H "X-Custom: value"
```

`--transport` 实测支持 `stdio`（默认）、`sse`、`http`；不指定时按 stdio 处理，所以**加 URL 一定要显式写 `--transport http`**，否则会被当成命令名去起进程。

### 3.3 OAuth

远程 Server 走 OAuth 时，配置完再登录一次：

```powershell
claude mcp login my-remote
```

SSH / 无头环境（打不开浏览器）加 `--no-browser`，它会打印授权 URL，让你把回调地址粘回来：

```powershell
claude mcp login my-remote --no-browser
```

服务商要求预注册 OAuth 客户端时，加 `--client-id`；需要固定回调端口（服务端登记了 redirect URI）时用 `--callback-port`：

```powershell
claude mcp add --transport http my-remote https://mcp.example.com/mcp `
  --client-id your-client-id --callback-port 8080
```

`--client-secret` 会交互式提示输入，也可以提前设 `MCP_CLIENT_SECRET` 环境变量，避免写进命令历史。清掉已存的凭据用 `claude mcp logout <name>`。

### 3.4 add-json（一次给完整配置）

需要一次配多个字段（或从别处抄来一段配置）时：

```powershell
claude mcp add-json my-server '{"type":"http","url":"https://mcp.example.com/mcp","headers":{"Authorization":"Bearer xxx"}}'
```

支持 stdio / SSE / HTTP / WebSocket 四种，同样接受 `-s` 指定作用域。

### 3.5 查看、验证和删除

```powershell
claude mcp list                  # 列出所有 Server，并做健康检查
claude mcp get codegraph         # 看某个 Server 的作用域、状态、命令
claude mcp remove codegraph      # 删除（自动从它所在的作用域删）
claude mcp remove codegraph -s user   # 指定作用域删
```

`claude mcp list` 会真的去连一遍，输出形如：

```text
codegraph: codegraph serve --mcp - ✔ Connected
```

连不上的会标出来，这是**最快的自检手段**。

---

## 4. 用配置文件配

CLI 适合加单条，配置文件和团队协作、密钥外置打交道更多。

### 4.1 项目级 `.mcp.json`

放在**项目根目录**，可以提交进 git：

```json
{
  "mcpServers": {
    "context7": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp"]
    },
    "internal-api": {
      "type": "http",
      "url": "https://mcp.example.com/mcp",
      "headers": {
        "Authorization": "Bearer ${INTERNAL_MCP_TOKEN}"
      }
    }
  }
}
```

可用的字段实测为：`type`、`command`、`args`、`env`、`url`、`headers`（以及 OAuth 相关的客户端配置）。

> 只提交**不含密钥**的 `.mcp.json`。`env` 和 `headers` 是最容易夹带密钥的两处，提交前扫一眼。

### 4.2 密钥外置

`.mcp.json` 支持 `${VAR}` 和 `${VAR:-default}` 展开，可用在 `command` / `args` / `env` / `url` / `headers`（官方文档口径）。这样团队提交共用结构，密钥各自在本地环境变量里：

```json
{
  "mcpServers": {
    "internal-api": {
      "type": "http",
      "url": "https://mcp.example.com/mcp",
      "headers": { "Authorization": "Bearer ${INTERNAL_MCP_TOKEN}" }
    }
  }
}
```

> ⚠️ 官方文档口径：**变量在环境里不存在、又没写默认值时，整个配置解析会失败**。所以换机器 / 新同事拉下仓库后第一件事是把这些变量配好，否则表现是"Server 一个都加载不出来"，而不是只少那一个。

### 4.3 批准与 settings 三个键

`.mcp.json` 来自仓库，等于**别人可以替你决定启动什么进程**，所以 Claude Code 加了批准这一关：首次使用项目级 Server 会弹一次确认；没批准的在 `claude mcp list` 里显示为 `⏸ Pending approval`，也不会去连。

三个相关设置键（实测的描述文案）：

| 键 | 含义 |
|---|---|
| `enableAllProjectMcpServers` | 是否**自动批准**项目里所有 MCP Server |
| `enabledMcpjsonServers` | 已批准的 `.mcp.json` Server **白名单** |
| `disabledMcpjsonServers` | 已拒绝的 `.mcp.json` Server 名单 |

写在 `~/.claude/settings.json`：

```json
{
  "enabledMcpjsonServers": ["context7"]
}
```

**推荐用白名单而不是 `enableAllProjectMcpServers: true`** —— 后者等于对所有仓库开绿灯，克隆任何一个陌生项目都可能直接起一堆进程。误点了拒绝想反悔，用：

```powershell
claude mcp reset-project-choices
```

> ⚠️ **非交互模式拿不到批准提示**：`claude -p` 之类的无人值守运行不会弹确认，直接加载。所以**跑之前先看 `.mcp.json` 里有什么**，别在陌生仓库里直接跑自动化。

### 4.4 local / user 落在哪

这两类不写在项目里，而在 `~/.claude.json`：

- `user` 作用域 → 该文件**顶层**的 `mcpServers`（本机实测：codegraph 装在这里，所有项目可见）
- `local` 作用域 → `projects.<项目路径>.mcpServers`（每个项目一段）

这个文件**不要提交、不要手工大改**，用 `claude mcp add/remove` 维护最稳。

---

## 5. 在会话里用

### 5.1 `/mcp`

会话里敲 `/mcp` 打开管理界面。实测支持的子参数：

```text
/mcp
/mcp reconnect <server>
/mcp enable [<server>|all]
/mcp disable [<server>|all]
```

- 刚改完配置**不一定要重启会话**，`/mcp reconnect <server>` 通常就够了；
- 临时不想用某个 Server 用 `disable`，比删配置安全，`enable` 能加回来；
- 列表里能看到每个 Server 的连接状态和它提供的工具。

### 5.2 工具长什么样

MCP 工具在模型眼里是 `mcp__<server>__<tool>`，比如 Server 名叫 `context7`、工具叫 `resolve-library-id`，就是 `mcp__context7__resolve-library-id`。claude.ai connector 接进来的形式是 `mcp__<connector>__<toolName>`。

想让它列出工具、又不想触发写操作，可以这么说：

```text
列出当前可用的 MCP 工具，并说明每个工具的用途。不要执行写入操作。
```

> **Server 会占上下文**：每个 Server 的工具定义都要进上下文，装多了会挤掉真正要看的代码。这就是第 6 节反复强调"按需装"的原因，也是 `/context` 值得常看的原因。

### 5.3 输出上限与超时

三个实测的默认值：

| 环境变量 / 配置 | 默认值 | 管什么 |
|---|---|---|
| `MAX_MCP_OUTPUT_TOKENS` | **25000** | 单次 MCP 工具结果允许的 token 上限 |
| `MCP_TIMEOUT` | **30000** ms | Server 启动 / 连接等待 |
| `MCP_CONNECT_TIMEOUT_MS` | **5000** ms | 连接阶段的超时 |
| `MCP_TOOL_TIMEOUT` | 见下 | 单次工具调用的执行超时 |

- **结果超过 `MAX_MCP_OUTPUT_TOKENS` 是报错，不是自动截断** —— 会提示结果超过允许的 token 数。想看完整结果就调大这个值：在 `~/.claude/settings.json` 的 `env` 段里写 `"MAX_MCP_OUTPUT_TOKENS": "50000"`（和其它 `env` 项一样是**字符串**，见 [`README.md`](./README.md) 5.4 节同类写法）。
- 单个 Server 也能单独配工具调用超时，它会**覆盖** `MCP_TOOL_TIMEOUT`；实测小于 1000ms 的值会被忽略。
- 超时先怀疑 Server 本身慢或网络问题，**别一上来就调大数值**。

---

## 6. 常用 Server 推荐清单

下面几个的包名都已确认存在（`npm view` 实测）。**装之前先用 `claude mcp add ... -s local` 试，确认有用了再考虑提到 `user` 或写进 `.mcp.json`。**

| Server | 用途 | 装法 | 适合谁 |
|---|---|---|---|
| codegraph | 把代码库预建成知识图谱，一次调用拿到相关代码 | `claude mcp add codegraph -- codegraph serve --mcp` | 大仓库里 agent 靠 grep 找代码的场景，详见 [`../README.md#codegraph`](../README.md#codegraph) |
| context7 | 查第三方库 / 框架的最新文档 | `claude mcp add context7 -- npx -y @upstash/context7-mcp` | 经常被版本差异坑到，讨厌模型凭记忆答 API |
| filesystem | 受控地读写指定目录 | `claude mcp add fs -- npx -y @modelcontextprotocol/server-filesystem <路径>` | 需要让它访问项目外的固定目录 |
| playwright | 驱动浏览器：打开页面、点击、截图 | `claude mcp add playwright -- npx -y @playwright/mcp@latest` | 调前端、复现 UI bug、写端到端测试 |
| sentry | 查线上错误和堆栈 | `claude mcp add --transport http sentry https://mcp.sentry.dev/mcp` | 排查线上报错（需 OAuth 登录） |
| dbx | 通过你已配置好的连接直接查库 | `claude mcp add dbx -- npx @dbx-app/mcp-server` | 常和数据库打交道，详见 [`../README.md#dbx`](../README.md#dbx) |

几条选型经验：

- **本地/自建项目优先 `-s local`**，先在当前项目验证，别一上来装成 `user` 污染所有项目。
- **只读工具先跑**，确认返回质量再放开写操作。
- **不要给 `npx -y` 的包不写版本**：不锁版本意味着每次启动都拉最新，某个未来版本被投毒就会直接在你机器上跑起来。能锁版本就锁（`@upstash/context7-mcp@4.1.1`）。
- 团队要共用的 Server 才写进 `.mcp.json`；个人口味留在 `local` / `user`。

---

## 7. 安全注意事项

- **MCP 不绕过 Claude Code 现有的权限和审批**：权限模式（见 [`README.md`](./README.md) 5.2 节）、`/permissions` 规则、工具确认弹窗**照样生效**。Server 提供的是工具，不是绕过审批的手段。
- **Server 拿到的是你给的权限**：`-e` 注入的环境变量、`env` 块里的密钥、`headers` 里的 Token，等于直接交给了它。只给必要的，能只读就别给写。
- **`.mcp.json` 是代码，不是配置**：克隆陌生仓库时先看这个文件，它决定本机会起什么进程；非交互模式（`claude -p`）**不会弹批准**（4.3 节）。
- **MCP 返回的内容属于外部输入**：Server 返回的文本里可能夹带指令。**不要盲目执行返回内容里出现的命令**，把它当数据看，不当指令看。
- **密钥不进仓库**：`.mcp.json`、Skill、Hook 里都不写 Token / API Key / 私钥。走 `${VAR}` 或环境变量（4.2 节）。
- **只读验证优先**：先用只读请求确认工具行为符合预期，再允许写入、删除、发消息这类操作。
- **第三方 Server 要确认来源**：装之前确认仓库归属、维护活跃度和它要访问的范围；公司合规要求先过一遍（见 [`../README.md`](../README.md) 第 6 节）。

---

## 8. 排查

**先跑这几条：**

```text
claude mcp list          列出所有 Server 并做健康检查（最快自检）
claude mcp get <name>     看单个 Server 的作用域 / 状态 / 命令
/mcp                     会话内看连接状态和工具列表
/context                 看上下文占用（Server 装太多会挤掉代码）
```

**常见现象对照：**

| 现象 | 先查什么 |
|---|---|
| `/mcp` 里根本看不到 | 配置写进了哪个作用域：`claude mcp list` 里有没有；`~/.claude/settings.json` 里写 `mcpServers` 是**无效**的（第 2 节） |
| 显示 `⏸ Pending approval` | 项目级 `.mcp.json` 还没批准。交互会话里确认一次，或用 `enabledMcpjsonServers` 加白名单（4.3 节） |
| `claude mcp list` 里连不上 | 是最常见的失败：命令路径对不对（`npx` / `node` / `python` 装了没）、URL 通不通、`type` 写没写对 |
| 加了 URL 却去起进程了 | 忘了 `--transport http`，默认按 stdio 处理（3.2 节） |
| 一个 Server 都加载不出来 | 多半是 `${VAR}` 变量在环境里不存在且无默认值，整个配置解析失败（4.2 节） |
| 工具调用超时 | Server 慢或网络问题，先排查再考虑调 `MCP_TOOL_TIMEOUT`（5.3 节） |
| 结果报超过 token 上限 | 调大 `MAX_MCP_OUTPUT_TOKENS`（默认 25000，5.3 节） |
| OAuth 登录失败 | 确认 Server 支持 MCP OAuth；无头环境用 `--no-browser`；要预注册客户端就补 `--client-id`（3.3 节） |
| 改完配置不生效 | `/mcp reconnect <server>`，插件来源的重开会话；`~/.claude.json` 手工改坏了用 CLI 重加一遍 |
| 工具太多、上下文被吃掉 | `/context` 看占用，`/mcp disable` 掉暂时不用的 Server（5.2 节） |
| 在陌生仓库里它自己起了进程 | `.mcp.json` 在你批准过或非交互模式下会直接加载，先看文件内容（4.1 / 4.3 节） |

---

## 一页速查

```text
是什么     给模型加外部工具/数据源；是"加工具"，不是"教流程"

作用域      local（默认）  ~/.claude.json → projects.<路径>.mcpServers   只有你，只在这个项目
            project      项目根 .mcp.json（可提交）                    全团队
            user         ~/.claude.json → 顶层 mcpServers             只有你，所有项目
            同时生效；同名 local > project > user，整体覆盖不合并
            ⚠️ settings.json 里写 mcpServers 无效

添加        claude mcp add <name> -- npx -y <包>                  stdio（-- 后面是启动命令）
            claude mcp add <name> -e KEY=value -- <命令>           注入环境变量
            claude mcp add --transport http <name> <url>          远程（不写 transport 会当 stdio）
            claude mcp add --transport http <name> <url> -H "X-Api-Key: xxx"
            claude mcp add-json <name> '{"type":"http","url":"..."}'
            都支持 -s local|user|project 指定作用域

管理        claude mcp list         列出 + 健康检查
            claude mcp get <name>    看作用域/状态/命令
            claude mcp login <name>  OAuth 登录（--no-browser 无头）
            claude mcp logout <name> 清凭据
            claude mcp remove <name> [-s 作用域]
            claude mcp reset-project-choices   重置批准/拒绝记录

会话内      /mcp  /mcp reconnect <server>  /mcp enable|disable [<server>|all]
            工具名 mcp__<server>__<tool>

配置文件    项目根 .mcp.json → mcpServers: type/command/args/env/url/headers
            ${VAR} / ${VAR:-default} 展开；变量缺失且无默认值会让整份配置解析失败
            批准：enableAllProjectMcpServers / enabledMcpjsonServers / disabledMcpjsonServers
            （推荐白名单；非交互 claude -p 不弹批准，直接加载）

默认值      MAX_MCP_OUTPUT_TOKENS 25000      超了报错，不自动截断
            MCP_TIMEOUT 30000 ms            Server 启动/连接等待
            MCP_CONNECT_TIMEOUT_MS 5000 ms
            MCP_TOOL_TIMEOUT                单 Server 配置可覆盖；<1000ms 忽略

安全        MCP 不绕过权限模式和审批；密钥走 ${VAR}/环境变量，不进仓库
            .mcp.json 是代码，克隆陌生仓库先看；MCP 返回内容是外部输入，不要当指令执行
            npx -y 不锁版本 = 每次拉最新，能锁就锁
```
