# Pi 上手指南

给开发人员的 Pi 使用说明。基于 Pi 1.3.x（Windows 原生 + PowerShell、WSL 两种跑法都覆盖），命令和默认值随版本变化，遇到不一致先执行 `pi --help`。

Pi 是 Earendil Works 做的终端编程 Agent。和 Claude Code、Codex 相比，它的取舍是"**最小内核 + 一切靠扩展**"：内置工具只有 `read` / `bash` / `powershell` / `edit` / `write` / `grep` / `find` / `ls`，其余能力（规划模式、代码图谱、技能）都通过扩展、包和 Skill 挂上去。

**本文只讲怎么用。配置、权限、环境选择都在下面的独立文档里，本文只做引用。**

---

## 文档地图

| 你想解决的问题 | 看这篇 |
|---|---|
| Pi 跑在 Windows 原生还是 WSL？用 PowerShell 还是 Bash？ | [`配置-运行环境.md`](./配置-运行环境.md) |
| 怎么接公司网关、换模型、修"聊几句就压缩" | [`配置-模型接入.md`](./配置-模型接入.md) |
| 权限为什么没有审批弹窗、工具怎么限制、`AGENTS.md` 怎么写 | [`配置-权限与规则.md`](./配置-权限与规则.md) |
| WSL 本体怎么装（镜像网络、代理、interop、排错） | [`wsl.md`](./wsl.md) |
| 想写扩展 | [`Extension使用指南.md`](./Extension使用指南.md) |
| 想用/写 Skill | [`skill使用指导.md`](./skill使用指导.md) |

---

## 1. 快速开始

```powershell
# 1) 安装（需要 Node.js 22.19+）
npm install -g --ignore-scripts @earendil-works/pi-coding-agent

# 2) 选运行环境（见第 2 节，花两分钟值得）
#    正在做 Windows 原生应用 → Windows 原生 + PowerShell
#    其它项目            → WSL

# 3) 配模型接入：/login 官方账号，或写 ~/.pi/agent/models.json 接自建网关
#    详见 配置-模型接入.md

# 4) 进项目目录启动
cd C:\Users\你的名字\IdeaProjects\你的项目
pi
```

启动时如果目录里有需要信任的资源（`.pi/` 下的配置/扩展、`.agents/skills/` 等），Pi 会先问一次是否**信任这个工作目录**，确认后就能直接提问。信任的含义和风险见 [`配置-权限与规则.md`](./配置-权限与规则.md) 第 1 节。

> 本机已经配好公司网关，直接 `pi` 就能用，不用再 `/login`。

---

## 2. 先选运行环境

这是唯一一个"选错了后面一直难受"的决定：

| 你在做什么 | 用什么 |
|---|---|
| **Windows 原生应用开发**（.NET / WPF / WinForms、MSBuild、COM、注册表、Windows 服务、驱动、Office 插件、Windows SDK 工具链） | **Windows 原生 + `powershell` 工具** |
| 其它项目（Java / Go / Node / Python / 前端 / 部署在 Linux 的服务） | **WSL** |
| 只读代码、评审、写文档 | Windows 原生最省事 |

一句话：**依赖 Windows 工具链的留在原生，其余全部进 WSL。**

两种方案的工具、配置位置、快捷键差异、常见坑，全部在 [`配置-运行环境.md`](./配置-运行环境.md)。记住最关键的一条：**配置和会话不跨环境共享**，Windows 上配好的模型在 WSL 里要重配一遍。

---

## 3. 基本用法

### 3.1 启动

```powershell
cd C:\path\to\your\project     # 必须先切目录：Pi 按它发现文件、AGENTS.md 和配置，并按它给会话分组
pi                             # 进入交互界面
```

常用启动方式：

```powershell
pi "先介绍一下这个项目结构，不要改文件"   # 带初始问题启动
pi --continue                             # 继续当前目录上一次会话（-c）
pi --resume                               # 从历史会话列表里挑一个（-r）
pi -p "解释一下 pom.xml"                  # 非交互模式，跑完就退出，适合脚本
pi -p --mode json "列出所有 TODO" > out.jsonl   # 结构化输出，喂给别的程序
pi --name "重构登录模块"                  # 给本次会话起名，方便以后 /resume 找到
pi --model sonnet:high "帮我分析这个并发问题"    # 指定模型 + 思考等级
pi --tools read,grep,find,ls -p "评审 src/ 下的代码"   # 只读模式，物理上改不了文件
pi update                                 # 升级 Pi 和模型目录
```

其它值得知道的开关：

| 参数 | 作用 |
|---|---|
| `-p, --print` | 非交互：处理完就退出 |
| `--mode text\|json\|rpc` | 输出模式，`rpc` 可以用程序双向驱动 |
| `-c` / `-r` / `--session <id>` / `--fork <id>` | 继续 / 选会话 / 指定会话 / 从某会话分叉 |
| `--no-session` | 本次不存会话（临时用） |
| `--tools` / `--exclude-tools` / `--no-tools` | 工具白名单 / 黑名单 / 全关 |
| `--thinking off\|minimal\|low\|medium\|high\|xhigh\|max` | 启动时的思考等级 |
| `--models "anthropic/*,deepseek-*"` | 限定 `Ctrl+P` 循环的模型范围 |
| `-a, --approve` / `-na, --no-approve` | 本次信任 / 不信任项目本地资源 |
| `--no-context-files` | 不加载 `AGENTS.md` / `CLAUDE.md` |
| `-e <path>` / `--skill <path>` | 临时加载扩展 / Skill |
| `--tui-mode fullscreen` | 全屏模式（编辑器和状态栏固定） |
| `--offline` | 关掉启动时的联网动作 |

### 3.2 交互界面

界面分三块：上方是对话记录（transcript），中间是输入编辑器，底部状态栏显示当前目录、会话、模型、上下文用量和累计费用。**编辑器边框的颜色/样式表示当前思考等级**。

| 操作 | 效果 |
|---|---|
| 直接输入 + `Enter` | 提交提问或指令 |
| `Shift+Enter` 或 `Ctrl+J` | 输入内换行（不发送） |
| `Ctrl+G` | 用外部编辑器写长提示词 |
| `@文件名` | 搜索并引用文件，`Tab` 补全路径 |
| `Alt+V` | 粘贴图片（Windows / WSL 都是这个键） |
| `!命令` | 执行 shell 命令，**输出进上下文**（如 `!git status`） |
| `!!命令` | 执行 shell 命令，**输出不进上下文**（如 `!!npm test`） |
| `/命令` | 斜杠命令，输入 `/` 弹出可搜索菜单 |
| `Esc` | 打断它正在做的事 |
| `Ctrl+O` | 展开 / 折叠工具调用输出 |
| `Ctrl+T` | 显示 / 隐藏 thinking 块 |
| `Ctrl+L` | 打开模型选择器 |
| `Ctrl+P` | 循环切换模型（反向 `Alt+P`） |
| `Shift+Tab` | 循环切换思考等级（**不是权限模式**） |
| `Ctrl+X` | 复制最后一条回复 |
| `Ctrl+C` | 第一次清空输入框，第二次退出 |
| `Ctrl+D` | 退出（输入框为空时） |

**任务执行中还能再说话**，这点比"等它跑完"实用：

| 想要的效果 | 操作 |
|---|---|
| 调整当前任务方向 | 输入内容按 `Enter`（等当前工具调用跑完就生效） |
| 追加后续任务 | Windows / WSL：`Ctrl+Q`（其它平台 `Alt+Enter`） |
| 把排队的消息收回输入框 | Windows / WSL：`Alt+Q`（其它平台 `Alt+Up`） |
| 中止当前任务 | `Esc` |

实际生效的键以 `/hotkeys` 为准。

### 3.3 斜杠命令

输入 `/` 打开可搜索菜单，**菜单里显示的就是当前会话实际加载的全部命令**（扩展、Skill、提示词模板都会往里加）。

| 命令 | 作用 |
|---|---|
| `/model [provider/model]` | 选择模型，`Ctrl+S` 存为默认 |
| `/thinking [级别]` | 设置思考等级，`Ctrl+S` 存为启动默认 |
| `/scoped-models` | 配置 `Ctrl+P` 循环包含哪些模型 |
| `/login` / `/logout` | 添加 / 移除 provider 凭据 |
| `/new` | 开新会话 |
| `/resume` | 切到别的历史会话 |
| `/name [名字]` | 给当前会话命名 |
| `/session` | 看当前会话的文件、ID、消息数、token 用量和费用 |
| `/tree` | 会话树导航（回看、跳到某个节点） |
| `/fork` / `/clone` | 从某条历史消息另起会话 / 复制当前会话 |
| `/compact [额外要求]` | 压缩上下文，可带自定义总结要求 |
| `/export [路径]` | 导出成 HTML 或 JSONL |
| `/share` | 上传会话并返回查看链接（导出前先自己看一遍内容） |
| `/trust` | 保存对当前项目的信任决定，后续进程不再问 |
| `/reload` | 重新加载按键、扩展、Skill、模板、主题和上下文文件 |
| `/hotkeys` | 查看当前会话实际生效的快捷键 |
| `/settings` | 打开设置界面 |
| `/changelog` / `/debug` / `/quit` | 更新日志 / 写 `pi-debug.log` / 退出 |

---

## 4. 让 Pi 遵守项目规矩

在项目根目录放一个 `AGENTS.md`，写清技术栈、构建测试命令、代码规范、禁止事项 —— 这是减少跑偏最有效的手段。

文件放哪、认哪些文件名、模板长什么样、`CLAUDE.md` 能不能直接用：[`配置-权限与规则.md`](./配置-权限与规则.md) 第 3 节。

另外两条和规则相关的行为，也记在那里：

- **上下文文件不受项目信任保护** —— 看别人的仓库时，它的 `AGENTS.md` 已经被读进上下文了，当不可信输入对待。
- **Pi 默认不问你每一步** —— 限制能力靠 `--tools` / `defaultTools`，或把 Pi 放进容器。

---

## 5. 把它用好的几个习惯

1. **说清楚"在哪、做什么、期望什么"**
   - 差：`登录有问题，修一下`
   - 好：`src/main/java/.../AuthService.java 的 login 方法，用户输错密码时返回的是 500，应该是 401 加错误提示，帮我改`

2. **复杂需求先让它只读调研**

   ```powershell
   pi --tools read,grep,find,ls "读一下 order 模块，告诉我如果要加「超时未支付自动取消」该改哪些地方"
   ```

   也可以加载 plan-mode 扩展，用 `pi --plan` 直接进只读规划模式。

3. **一次只做一件事**，别一口气丢十个需求

4. **用 git 兜底**：动手前确认工作区干净，搞砸了 `git checkout .`。`/tree`、`/fork`、`/clone` 管的是对话，**不回滚文件**

5. **让它自己验证**：`改完跑一下相关单测` / `启动服务确认接口能通`

6. **聊长了 `/compact`，换话题 `/new`**；重要的会话先 `/name`

7. **问它"你是什么模型"没有意义** —— shell 工具里注入了 `PI_PROVIDER` / `PI_MODEL` / `PI_REASONING_LEVEL` / `PI_SESSION_ID`，直接打印才准：

   ```text
   !printf '%s/%s\n' "$PI_PROVIDER" "$PI_MODEL"
   ```

   （这些变量只注入模型调用的 `bash` / `powershell` 工具，你自己敲的 `!` 命令拿不到。）

8. **一次性的小问题加 `--no-session`**，别把会话目录堆满。

---

## 6. 常见问题

配置类问题（网关、窗口、权限、环境）在各自的配置文档里有专门一节，这里只放使用层面的。

**Q：`pi` 命令找不到**
关掉终端重开刷新 PATH。还不行检查 `npm config get prefix` 对应的目录在不在 PATH 里。

**Q：安装/启动报 Node 版本错误**
Pi 要求 Node.js 22.19+，`node --version` 确认，升级后重开终端。

**Q：它为什么不问我"是否允许"就直接改文件了？**
设计如此，见 [`配置-权限与规则.md`](./配置-权限与规则.md) 第 2 节。

**Q：它说改了但文件不对 / 改错文件了**
`git status`、`git diff` 看实际差异，`git checkout` 恢复。对话层面用 `/tree` 回看它当时读了什么。

**Q：`AGENTS.md` 没生效**
确认文件名和所在目录（父目录也生效），检查同目录有没有 `AGENTS.override.md` 顶掉它，改完 `/reload`。启动头部会列出实际加载的指令。

**Q：聊久了开始忘事 / 没聊几句就压缩**
先看 `/session` 的上下文用量；窗口明显偏小就检查 `models.json` 里的 `contextWindow`，见 [`配置-模型接入.md`](./配置-模型接入.md) 第 5 节。

**Q：Windows 上 `Alt+Enter` 没反应 / 图片粘不上**
Windows Terminal 占用了部分 `Alt` 组合，Pi 换成了 `Ctrl+Q` / `Alt+Q` / `Alt+P`，图片粘贴是 `Alt+V`。详见 [`配置-运行环境.md`](./配置-运行环境.md) 第 2.5 节。

**Q：会话文件在哪**
`~/.pi/agent/sessions/`。`/session` 会打印当前会话的完整路径；`--session-dir` 或 `PI_CODING_AGENT_SESSION_DIR` 可以改位置。

**Q：想换配置目录**
设 `PI_CODING_AGENT_DIR`，默认 `~/.pi/agent`。

**Q：卸载后配置还在吗**
在。npm 卸载不会动 `~/.pi/agent/` 里的配置、凭据、会话和已装的包，需要手动删。

**Q：出问题了怎么排查**
`/debug` 把渲染结果和会话消息写到 agent 目录的 `pi-debug.log`；`/session` 看会话状态；`/changelog` 看最近改了什么。**分享这些文件前先自己看一遍**，里面可能有代码和凭据。

---

## 7. 进阶方向（用熟了再看）

Pi 的扩展顺序是官方明确建议的 —— **从最轻的机制开始**：

| 想解决的问题 | 用什么 | 详细文档 |
|---|---|---|
| 给某个目录加长期规则 | `AGENTS.md` | [`配置-权限与规则.md`](./配置-权限与规则.md) |
| 把常用提示词变成 `/命令` | 提示词模板（`prompts/`） | [`Extension使用指南.md`](./Extension使用指南.md) |
| 加任务专属说明和配套文件 | Skill（`skills/`） | [`skill使用指导.md`](./skill使用指导.md) |
| 加可执行工具、命令、事件钩子 | 扩展（`extensions/`，TypeScript） | [`Extension使用指南.md`](./Extension使用指南.md) |
| 接一个协议不兼容的模型服务 | 自定义 provider 扩展 | [`Extension使用指南.md`](./Extension使用指南.md) |
| 打包分发上面这些 | Pi 包（`packages`） | 官方 Packages 文档 |

其它值得知道的：

- **Pi 包管理**：`pi install <source>` / `pi list` / `pi config`（TUI 里逐个开关资源）。本机已经装了一个包：`git:github.com/DietrichGebert/ponytail`（见 [`../README.md`](../README.md) 里的 ponytail 一节）。
- **Skill 加载位置**：`~/.pi/agent/skills/`、项目 `.pi/skills/`、以及项目或祖先目录的 `.agents/skills/`（后者受项目信任保护）。启用后可用 `/skill:name` 调用。
- **自定义按键**：`~/.pi/agent/keybindings.json`，按官方 Keybindings 文档里的 action id 覆盖，改完 `/reload`。
- **本地模型**：llama.cpp router（`/llama` 管理）；Ollama / LM Studio / vLLM / SGLang 走 `models.json` 兼容端点，见 [`配置-模型接入.md`](./配置-模型接入.md) 第 7 节。
- **非交互与自动化**：`pi -p`、`--mode json`（JSONL 事件）、`--mode rpc`（双向）、TypeScript SDK。
- **隔离运行**：容器化运行 Pi（官方有专门的容器化文档），这是唯一的强边界。

---

## 附录：一页速查

```text
安装        npm install -g --ignore-scripts @earendil-works/pi-coding-agent   (Node >= 22.19)
启动        cd 项目目录; pi
继续上次    pi -c
选会话      pi -r                       |  pi --session <id>   |  pi --fork <id>
一行问答    pi -p "问题"                |  pi -p --mode json "问题" > out.jsonl
命名会话    pi --name "重构登录模块"
只读模式    pi --tools read,grep,find,ls -p "评审这个模块"
换模型      pi --model sonnet:high "..."   |  pi --models "deepseek-*"
升级        pi update        |  刷新模型目录 pi update --models
装扩展/包   pi install <source>   |  pi list   |  pi config

界面        Enter                 提交
            Shift+Enter / Ctrl+J  换行
            Ctrl+G                外部编辑器写长提示词
            @文件 / Tab           引用文件、补全路径
            Alt+V                 粘贴图片（Windows / WSL）
            !命令 / !!命令        执行命令，输出进 / 不进上下文
            Esc                   打断
            Ctrl+O / Ctrl+T       展开工具输出 / 显示 thinking
            Ctrl+L / Ctrl+P       选模型 / 循环切模型（反向 Alt+P）
            Shift+Tab             切思考等级（不是权限模式！）
            Ctrl+Q / Alt+Q        追加任务 / 收回排队消息（Windows / WSL）
            Ctrl+X                复制最后一条回复
            Ctrl+C 两次           退出

配置        见 配置-运行环境.md / 配置-模型接入.md / 配置-权限与规则.md
            ~/.pi/agent/settings.json     偏好、默认模型、defaultTools、compaction
            ~/.pi/agent/models.json       网关、自定义模型、contextWindow
            ~/.pi/agent/auth.json         /login 存下的凭据（勿提交）
            ~/.pi/agent/trust.json        项目信任决定
            ~/.pi/agent/AGENTS.md         全局规则
            ./.pi/settings.json           项目级设置（需要项目信任）
            ./AGENTS.md 或 ./CLAUDE.md    项目规则（不信任也加载！）

排查        /hotkeys    看当前生效快捷键
            /session    看会话、用量、费用
            /debug      写 pi-debug.log
            pi auth check --provider <name> --json

文档        pi.dev  |  github.com/earendil-works/pi
```
