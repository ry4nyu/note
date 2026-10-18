# codegraph：给 Agent 一张代码图谱

**仓库**：<https://github.com/colbymchenry/codegraph>
**类型**：MCP Server

大仓库里最常见的一幕：你问一个跨文件的问题，agent 开始 `grep` → `read` → 再 `grep`，八轮之后上下文塞满了，答案还没出来。

codegraph 把代码库**预建成知识图谱**再通过 MCP 暴露给 agent，让它一次调用就拿到该看的代码。它提供"符号 + 调用边 + 依赖"的完整图谱，内核用 Rust 写的，**100% 本地**。

## 没有它之前

| 做法 | 代价 |
|---|---|
| 让 agent `grep` / `ripgrep` 关键词 | 关键词一歪就零结果；重名符号要人工筛；一次审查要七八轮工具调用 |
| 直接 `read` 整个文件 | 一个文件几百行，真正的调用点只有三行，上下文全被无关代码吃掉 |
| 你自己在 prompt 里写"先看 `a.py`，再看 `b.py`" | 你成了那个图谱；换个问题就得重来 |
| `ctags` / IDE 跳转 | 人肉能用，agent 用不了 |

差别在**轮次**：grep 路线是"猜 → 错 → 再猜"，图谱路线是"一次拿到调用链"。

## 优点 / 缺点

| 优点 | 缺点 |
|---|---|
| 一次调用返回 verbatim 源码 + 调用路径 + 影响面 | **索引要维护**：改完代码 `codegraph status` 会挂 Pending sync，不同步就给你旧代码 |
| 全本地、Rust 内核，不上传代码 | 跨语言/动态语言（鸭子类型、反射、DI）解析不完整，结果当线索别当真相 |
| 支持 Claude Code / Codex / Cursor / opencode / Gemini 等自动配置 | **Pi 不在自动检测列表里**，得手动加一条 MCP |
| 工具集很小（`explore` / `node`），不像别的 MCP 一次灌几十个工具 | 小仓库纯属负担：建库、同步、占一条 MCP 配置，收益接近零 |

## 安装

```bash
npm i -g @colbymchenry/codegraph
# 或官方脚本（Linux / WSL）：
curl -fsSL https://raw.githubusercontent.com/colbymchenry/codegraph/main/install.sh | sh
```

然后：

```bash
codegraph install --yes --init    # 自动检测并配置 agent，同时索引当前项目
codegraph status                  # 看索引状态，有 Pending sync 会列出具体文件
codegraph upgrade                 # 升级（会自动识别安装方式）
```

> Windows 原生环境用 `install.ps1`（`irm ... | iex`），见 [`../windows.md`](../windows.md)。

> 注意：`install` **只配置 agent，不索引代码**，每个项目要自己跑一次 `codegraph init`。

## 三个 agent 怎么接

`codegraph install --yes` 会自己往 Claude Code（`~/.claude.json` / 项目 `.mcp.json`）和 Codex（`~/.codex/config.toml`）写配置，写完用各自命令确认；Pi 手动补一条。

| Agent | 接法 | 确认 |
|---|---|---|
| Pi | `pi mcp add codegraph -- codegraph serve --mcp` | `pi mcp list` |
| Claude Code | `claude mcp add --scope user codegraph -- codegraph serve --mcp` | `claude mcp list` |
| Codex | `codex mcp add codegraph -- codegraph serve --mcp` | `codex mcp list` |

三家的 stdio 启动命令是同一个：`codegraph serve --mcp`。Pi 的手写配置长这样（`~/.pi/agent/mcp.json`，项目级 `.pi/mcp.json`）：

```json
{
  "mcpServers": {
    "codegraph": {
      "command": "codegraph",
      "args": ["serve", "--mcp"],
      "env": { "CODEGRAPH_MCP_TOOLS": "explore,node" },
      "exposure": "codemode",
      "description": "Local CodeGraph index: explore indexed code - verbatim source, call paths, blast radius"
    }
  }
}
```

`CODEGRAPH_MCP_TOOLS` 收窄工具面，`description` 会被 Pi 放进 system prompt 也用于工具搜索排序 —— 两句都不写也能跑，只是 agent 更容易忘记用。

Pi 默认 `exposure: "codemode"`：工具定义不全量进上下文，agent 通过 `searchTools()` 找到再调。别急着改 `direct`，那点工具描述省下来的上下文，正好是 codegraph 要帮你省的那部分。

## 用起来

配好后正常提问就行，不用点名工具：

- "这个 `OrderService` 的调用方都有谁"
- "改这个接口会影响哪些地方"
- "这个函数在哪定义、谁在调"

每个仓库都要过一次：

```bash
cd ~/work/project
codegraph init            # 建索引
codegraph status          # 改完代码后看同步状态
```

## 踩坑清单

- **答案里出现已删除的代码**：索引过期。`codegraph status` 看 Pending sync，同步后再问。
- **agent 说没有这个工具**：MCP 没连上。`pi mcp list` / `claude mcp list` / `codex mcp list` 挨个确认；Pi 改完 `mcp.json` 要 `/reload`。
- **Pi 里 agent 从不主动用**：多半是 `description` 没写，或 `exposure` 被改成了别的值。`/mcp` 里看服务器状态和工具数。
- **结果只覆盖一半仓库**：`init` 的目录范围不对，或该语言没解析器；别当权威，当"快速线索"。
- **新项目一开始就装**：反过来 —— 先跑两周，确认 agent 真的在为大仓库发愁再装。

**适合谁**：几十万行以上、跨模块调用链复杂、agent 每天要翻七八个文件的仓库。几千行的项目，`grep` 比索引快。
