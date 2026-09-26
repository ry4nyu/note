# AI 编程 Agent 技能与工具推荐

这是一份**给同事的推荐清单**，收集了一批能实实在在提升日常开发体验的开源技能（Skill）、MCP 服务和命令行工具。

## 目录

```text
agent/
├─ README.md          本文件：跨工具的 Skill / MCP / CLI 推荐清单
├─ claude/            Claude Code 文档：README.md（上手指南）、hook / rule / skill / mcp / command / subagent 使用指南.md
├─ codex/             Codex CLI 文档：README.md 及 hook / mcp / plugin / skill / subagent 五篇
└─ pi/                Pi 文档：README.md、Extension使用指南.md、skill使用指导.md
```

前置阅读：还没装 Claude Code，看 [`claude/README.md`](./claude/README.md)；用 Codex CLI 看 [`codex/README.md`](./codex/README.md)；用 Pi 看 [`pi/README.md`](./pi/README.md)。

**说明几点：**

- 下面所有工具都是**第三方开源项目**，不是 Anthropic / 官方出品。装之前建议先确认公司合规要求。
- 文中引用的百分比、倍数等效果数据**均为各项目 README 自述**，我没有逐一本地复现，当参考值看待。
- 安装方式我按 Windows 环境整理，非 Windows 的会单独标注。

---

## 0. 30 秒选型

| 你想解决的问题 | 装这个 | 类型 |
|---|---|---|
| Agent 说话太啰嗦，废话烧 token | [caveman](#caveman) | Skill |
| Agent 过度设计，写一堆没用的代码 | [ponytail](#ponytail) | Skill |
| 大仓库里 agent 找代码靠瞎 grep | [codegraph](#codegraph) | MCP |
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

## 2. 看懂代码库：代码图谱类

大仓库里最常见的一幕：你问一个跨文件的问题，agent 开始 `grep` → `read` → 再 `grep`，八轮之后上下文塞满了，答案还没出来。

它把代码库**预建成知识图谱**再通过 MCP 暴露给 agent，让它一次调用就拿到该看的代码。

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

### open-code-review

**仓库**：<https://github.com/alibaba/open-code-review>

前面是"让 agent 看懂代码"，这个是**让 agent 评审代码**。它原本是阿里内部的官方 AI 代码评审助手，服务了两年多后开源。

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

## 3. 干活工具：文档 / 数据库 / 可视化

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

## 4. herdr：管一堆 Agent 的终端运行时

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

## 5. 怎么开始：建议的安装顺序

别一次全装 —— 装太多会互相干扰。建议按这个顺序：

1. **第一周**：装一个输出类 skill，[caveman](#caveman) 或 [ponytail](#ponytail)。零风险，立刻能感受到区别。
2. **大仓库**：接 [codegraph](#codegraph)，感受"一次调用拿到代码"和"grep 八轮"的差别。
3. **按需**：团队有评审流程 → [open-code-review](#open-code-review)；天天写文档表格 → [OfficeCLI](#officecli)；要和数据库打交道 → [dbx](#dbx)。

**推荐验证方式**：装前后各找同一个真实任务跑一遍，对比一下 token 消耗和你的体感。README 上的数字是别人的机器、别人的仓库，你的场景未必一样。

---

## 6. 注意事项（重要）

1. **合规先行**：这些是第三方开源项目，多数由个人或小团队维护。装之前确认公司对代码外发、第三方工具接入的规定。
2. **配置类工具会改你的 agent 配置**：codegraph、OfficeCLI 的安装脚本都会自动检测并写入 agent 配置（`settings.json`、`.mcp.json`、`CLAUDE.md`、hooks 等）。装之前看一眼脚本内容，装完 `git diff` 一下项目里的配置文件。
3. **不要把网关 token 贴进命令行**：涉及 `ANTHROPIC_AUTH_TOKEN` 之类的操作，统一走配置文件（见 [`claude/README.md`](./claude/README.md) 第 5.1 节），别在 shell 历史里留痕。
4. **版本行为会变**：这类项目迭代极快，命令和默认行为随时可能调整。本文命令基于我查看 README 时的版本（2026-09），遇到不一致先看官方 README。

---

## 附：仓库地址一览

| 项目 | 地址 | 类型 |
|---|---|---|
| archify | <https://github.com/tt-a1i/archify> | Skill |
| open-code-review | <https://github.com/alibaba/open-code-review> | CLI |
| herdr | <https://github.com/herdrdev/herdr> | 终端运行时 |
| ponytail | <https://github.com/DietrichGebert/ponytail> | Skill |
| OfficeCLI | <https://github.com/iOfficeAI/OfficeCLI> | CLI + Skill |
| caveman | <https://github.com/JuliusBrussee/caveman> | Skill / 代理 |
| dbx | <https://github.com/t8y2/dbx> | 桌面 + MCP |
| codegraph | <https://github.com/colbymchenry/codegraph> | MCP |
