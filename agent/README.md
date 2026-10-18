# AI 编程 Agent 技能与工具推荐

这是一份**给同事的推荐清单**，收集了一批能实实在在提升日常开发体验的开源技能（Skill）、MCP 服务和命令行工具。每个工具一篇独立文档，放在 [`tool/`](./tool)。

## 目录

```text
agent/
├─ README.md          本文件：工具清单、选型建议、安装顺序
├─ wsl.md             WSL 本体配置：镜像网络、代理、interop、排错（先看这篇）
├─ windows.md         Windows 原生环境的差异：PowerShell 工具、路径、Hook 脚本坑
├─ tool/              外部工具文档：每个工具一篇
│   ├─ caveman.md           省 token：让 agent 少说废话（Skill）
│   ├─ ponytail.md          省 token：让 agent 别过度设计（Skill）
│   ├─ mattpocock-skills.md 工程流程：追问需求 / 拆票 / TDD / 评审（Skill 合集）
│   ├─ codegraph.md         代码图谱（MCP）
│   ├─ open-code-review.md  AI 代码评审（CLI + 插件）
│   ├─ officecli.md         Word / Excel / PPT 生成（CLI + Skill）
│   ├─ dbx.md               数据库客户端 + MCP
│   ├─ archify.md           可交互架构图（Skill）
│   ├─ diagram-design.md    有设计感的架构图 / 流程图（Skill）
│   └─ herdr.md             多 agent 终端运行时
├─ claude/            Claude Code 文档：README.md（上手指南）、rule / skill / mcp / hook / command / subagent 使用指南.md
├─ codex/             Codex CLI 文档：README.md 及 hook / mcp / plugin / skill / subagent 五篇
└─ pi/                Pi 文档：README.md、配置-运行环境.md、配置-模型接入.md、
                      配置-权限与规则.md、Extension使用指南.md、skill使用指导.md
```

**阅读前提**：三份文档都按 **Linux / WSL** 写，命令是 bash、路径是 `~`。还没装 WSL 先看 [`wsl.md`](./wsl.md)；必须在 Windows 原生环境跑的情况（.NET / MSBuild / COM / Windows 服务）看 [`windows.md`](./windows.md)。
装哪个工具：Claude Code 看 [`claude/README.md`](./claude/README.md)；Codex CLI 看 [`codex/README.md`](./codex/README.md)；Pi 看 [`pi/README.md`](./pi/README.md)。

**说明几点：**

- 下面所有工具都是**第三方开源项目**，不是 Anthropic / 官方出品。装之前建议先确认公司合规要求。
- 文中引用的百分比、倍数等效果数据**均为各项目 README 自述**，我没有逐一本地复现，当参考值看待。
- 安装方式统一按 Linux / WSL 整理；只有 Windows 才有的做法会在 [`windows.md`](./windows.md) 里说明。

---

## 工具一览

| 项目 | 一句话 | 类型 | 地址 |
|---|---|---|---|
| [caveman](./tool/caveman.md) | Agent 说话太啰嗦，废话烧 token | Skill | <https://github.com/JuliusBrussee/caveman> |
| [ponytail](./tool/ponytail.md) | Agent 过度设计，写一堆没用的代码 | Skill | <https://github.com/DietrichGebert/ponytail> |
| [mattpocock-skills](./tool/mattpocock-skills.md) | 想动手前先问清楚，拆票 / TDD / 评审都有规矩 | Skill 合集 | <https://github.com/mattpocock/skills> |
| [codegraph](./tool/codegraph.md) | 大仓库里 agent 找代码靠瞎 grep | MCP | <https://github.com/colbymchenry/codegraph> |
| [open-code-review](./tool/open-code-review.md) | 代码评审靠人肉，想加一层 AI 兜底 | CLI + 插件 | <https://github.com/alibaba/open-code-review> |
| [officecli](./tool/officecli.md) | 让 agent 帮你写 Word / Excel / PPT | CLI + Skill | <https://github.com/iOfficeAI/OfficeCLI> |
| [dbx](./tool/dbx.md) | 轻量数据库客户端 + 让 agent 直连查库 | 桌面 + MCP | <https://github.com/t8y2/dbx> |
| [archify](./tool/archify.md) | 把方案 / 架构图讲清楚给同事看 | Skill | <https://github.com/tt-a1i/archify> |
| [diagram-design](./tool/diagram-design.md) | 图一画就是 AI 味，进不了文档和 PPT | Skill + 命令 | <https://github.com/cathrynlavery/diagram-design> |
| [herdr](./tool/herdr.md) | 开一堆 agent 跑长任务，窗口管不过来 | 终端运行时 | <https://github.com/herdrdev/herdr> |

---

## 三个 agent 怎么接（一张表）

每篇工具文档都有“**没有它之前** / **优点缺点** / **三个 agent 怎么接**”三节，这里是索引。凭据和配置都在 WSL 的家目录下，不在 Windows。

| 工具 | 形态 | Pi | Claude Code | Codex |
|---|---|---|---|---|
| caveman、ponytail、archify | Skill | `~/.pi/agent/skills/<名>/`，`/skill:<名>` | `~/.claude/skills/<名>/`，`/<名>` | `~/.codex/skills/<名>/` 或项目 `.agents/skills/`，`$<名>` |
| mattpocock-skills（27 个） | Skill 合集 | `npx skills add mattpocock/skills -a pi`，或 `pi install git:github.com/mattpocock/skills` | `claude plugin install mattpocock-skills@claude-plugins-official` | `codex plugin marketplace add mattpocock/skills` → `codex plugin add mattpocock-skills@mattpocock` |
| diagram-design | Skill + 命令 | `pi install https://github.com/cathrynlavery/diagram-design` | `/plugin marketplace add …` | `codex plugin marketplace add …` |
| codegraph | MCP（stdio） | `pi mcp add codegraph -- codegraph serve --mcp` | `claude mcp add --scope user codegraph -- codegraph serve --mcp` | `codex mcp add codegraph -- codegraph serve --mcp` |
| dbx | MCP（HTTP） | `pi mcp add dbx --url http://localhost:5225/mcp --bearer-token-env-var DBX_MCP_TOKEN` | 项目 `.mcp.json`（`type: http` + headers） | `codex mcp add dbx --url ...` + `bearer_token_env_var` |
| open-code-review、officecli | CLI（+ 可选 Skill） | 让 agent 自己跑 CLI，`AGENTS.md` 里写一句 | 同上；有官方插件 | 同上；有官方 skill |
| herdr | 终端运行时 | `herdr integration install pi` | `herdr integration install claude` | `herdr integration install codex` |

三条通用规矩：

1. **Skill 目录三家互不相通**。Claude Code 不读 `.agents/skills/`，Pi 和 Codex 读。同一份规则三家都要，就各放一份。
2. **能命令行就别装插件**。CLI 类工具（ocr、officecli）只需要 shell 权限，在 `AGENTS.md` 里写一行用法比往四个目录装 skill 稳。
3. **MCP 工具面要收窄**。Pi 默认 `exposure: "codemode"`（按需搜索，不全量进上下文）；Codex 用 `enabled_tools` 白名单；Claude Code 用 `claude mcp list` 确认连接。

---

## 怎么开始：建议的安装顺序

别一次全装 —— 装太多会互相干扰。建议按这个顺序：

1. **第一周**：装一个输出类 skill，[caveman](./tool/caveman.md) 或 [ponytail](./tool/ponytail.md)。零风险，立刻能感受到区别。
2. **大仓库**：接 [codegraph](./tool/codegraph.md)，感受"一次调用拿到代码"和"grep 八轮"的差别。
3. **按需**：团队有评审流程 → [open-code-review](./tool/open-code-review.md)；天天写文档表格 → [officecli](./tool/officecli.md)；要和数据库打交道 → [dbx](./tool/dbx.md)。
4. **流程类**：[mattpocock-skills](./tool/mattpocock-skills.md) 别一次全装，先只留 `grill-me` / `diagnosing-bugs` 两个，用顺了再加。

**推荐验证方式**：装前后各找同一个真实任务跑一遍，对比一下 token 消耗和你的体感。README 上的数字是别人的机器、别人的仓库，你的场景未必一样。

---

## 注意事项（重要）

1. **合规先行**：这些是第三方开源项目，多数由个人或小团队维护。装之前确认公司对代码外发、第三方工具接入的规定。
2. **配置类工具会改你的 agent 配置**：codegraph、OfficeCLI 的安装脚本都会自动检测并写入 agent 配置（`settings.json`、`.mcp.json`、`CLAUDE.md`、hooks 等）。装之前看一眼脚本内容，装完 `git diff` 一下项目里的配置文件。
3. **不要把网关 token 贴进命令行**：涉及 `ANTHROPIC_AUTH_TOKEN` 之类的操作，统一走配置文件（见 [`claude/README.md`](./claude/README.md) 第 5.1 节），别在 shell 历史里留痕。
4. **代码和工具都装在 Linux 侧**：`npm install -g`、`~/.claude`、`~/.pi/agent` 都在 WSL 里，别在 `/mnt/c` 上折腾；跨文件系统的读写和文件监听问题不值得。
5. **版本行为会变**：这类项目迭代极快，命令和默认行为随时可能调整。本文命令基于我查看 README 时的版本（2026-09），遇到不一致先看官方 README。
