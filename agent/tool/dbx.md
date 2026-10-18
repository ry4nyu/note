# dbx：Windows 桌面版数据库客户端 + MCP Server

**仓库**：<https://github.com/t8y2/dbx>
**类型**：桌面客户端 + MCP Server

25MB 单文件数据库客户端，支持 100+ 种数据库（MySQL、PostgreSQL、SQLite、Redis、MongoDB、ClickHouse、SQL Server、Oracle……），不需要 Java 运行时。

两个用途：**Windows 上当 GUI 客户端用**，以及**让 WSL 里的 agent 通过 MCP 查你已配好的连接**。

## 没有它之前

| 做法 | 代价 |
|---|---|
| DataGrip / Navicat / DBeaver | 要 Java、要许可证、启动慢；而且它们都只是给人看的 |
| `mysql -e` / `psql -c` 手敲 | 密码进 shell 历史；改一个字段名要重开一次命令行 |
| 手动“贴 schema + 贴数据”给 agent | agent 自己看不到库，你成了人肉代理：查一次、贴一次、改 SQL 再贴一次 |
| 把库密码写进环境变量让 agent 直连 | 能跑，但没人管得住它 `DROP TABLE`，也没审计 |

关键差别不是“能不能连库”，而是**权限闸门在那里**：以前要么完全不放，要么一旦放开就什么都能干；DBX 把闸门放在 UI 里，三档可选、每次请求重读。

## 优点 / 缺点

| 优点 | 缺点 |
|---|---|
| 25MB 单文件、100+ 种数据库、无 Java | 桌面端在 Windows 侧，WSL 里就得靠 HTTP 跳一层（见第二节的架构约束） |
| 权限三档在 UI 里改，改完立刻生效 | 要多养一个常驻进程（`dbx-mcp.exe --http`） |
| MCP 走 Streamable HTTP + Bearer，连接和密钥都留在 Windows | 密钥在 Windows 凭据管理器，Linux 版 MCP 读不到 —— 不是配置问题，是设计 |
| agent 能拿紧凑 schema，比自己翻表省上下文 | 开了读写权限后，agent 真能改数据；日常建议只给只读 |
| 查完能让结果直接回到桌面端看 | GUI 深度不如专业客户端（执行计划可视化、数据同步这些） |

## 一、Windows 桌面版安装

```powershell
winget install t8y2.dbx                    # 推荐
scoop bucket add dbx https://github.com/t8y2/scoop-bucket; scoop install dbx
```

或者从 [releases](https://github.com/t8y2/dbx/releases) 下安装包。

装完先做两件事：

1. **建连接**：新建 → 填 host / port / 用户名 / 密码 → 保存。
2. **配权限**：设置 → MCP，选一档 + 把「允许访问的连接」勾上要开放的那几条。

| 模式 | 允许 |
| --- | --- |
| 只读 | 查询 + 读元数据 |
| 数据读写 | 普通 insert、带有效过滤条件的 update/delete、普通 Redis 写 |
| 完全访问 | 再加 DDL、`TRUNCATE`、大范围 update/delete、`FLUSH*` |

权限**只在 DBX 里配**，客户端配置里没有权限开关；这一页每次请求都会重读，改完立刻生效。

数据在 `%APPDATA%\com.dbx.app\dbx.db`，**密码是用 Windows 凭据管理器里的密钥加密的**——这点决定了下一步的架构。

## 二、为什么 MCP 进程要跑在 Windows 侧

在 WSL 里直接跑 Linux 版 `dbx-mcp` 是行不通的：它读得到 `/mnt/c/Users/.../dbx.db`，但拿不到 Windows 凭据管理器里的那把密钥，一连库就会报 `SECRET_KEY_UNAVAILABLE` / `KEYRING_ACCESS_FAILED`（或者 `DATA_MIGRATION_REQUIRED`）。

所以正确的切分是：

```
Windows: DBX 桌面版 + dbx-mcp.exe --http (127.0.0.1:5225)
                    ▲
                    │ Streamable HTTP + Bearer Token
WSL:      pi / Claude Code / Codex
```

好处：连接和密钥都留在 Windows，不用迁移数据；`dbx_open_table` 这类要在桌面端打开表的工具也能用。

跨 WSL 边界本来最麻烦的是网络，但这台机器 `~/.wslconfig` 已经是 `networkingMode=mirrored`，Windows 上**只绑 127.0.0.1** 的端口在 WSL 里 `curl localhost` 直接通（实测过）——不用开防火墙，也不需要 `--http-allow-remote` 那套白名单。

## 三、Windows 侧：起 MCP HTTP 服务

```powershell
irm https://dbxio.com/install-mcp.ps1 | iex      # 装到 %USERPROFILE%\.dbx\bin\dbx-mcp.exe

[Environment]::SetEnvironmentVariable("DBX_MCP_HTTP_TOKEN", "<一串长随机>", "User")
$env:DBX_MCP_HTTP_TOKEN = "<一串长随机>"
& "$env:USERPROFILE\.dbx\bin\dbx-mcp.exe" --http
```

监听 `http://127.0.0.1:5225/mcp`。重跑安装命令就是升级。

想常驻：把上面那条 exe 命令做成开机启动项或计划任务（token 已经落在用户环境变量里，不用写进启动命令）。

## 四、WSL 里的三个客户端

先在 WSL 的 `~/.bashrc` 里放同一个 token：

```bash
export DBX_MCP_TOKEN="<和 DBX_MCP_HTTP_TOKEN 一样的值>"
```

URL 统一 `http://localhost:5225/mcp`。三家的配置都在 WSL 的家目录下，不是 Windows。

| Agent | 一行命令 | 配置文件 |
|---|---|---|
| Pi | `pi mcp add dbx --url http://localhost:5225/mcp --bearer-token-env-var DBX_MCP_TOKEN` | `~/.pi/agent/mcp.json` |
| Claude Code | `claude mcp add --transport http dbx --scope user http://localhost:5225/mcp --header "Authorization: Bearer $DBX_MCP_TOKEN"` | 项目 `.mcp.json` / `~/.claude.json` |
| Codex | `codex mcp add dbx --url http://localhost:5225/mcp` | `~/.codex/config.toml` |

### pi

```bash
pi mcp add dbx --url http://localhost:5225/mcp --bearer-token-env-var DBX_MCP_TOKEN
pi mcp list
```

或写 `~/.pi/agent/mcp.json`（项目级是 `.pi/mcp.json`，需要项目已信任）：

```json
{
  "mcpServers": {
    "dbx": {
      "url": "http://localhost:5225/mcp",
      "headers": { "Authorization": "Bearer ${DBX_MCP_TOKEN}" },
      "description": "Query MySQL/PostgreSQL/Redis/MongoDB connections configured in DBX"
    }
  }
}
```

`description` 值得写：pi 把它放进 system prompt，`searchTools()` 也按它排序。

pi 默认 `exposure: "codemode"`，dbx 那 25 个工具定义**不会全量进上下文**，agent 按需用 `searchTools()` 找。别改成 `direct`——那是 6~7k token 常驻。

### Claude Code

项目根目录 `.mcp.json`（HTTP 服务器不用装任何东西）：

```json
{
  "mcpServers": {
    "dbx": {
      "type": "http",
      "url": "http://localhost:5225/mcp",
      "headers": { "Authorization": "Bearer ${DBX_MCP_TOKEN}" }
    }
  }
}
```

或者命令行：

```bash
claude mcp add --transport http dbx --scope user http://localhost:5225/mcp \
  --header "Authorization: Bearer $DBX_MCP_TOKEN"
claude mcp list
```

注意命令里是 `$DBX_MCP_TOKEN`（会被 shell 展开成明文写进 `~/.claude.json`），配置文件里的 `${DBX_MCP_TOKEN}` 才是运行时读取。

### Codex

```bash
codex mcp add dbx --url http://localhost:5225/mcp
codex mcp list
```

或 `~/.codex/config.toml`：

```toml
[mcp_servers.dbx]
url = "http://localhost:5225/mcp"
bearer_token_env_var = "DBX_MCP_TOKEN"
tool_timeout_sec = 120
```

Codex 顺手能把不用的工具关掉，明显省上下文：

```toml
enabled_tools = ["dbx_list_connections", "dbx_list_databases", "dbx_list_tables",
                 "dbx_describe_table", "dbx_get_schema_context", "dbx_execute_query"]
default_tools_approval_mode = "prompt"
```

## 五、用起来

配好后直接说人话：

- 「列一下我的 dbx 连接」
- 「production-postgres 里有哪些表」
- 「orders 表结构发我」
- 「统计最近七天创建的订单」
- 「把这个结果在 DBX 里打开」

常用工具：

| 工具 | 干什么 |
| --- | --- |
| `dbx_list_connections` / `dbx_list_databases` | 列连接和库 |
| `dbx_list_tables` / `dbx_describe_table` | 表、字段 |
| `dbx_get_schema_context` | 一次拿紧凑 schema，比挨个翻表省事 |
| `dbx_execute_query` | 执行 SQL / Mongo shell，默认 100 行，`max_rows` 最大 1000 |
| `dbx_execute_batch` | 一次跑多条语句，可带 `use_transaction` |
| `dbx_open_session` / `dbx_close_session` | 事务用，空转 30 分钟回收 |
| `dbx_execute_redis_command` | Redis 命令 |
| `dbx_open_table` / `dbx_execute_and_show` | 在桌面端打开表或结果给人看（需要 DBX 开着） |

DBX 桌面端**不用一直开着**也能查库，只有最后那两个 UI 工具需要。

## 六、备选路线

不想让 Windows 上多一个常驻进程的话，只有两条路，代价都比上面大：

- **换便携版**：便携版的密钥是文件 `data\.dbx\secret.key`，WSL 里可以 `DBX_DATA_DIR=/mnt/d/DBX/data` + `DBX_SECRET_KEY_FILE=/mnt/d/DBX/data/.dbx/secret.key` 直接读，用 stdio 跑 Linux 版 `dbx-mcp`。代价是整个数据目录要搬家，且跨 `/mnt` 的 SQLite 并发写会慢。
- **用 Docker 版**：`docker run -d -p 4224:4224 -v dbx-data:/app/data t8y2/dbx:latest`，WSL 里 `DBX_WEB_URL=http://localhost:4224` + `DBX_WEB_PASSWORD` 走 stdio。代价是连接库跟桌面版是两套，要重建。

装好的桌面版**不能**用第一条（它的密钥不在文件里），这是关键区别。

## 七、踩坑清单

- **agent 说没有连接 / 工具列表是空的**：设置 → MCP 里的「允许访问的连接」没勾。
- **`SECRET_KEY_UNAVAILABLE`、`KEYRING_ACCESS_FAILED`**：你在 WSL 里跑 Linux 版 MCP 去读桌面版的数据了，回到 HTTP 方案。
- **「DBX 未运行」**：只有 `dbx_open_table` / `dbx_execute_and_show` 需要桌面端开着，普通查询不需要。
- **WSL 连不上 `localhost:5225`**：确认 `.wslconfig` 里是 `networkingMode=mirrored`（改完要 `wsl --shutdown`），否则得走 Windows 主机 IP + `--http-allow-remote` + 防火墙。
- **401 / 连上但报错**：Windows 侧 `DBX_MCP_HTTP_TOKEN` 和 WSL 侧 `DBX_MCP_TOKEN` 不一致，或者 WSL 那个变量没 `export`。
- **改完权限没反应**：确认改的是「设置 → MCP」，不是连接自己的只读开关；连接级只读是上限，任何模式都突破不了。
- **慢查询超时**：pi 加 `"timeout": 120`（秒），Codex 调 `tool_timeout_sec`；更稳的是把查询范围缩小。
- **想看 SQL 报错细节**：Windows 侧临时加 `DBX_MCP_DEBUG_SQL=1`，排完删掉。
- **DuckDB / Oracle / 达梦 / 金仓连不上**：走 DBX Agent/JDBC，先在桌面端 Driver Manager 装对应 Agent + JDBC Driver + JRE，MCP 不捆绑厂商驱动。
