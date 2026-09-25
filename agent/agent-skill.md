# AI 编程 Agent 技能与工具推荐

这是一份**给同事的推荐清单**，收集了一批能实实在在提升日常开发体验的开源技能（Skill）、MCP 服务和命令行工具。

前置阅读：如果你还没装 Claude Code，先看同目录下的 [`claude使用教程.md`](./claude使用教程.md)。

**说明几点：**

- 下面所有工具都是**第三方开源项目**，不是 Anthropic / 官方出品。装之前建议先确认公司合规要求。
- 文中引用的百分比、倍数等效果数据**均为各项目 README 自述**，我没有逐一本地复现，当参考值看待。
- 安装方式我按 Windows 环境整理，非 Windows 的会单独标注。

---

## 0. 30 秒选型

| 你想解决的问题 | 装这个 | 类型 |
|---|---|---|
| Agent 说话太啰嗦，废话烧 token | [caveman](#caveman) / [i-have-adhd](#i-have-adhd) | Skill |
| Agent 过度设计，写一堆没用的代码 | [ponytail](#ponytail) | Skill |
| 命令输出太长，把上下文撑爆 | [rtk](#rtk) | CLI 代理 |
| 整体上下文太大，想全面压缩 | [headroom](#headroom) / [lean-ctx](#lean-ctx) | 代理 / MCP |
| 大仓库里 agent 找代码靠瞎 grep | [codegraph](#codegraph) / [codebase-memory-mcp](#codebase-memory-mcp) | MCP |
| 代码评审靠人肉，想加一层 AI 兜底 | [open-code-review](#open-code-review) | CLI |
| 让 agent 帮你写 Word / Excel / PPT | [OfficeCLI](#officecli) | CLI + Skill |
| 轻量数据库客户端 + 让 agent 直连查库 | [dbx](#dbx) | 桌面 + MCP |
| 把方案 / 架构图讲清楚给同事看 | [archify](#archify) | Skill |
| 开一堆 agent 跑长任务，窗口管不过来 | [herdr](#herdr) | 终端运行时 |

---

## 1. 省 token：让 Agent 少说废话

这类是 Skill（提示词规则包），装完对**所有对话**生效，不涉及代码执行，风险最低，建议先试这几个。

### caveman

**仓库**：<https://github.com/JuliusBrussee/caveman>

让 agent 用"原始人语"回答：砍掉所有铺垫、客套和复述问题，只留结论和动作。

> 普通 agent（69 token）："你的 React 组件重新渲染的原因，很可能是因为每次渲染都创建了新的对象引用……"
> caveman（19 token）："新对象引用每次渲染。内联对象 prop = 新引用 = 重渲染。用 `useMemo` 包起来。"

关键点是**代码、命令、文件路径、报错原文不压缩**，只砍散文部分；安全警告和确认提示也保持完整。

```powershell
npx skills add JuliusBrussee/caveman -g     # -g = 全局安装，所有项目生效
```

README 说支持 30+ 种 agent。另外它还有 proxy / middleware 组件压缩**输入**（日志、测试输出、JSON、diff），感兴趣再看。

### i-have-adhd

**仓库**：<https://github.com/ayghri/i-have-adhd>

换一个角度解决同一个问题：不是让话说得短，而是**强制输出结构**——先给动作、步骤编号、不寒暄。

> Before：一大段"好问题！让我想想……你的鉴权流程有几个环节……"
> After：`先跑 npm install jsonwebtoken@latest，然后改 src/auth.ts:42`，接着 1/2/3 编号步骤。

和 caveman 二选一即可，也可以都装（输出会更直给，看个人口味）。装法是把下面这句丢给你的 agent：

```text
Install the i-have-adhd skill/plugin from https://github.com/ayghri/i-have-adhd, refer to the repo's AGENTS.md for instructions.
```

### ponytail

**仓库**：<https://github.com/DietrichGebert/ponytail>

名字来自"公司里那位扎马尾的老程序员"——你甩给他五十行代码，他啥也不说，给你换成三行，还能跑。

这个 skill 专治 agent 的**过度设计**：你要个日期选择器，它给你装个 flatpickr、写个包装组件、加个样式文件，再顺便跟你讨论时区问题。ponytail 的规则是"只写任务需要的"，同时明确**不许砍校验、错误处理、安全和无障碍**。

官方在真实 Claude Code 会话上测的对比（vs 不装 skill）：

| | 代码量 | token | 成本 | 耗时 |
|---|--:|--:|--:|--:|
| ponytail | **-54%** | -22% | -20% | -27% |

装法：

```powershell
npx skills add DietrichGebert/ponytail -g
```

**我的看法**：如果你经常吐槽"这活儿我自己写十分钟就完了"，这个最值得先装。

---

## 2. 省上下文：把噪音挡在模型外面

Agent 的上下文窗口是稀缺资源，真正吃掉它的大头是**命令输出**（`ls`、`git diff`、测试日志）和**历史消息**。下面三个都在解决这件事，选一个就够，别叠着装。

### rtk

**仓库**：<https://github.com/rtk-ai/rtk>

Rust 单二进制，拦截 shell 命令并把输出压缩后再喂给 agent。支持 100+ 命令，官方说开销 <10ms，最高能砍掉 90% 的 bash 输出。

压缩策略是按命令定制的，比如：

| 命令 | 处理方式 |
|---|---|
| `ls` / `tree` | 目录树 + 文件计数，不再一行一个文件 |
| `cat` / 读文件 | 优先给签名和结构，而不是全文 |
| `grep` / `rg` | 截断超长行，按文件分组 |
| `git diff` | 减少上下文行，去掉头部噪音 |
| `cargo test` / `npm test` | 只留失败项，通过的折叠成计数 |

Windows 安装：

```powershell
winget install rtk-ai.rtk
rtk init -g                     # Claude Code / Copilot 默认
rtk init -g --codex             # OpenAI Codex
rtk init -g --agent cursor      # Cursor
```

> ⚠️ crates.io 上有一个**同名但完全无关**的包（Rust Type Kit）。如果你 `cargo install` 装完发现 `rtk gain` 跑不通，就是装错了。

### headroom

**仓库**：<https://github.com/headroomlabs-ai/headroom>

压缩"agent 读到的一切"——工具输出、日志、RAG 片段、文件内容、对话历史。**压缩在全本机完成，不上传任何 prompt 或文件内容**，这点对内部代码比较友好。

几种用法：

- **库**：Python / TypeScript 里直接调 `compress(messages)`
- **代理**：`headroom proxy --port 8787`，零代码改动，任何语言都行
- **包一层**：`headroom wrap claude`（也支持 codex / cursor / aider / opencode 等十几个）
- **MCP 服务**：提供 `headroom_compress` / `headroom_retrieve` / `headroom_stats`
- **跨 agent 记忆**：Claude / Codex / Gemini / Grok 共用一个记忆库，自动去重
- **`headroom learn`**：挖失败会话，把教训写成 `CLAUDE.local.md`（默认，已 gitignore）或 `CLAUDE.md` / `AGENTS.md`

可逆（CCR）：原文缓存在本地，需要时能取回。

```powershell
pip install "headroom-ai[all]"    # 会装 headroom CLI
```

### lean-ctx

**仓库**：<https://github.com/yvgude/lean-ctx>

定位是"AI 价值门"（Value Gate）：**理解**任务 → **路由**对的上下文 → **压缩**再发送 → **追踪**成本和效果。本地优先，零配置可用。

五个能力对应解决的痛点：

| 痛点 | 它的做法 |
|---|---|
| 重复读同一个文件，每次都重发全文 | 缓存复用，命中就返回紧凑引用 |
| 命令输出里有大量重复噪音 | 按命令定制压缩，保留关键信息 |
| 每轮都重发整段历史 | 代理层逐请求压缩，**对 prompt cache 友好** |
| 换个对话上下文就清零 | 会话记忆跨对话保留 |
| 不知道 context 花在哪 | 实时面板 + 预算控制；还能跑 Shadow Mode 对比基线 |

```powershell
curl -fsSL https://leanctx.com/install.sh | sh
# 或 npm install -g lean-ctx-bin
```

> Windows 上目前主要靠**源码构建**（`./install.ps1`，需要 Rust 环境）。嫌麻烦的话 Windows 用户优先选 rtk 或 headroom。

---

## 3. 看懂代码库：代码图谱类

大仓库里最常见的一幕：你问一个跨文件的问题，agent 开始 `grep` → `read` → 再 `grep`，八轮之后上下文塞满了，答案还没出来。

这两个都是**预建代码知识图谱**再通过 MCP 暴露给 agent，让它一次调用就拿到该看的代码。**二选一**，功能重叠。

### codegraph

**仓库**：<https://github.com/colbymchenry/codegraph>

给 agent 提供"符号 + 调用边 + 依赖"的完整图谱，内核用 Rust 写的，**100% 本地**。

Windows 安装（二选一）：

```powershell
irm https://raw.githubusercontent.com/colbymchenry/codegraph/main/install.ps1 | iex
# 或已有 Node.js：
npm i -g @colbymchenry/codegraph
```

然后：

```powershell
codegraph install --yes --init    # 自动检测并配置 agent，同时索引当前项目
codegraph status                  # 看索引状态，有 Pending sync 会列出具体文件
codegraph upgrade                 # 升级（会自动识别安装方式）
```

支持 Claude Code、Cursor、Codex CLI、opencode、Gemini CLI、Copilot（VS Code / JetBrains）、Antigravity、Kiro 等。

> 注意：`install` **只配置 agent，不索引代码**，每个项目要自己跑一次 `codegraph init`。

### codebase-memory-mcp

**仓库**：<https://github.com/DeusData/codebase-memory-mcp>

用 tree-sitter 做 AST 分析（内置 162 种语言的语法），再加一层混合 LSP 做类型解析。速度是它的卖点：Linux 内核（2800 万行、7.5 万文件）3 分钟建完索引。

官方给的 token 对比：5 个结构查询约 **3,400 token**，而逐个文件搜索约 **412,000 token**（约 120 倍差距）。提供 17 个 MCP 工具（搜索、调用链追踪、架构、影响面分析、Cypher 查询、死代码检测、跨服务 HTTP 关联等），自带 `localhost:9749` 的 3D 图谱可视化。

Windows 安装：

```powershell
# 先从 Releases 下载 codebase-memory-mcp-windows-amd64.zip
Expand-Archive codebase-memory-mcp-windows-amd64.zip -DestinationPath .
Unblock-File .\install.ps1
.\install.ps1
```

> ⚠️ 这个工具**会读你的代码库，并写入 agent 的配置文件**（这是它的设计目的）。README 自己也提醒：介意的话先审计再加脚本。装完需要重启 agent。

### open-code-review

**仓库**：<https://github.com/alibaba/open-code-review>

前面两个是"让 agent 看懂代码"，这个是**让 agent 评审代码**。它原本是阿里内部的官方 AI 代码评审助手，服务了两年多后开源。

思路和"给 Claude Code 写个 review skill"不同：它是**确定性工程 + agent 混合**，读 git diff，把改动文件交给可配置的 LLM（带 tool-use），输出**行级精确**的结构化评审意见。README 点名了纯自然语言方案的三个毛病：大改动会偷懒只审部分文件、报的问题定位漂移、质量随 prompt 波动。

官方还放了 AACR-Bench（50 个热门开源仓库、200 个真实 PR、10 种语言），并与 Claude Code 对比，声称同模型下 precision / F1 明显更高、token 消耗更低。

```powershell
npm install -g @alibaba-group/open-code-review

ocr config provider                 # 选内置 provider 或加自定义的
ocr config model                    # 选模型
ocr review                          # 评审当前改动
ocr review --from main --to feature-branch
ocr review --commit abc123
ocr scan --path internal/agent      # 全仓扫描或指定目录
ocr review --format json --output result.json   # 方便接 CI
```

**适合谁**：想给团队评审加一层机器兜底，或者接进 CI 门禁。个人的话，`ocr review` 提交前自己跑一遍挺香。

---

## 4. 干活工具：文档 / 数据库 / 可视化

### OfficeCLI

**仓库**：<https://github.com/iOfficeAI/OfficeCLI>

让 agent 直接操作 Word / Excel / PowerPoint。**单二进制、不需要装 Office、无依赖**。

它有两个关键能力：一是生成 `.docx` / `.xlsx` / `.pptx`；二是内置 HTML 渲染引擎，把文档渲染成 HTML 或 PNG —— 这等于给了 agent "眼睛"，能自己看渲染结果再改。

装法很省事，把这条丢给 agent 就行：

```text
curl -fsSL https://officecli.ai/SKILL.md
```

或者自己装二进制后跑：

```bash
officecli install   # 拷贝二进制到 PATH，并把 officecli skill 装进检测到的所有 agent
```

README 声称检测 Claude Code、Cursor、Windsurf、GitHub Copilot 等。

**典型场景**：周报、需求文档、数据统计表、汇报 PPT —— 尤其是"数据在代码/数据库里，要交个表格文档"这种活。

### dbx

**仓库**：<https://github.com/t8y2/dbx>

25MB 单文件数据库客户端，支持 100+ 种数据库（MySQL、PostgreSQL、SQLite、Redis、MongoDB、DuckDB、ClickHouse、SQL Server、Oracle、Elasticsearch、Qdrant、Milvus……）。桌面端 + Docker + Web + CLI，同一套连接配置。

相比 DBeaver 不需要 Java 运行时，相比 TablePlus 不限功能。亮点是 AI 集成：

- **内置 AI SQL 助手**：选中表用自然语言描述需求 → 出 SQL，还带安全校验（接 Claude / OpenAI / Ollama 本地模型）
- **MCP Server**：Claude Code / Cursor / Windsurf 可以通过你**已配置好的连接**直接查库

```powershell
winget install t8y2.dbx
# 或 scoop bucket add dbx https://github.com/t8y2/scoop-bucket && scoop install dbx
# 或 brew install --cask dbx

npx @dbx-app/mcp-server     # MCP 用法
```

### archify

**仓库**：<https://github.com/tt-a1i/archify>

把"你想讲清楚的东西"变成**可交互的 HTML 图**。起点可以是一个想法、一个问题、一个方案，也可以让 agent 直接读仓库生成有出处依据的架构图。

产出是能点开、能探索、能分享的 HTML，比静态截图强很多。README 里的例子包括系统架构图（"浏览器调 API，API 查 Redis，缓存 miss 就去 PostgreSQL 查并回填缓存"）和行程规划、学习地图这类非技术内容。

```powershell
npx skills add tt-a1i/archify -g
```

用法就是直接说人话，然后继续追加："加上鉴权"、"高亮缓存 miss 那条路径"、"换成浅色主题"。支持 Claude Code、Cursor、Codex CLI、OpenCode。

**典型场景**：给同事/领导讲清一个系统怎么跑的 —— 比在群里贴一段文字描述有效得多。

---

## 5. herdr：管一堆 Agent 的终端运行时

**仓库**：<https://github.com/herdrdev/herdr>

如果你开始"同时开三四个 agent 干活"，就会发现终端不够用、SSH 断了活就没了、不知道哪个卡住了。herdr 就是干这个的：**coding agent 的运行时**。

- **断连不停活**：关掉客户端或 SSH 掉线，terminal 继续跑在后台服务里；重连 `herdr` 就回来了
- **多机一个窗口**：本地任务 + 保存的 SSH 机器放一起，统一的 agent 列表，独立重连
- **一眼看出谁卡住了**：每个 pane 标注 working / blocked / idle，agent 停下来等你回答时会明确提示
- **agent 原生**：agent 自己能通过 CLI 和 socket API 驱动 herdr —— 开 pane、互相发消息、等另一个 agent 真正阻塞
- **不替换你的工具**：Claude Code、Codex、Cursor、opencode 照常用，herdr 只管它们的终端
- 键盘（tmux 风格前缀键）和鼠标（点选、拖拽、分屏）都是一等公民
- 单个 Rust 二进制，不是 Electron

Windows 安装：

```powershell
powershell -ExecutionPolicy Bypass -c "irm https://herdr.dev/install.ps1 | iex"
# 或 brew install herdr / mise use -g herdr
```

**适合谁**：已经在用 tmux 管 agent 的人，或者开始跑长任务、多任务的人。个人用一个 agent 的话暂时用不上。

---

## 6. 怎么开始：建议的安装顺序

别一次全装 —— 装太多会互相干扰（尤其是第 2 类，上下文压缩工具选一个就好）。建议按这个顺序：

1. **第一周**：装一个输出类 skill，[caveman](#caveman) 或 [ponytail](#ponytail)。零风险，立刻能感受到区别。
2. **第二周**：如果你是重度终端用户，加 [rtk](#rtk)。命令输出变短的效果最直观。
3. **大仓库**：接 [codegraph](#codegraph) 或 [codebase-memory-mcp](#codebase-memory-mcp)（选一个），感受"一次调用拿到代码"和"grep 八轮"的差别。
4. **按需**：团队有评审流程 → [open-code-review](#open-code-review)；天天写文档表格 → [OfficeCLI](#officecli)；要和数据库打交道 → [dbx](#dbx)。

**推荐验证方式**：装前后各找同一个真实任务跑一遍，对比一下 token 消耗和你的体感。README 上的数字是别人的机器、别人的仓库，你的场景未必一样。

---

## 7. 注意事项（重要）

1. **合规先行**：这些是第三方开源项目，多数由个人或小团队维护。装之前确认公司对代码外发、第三方工具接入的规定。
2. **配置类工具会改你的 agent 配置**：codegraph、codebase-memory-mcp、rtk、OfficeCLI 的安装脚本都会自动检测并写入 agent 配置（`settings.json`、`.mcp.json`、`CLAUDE.md`、hooks 等）。装之前看一眼脚本内容，装完 `git diff` 一下项目里的配置文件。
3. **不要把网关 token 贴进命令行**：涉及 `ANTHROPIC_AUTH_TOKEN` 之类的操作，统一走配置文件（见 [`claude使用教程.md`](./claude使用教程.md) 第 3 节），别在 shell 历史里留痕。
4. **版本行为会变**：这类项目迭代极快，命令和默认行为随时可能调整。本文命令基于我查看 README 时的版本（2026-09），遇到不一致先看官方 README。
5. **上下文压缩工具别叠加**：rtk / headroom / lean-ctx 三个同时上，可能互相打架或者造成难以排查的输出缺失。选一个，跑稳了再说。

---

## 附：仓库地址一览

| 项目 | 地址 | 类型 |
|---|---|---|
| archify | <https://github.com/tt-a1i/archify> | Skill |
| open-code-review | <https://github.com/alibaba/open-code-review> | CLI |
| i-have-adhd | <https://github.com/ayghri/i-have-adhd> | Skill |
| headroom | <https://github.com/headroomlabs-ai/headroom> | 代理 / MCP |
| herdr | <https://github.com/herdrdev/herdr> | 终端运行时 |
| ponytail | <https://github.com/DietrichGebert/ponytail> | Skill |
| OfficeCLI | <https://github.com/iOfficeAI/OfficeCLI> | CLI + Skill |
| lean-ctx | <https://github.com/yvgude/lean-ctx> | 代理 |
| caveman | <https://github.com/JuliusBrussee/caveman> | Skill / 代理 |
| codebase-memory-mcp | <https://github.com/DeusData/codebase-memory-mcp> | MCP |
| dbx | <https://github.com/t8y2/dbx> | 桌面 + MCP |
| codegraph | <https://github.com/colbymchenry/codegraph> | MCP |
| rtk | <https://github.com/rtk-ai/rtk> | CLI 代理 |
