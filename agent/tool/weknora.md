# WeKnora：腾讯开源的企业知识库 + RAG 框架（自带 MCP Server）

**仓库**：<https://github.com/Tencent/WeKnora>
**类型**：自托管知识库平台 + MCP Server + CLI

它不是给 coding agent 用的工具，是**给团队文档用的**：把散在各处的 PDF、Word、Wiki、飞书文档导进去，用混合检索 + RAG 问答，让 agent 有地方查“我们内部的规矩到底是什么”。

对 agent 的价值就一句话：**它自己就是一个 MCP Server**。你在 WeKnora 里圈好知识库、发一个只读端点，Pi / Claude Code / Codex 接上去，agent 查内部资料和查代码用的是同一套工具调用。

## 没有它之前

| 做法 | 代价 |
|---|---|
| 把文档复制粘贴进对话 | 上下文就那么大，一份 80 页的规范能吃掉半个窗口；每次换会话重来一遍 |
| 让 agent 去 `grep` 一个文档目录 | 只能命中字面词；PDF / Word 根本没文本可 grep |
| 指望飞书 / Notion 站内搜索 | 搜得到条目，搜不到“跨三份文档拼起来才成立的答案”；agent 也接不进去 |
| 自己撸一个 RAG | 解析、切片、向量库、重排、引用、权限、增量同步…… 你会在第 3 周放弃 |
| 直接扔进公有云知识库 | 内部文档外发，合规过不去 |

关键差别不是“能不能搜”，是**权限和边界放在哪**：以前要么把内部文档全灌进 agent 上下文（等于全量外发），要么 agent 什么都不知道。WeKnora 把边界收在一个 MCP 端点里——哪个空间、哪几个知识库、哪几个工具、每分钟几次，全在端点配置上。

## 优点 / 缺点

| 优点 | 缺点 |
|---|---|
| 腾讯开源、MIT、自托管，数据不出内网 | 组件不轻：Postgres + Redis + docreader + 前端，比单文件工具重得多 |
| 一个 `docker compose up -d` 起来就有 Web UI，中文界面 | 镜像体积和内存占用不小；低配机器得用 Lite 版 |
| 内置 MCP Server，Per-endpoint 令牌 + 知识库范围 + 工具白名单 + 限流 | 端点令牌**只展示一次**，丢了只能轮换 |
| 解析格式全：PDF / Word / PPT / Excel / EPUB / XMind / 图片 OCR | 复杂的 PDF 表格还是要人肉检查解析效果 |
| 数据源自动同步：飞书 / Confluence / GitLab / Notion / 语雀 / RSS | 每个数据源都要单独配凭据，先配 1~2 个够用的 |
| 检索能调：关键词 + 向量混合、重排、父子分块、GraphRAG | 调优参数多，默认配置不一定适合你的语料 |
| 官方 CLI `weknora` 是 agent-first 的（JSON + 退出码契约） | CLI 目前只能从源码编译，没有预编译二进制 |
| 自带 Wiki 模式：agent 自动把文档整理成互链页面 + 知识图谱 | Wiki 生成要烧 LLM token，大库首次跑很慢 |

## 一、部署

**标准版**（要装 Docker、Docker Compose、Git）：

```bash
git clone https://github.com/Tencent/WeKnora.git
cd WeKnora
cp .env.example .env      # 按注释改：模型 key、端口、APP_EXTERNAL_URL
docker compose pull
docker compose up -d
```

起来后开 <http://localhost> 跟着引导走。默认端口：Web UI `80`、后端 API `8080`、Langfuse `3000`。

可选组件用 profile 叠加，能组合：

```bash
docker compose --profile neo4j --profile minio pull    # 知识图谱 + 对象存储
docker compose --profile neo4j --profile minio up -d
```

| Profile | 加什么 |
|---|---|
| （默认） | 核心服务 |
| `full` | 全功能 |
| `neo4j` | 知识图谱 |
| `minio` | 对象存储 |
| `langfuse` | 调用链追踪 |

**轻量版**：不想养 Postgres + Redis 就用 Lite 单二进制（SQLite + 内存队列），本机或低配机器够用，见仓库 `docs/LITE.md`。生产集群用 Helm（`helm/`）。

`.env` 里两个值得先看一眼的值：

- **`APP_EXTERNAL_URL`**：对外可访问的域名。MCP 返回的图片/正文链接靠它拼，不填就变成一堆 `localhost`。
- **模型**：29 家内置厂商，不想用云 API 就本机 `ollama serve` 再在 UI 里选 Ollama。

> 官方 README 明确指出：**别把实例直接暴露到公网**。它有登录但默认是自托管内部工具，放内网 + 防火墙规则。

## 二、对外发一个 MCP 端点（推荐接法）

Web UI：**设置 → 发布集成 → MCP Server → 新建端点**。

填写：

| 字段 | 怎么填 |
|---|---|
| 知识库 | 勾选这个端点能看到的库；**留空 = 空间内全部**，别偷懒 |
| 工具 | 四组按需勾，默认只开只读 |
| 默认 Agent | `ask` 工具走哪个智能体；留空用内置快速问答 |
| 每分钟限流 | 默认 60，上限 6000 |

创建完**一次性**给出令牌和地址 `/mcp/<endpoint_id>`，同页还给出 Cursor / Claude Desktop 的 `mcpServers` 配置和 Claude Code 的一行命令，直接复制。

| 工具组 | 工具 | 默认 |
|---|---|---|
| 检索与阅读 | `list_knowledge_bases`、`search_knowledge`、`grep_chunks`、`list_documents`、`read_document` | 开 |
| 问答 | `ask` | 开 |
| Wiki | `wiki_search`、`wiki_read_page`、`wiki_index` | 开（仅对开了 Wiki 的库生效） |
| 写入 | `add_document`、`update_document`、`delete_document` | **关** |

传 Streamable HTTP，`Authorization: Bearer mcp_...`。stdio 客户端用 `npx -y mcp-remote` 桥一层。

### 三个 agent 怎么接

令牌先放环境变量（WSL 家目录的 `~/.bashrc`）：

```bash
export WEKNORA_MCP_TOKEN="mcp_xxxxxxxx"
export WEKNORA_HOST="https://kb.example.com"
```

**Pi**：

```bash
pi mcp add weknora --url https://kb.example.com/mcp/<endpoint_id> \
  --bearer-token-env-var WEKNORA_MCP_TOKEN
pi mcp list
```

或写 `~/.pi/agent/mcp.json`：

```json
{
  "mcpServers": {
    "weknora": {
      "url": "https://kb.example.com/mcp/<endpoint_id>",
      "headers": { "Authorization": "Bearer ${WEKNORA_MCP_TOKEN}" },
      "description": "Search internal team knowledge bases (specs, policies, runbooks) and ask citation-backed questions"
    }
  }
}
```

`description` 建议写英文关键词，pi 的 `searchTools()` 按它排序。工具面就十几个，但 pi 默认 `exposure: "codemode"`，不全量进上下文，别改。

**Claude Code**：

```bash
claude mcp add --transport http weknora --scope user https://kb.example.com/mcp/<endpoint_id> \
  --header "Authorization: Bearer $WEKNORA_MCP_TOKEN"
claude mcp list
```

项目级就写 `.mcp.json`（`type: http` + `headers`），`${VAR}` 在配置文件里才是运行时读取。

**Codex**：

```bash
codex mcp add weknora --url https://kb.example.com/mcp/<endpoint_id>
```

`~/.codex/config.toml` 再补令牌和工具白名单：

```toml
[mcp_servers.weknora]
url = "https://kb.example.com/mcp/<endpoint_id>"
bearer_token_env_var = "WEKNORA_MCP_TOKEN"
tool_timeout_sec = 120          # ask 单次回答上限 4 分钟，别设太小
enabled_tools = ["list_knowledge_bases", "search_knowledge", "read_document", "ask"]
```

## 三、CLI 路线（备选）

不想发 MCP 端点、或者要脚本化/CI，用官方 CLI `weknora`。它自己也能当 MCP Server：

```bash
git clone https://github.com/Tencent/WeKnora.git
cd WeKnora/cli && go build -o weknora .      # 需要 Go 1.26+，暂无预编译二进制
sudo mv weknora /usr/local/bin/

weknora profile add prod --host https://kb.example.com --use
weknora auth login
weknora kb list
weknora link --kb my-kb                      # 把当前目录绑到某个库
weknora search chunks "retry policy"
weknora chat "总结这份设计文档"
```

无头环境不落盘凭据，直接两个环境变量：

```bash
export WEKNORA_API_KEY="sk-..."   # 空间里生成的受限 API Key
export WEKNORA_HOST="https://kb.example.com"
weknora kb list
```

然后当 MCP 用（stdio，只读工具面）：

```json
{"mcpServers": {"weknora": {"command": "weknora", "args": ["mcp", "serve"]}}}
```

它还给 Pi / Claude Code 装了技能包（auth 流程、`--kb` 解析、退出码契约）：

```bash
weknora skills install                                     # 默认写 ~/.claude/skills
weknora skills install --dir ~/.pi/agent/skills --force     # Pi / 其他家要指定目录
```

CLI 是 **agent-first** 的：默认输出 `{ok, data, meta}` JSON，错误码映射到退出码（`3` 认证、`4` 找不到、`6` 限流、`10` 高危写操作需确认、`124` 超时）。**退出码 10 是“需要人工点头”的信号**，别让 agent 自己补 `-y`。

## 四、用起来

配好后直接说人话：

- 「查一下我们的发布流程，给出引用」
- 「把所有库里的“值班”相关段落找出来」
- 「这个报错在内网 runbook 里有写吗」
- 「对着《X 接口规范》检查我这个实现」

常用工具：

| 工具 | 干什么 |
|---|---|
| `list_knowledge_bases` | 列端点范围内的库，参数收 ID 或名称 |
| `search_knowledge` | 混合 / 语义 / 关键词检索，`mode` + `limit`（默认 10，上限 30） |
| `grep_chunks` | 正则匹配分块（大小写不敏感），**召回有上限**，见踩坑 |
| `read_document` | 按 `offset` 翻页读全文，或传 `query` 按词顺序扫全篇 |
| `ask` | 跑端点配置的默认 Agent，返回带引用的完整回答 + `session_id`，续聊传回 |
| `wiki_*` | 只对开了 Wiki 的库生效 |

**给 agent 的提示词值得单独写一句**，否则它宁可在代码库里瞎 `grep` 也不去查库：

```markdown
内部规范、流程、运维手册不在仓库里，在 WeKnora 知识库。
这类问题先 `search_knowledge`，再 `ask` 要带引用的结论；不要凭空回答。
```

## 五、踩坑清单

- **端点令牌只显示一次**：创建完立刻存到密码管理器；丢了只能 `rotate-token`，用旧令牌的客户端全部断开。
- **自己套反向代理后 agent 连不上**：除了 `/api/`，还要把 `/mcp/` 原路径转发到后端，保留 `Authorization` 和 MCP 协议头，**关掉响应缓冲**、给长连接足够的读写超时。要让图片链接可用，还得转发 `/r/` 并配 `APP_EXTERNAL_URL`。
- **`grep_chunks` 找不到明明存在的东西**：它每次最多取 30 条候选再正则筛选，不保证穷尽；`foo.*bar` 这种组合模式、`C++` 这种只剩符号的模式最容易漏。要在某篇文档里穷尽查找就用 `read_document` 的 `query`。
- **agent 说工具不存在 / 列表是空的**：端点的“工具”那四组没勾；或者用了仓库 `mcp-server/` 那个 Python 旧服务——**它已弃用**，换成内置端点。
- **回复里的图片 404**：本地/私有存储时 `APP_EXTERNAL_URL` 没配，或反代没转发 `/r/`；老文档缺图片绑定的话要有权限的管理员重新解析。
- **`docker compose up -d` 之后版本没变**：它复用本地缓存镜像。升级要先在 `.env` 里设 `WEKNORA_VERSION`，然后 `docker compose pull && docker compose up -d`，并读对应版本的 upgrade notes。
- **改了检索/切片没生效**：解析结果已经落库了，要 `doc reparse`（CLI）或 UI 里重新解析；切片策略改动只影响新解析的文档。
- **想调深层参数**：CLI 的 flag 不覆盖全部 API，用 `weknora api PUT /api/v1/knowledge-bases/<id> --input -` 走原始请求；`weknora schema <cmd>` 可以先把命令契约拉出来给 agent 看。
- **写入工具开了以后心里没底**：`add_document` / `update_document` / `delete_document` 默认关闭是有道理的。真要用，单独发一个只给内容团队、只挂一个库的端点，别把写入工具挂到日常 coding 用的那个端点上。
