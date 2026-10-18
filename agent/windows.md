# Windows 原生环境补充说明

**主文档都按 Linux / WSL 写**：`claude/`、`codex/`、`pi/` 里的命令、路径、脚本一律是 bash 与 POSIX 路径。本篇只收**在 Windows 原生环境里跑这三个工具**时的差异，其余场景请直接用 WSL（见 [`wsl.md`](./wsl.md)）。

## 0. 什么时候才用 Windows 原生

只有依赖 Windows 工具链的活儿才留在原生：.NET / WPF / WinForms、MSBuild、COM、注册表、Windows 服务、驱动、Office 插件、VS 工具链。其它（Java / Go / Node / Python / 前端 / 部署在 Linux 的服务）一律进 WSL —— 环境与生产一致，文件 IO 也快得多。

只读代码、评审、写文档这类不动工具链的活，原生最省事。

## 1. 安装

| 工具 | PowerShell |
|---|---|
| Claude Code | `irm https://claude.ai/install.ps1 \| iex`，装到 `%USERPROFILE%\.local\bin` |
| Codex | `powershell -ExecutionPolicy ByPass -c "irm https://chatgpt.com/codex/install.ps1 \| iex"` |
| Pi | `npm install -g --ignore-scripts @earendil-works/pi-coding-agent`（Node ≥ 22.19） |

三个都支持 `npm i -g` 兜底：`@anthropic-ai/claude-code`、`@openai/codex`。装完重开终端刷新 PATH。

## 2. 配置目录

| 工具 | Windows 位置 | WSL 里的对应位置（**不共享**） |
|---|---|---|
| Claude Code | `C:\Users\<you>\.claude\` | `~/.claude/` |
| Codex | `C:\Users\<you>\.codex\` | `~/.codex/` |
| Pi | `C:\Users\<you>\.pi\agent\` | `/home/<you>/.pi/agent/` |

**两份配置互不通用**，这是"我明明配了模型它却说没有"最常见的原因。想两边都能用：网关/模型配置复制过去，API Key 两边各设一次，会话历史接受不通用。

## 3. Claude Code

### 3.1 用 PowerShell 当 Claude 的 shell

两件独立的事，别只配一个：

| 想改什么 | 配什么 | 作用范围 |
|---|---|---|
| Claude **自己执行命令**用 PowerShell | `CLAUDE_CODE_USE_POWERSHELL_TOOL=1`（settings 的 `env` 段） | 模型调用的 shell 工具 |
| **你敲 `!命令`** 时用 PowerShell | 顶层 `defaultShell: "powershell"` | 只影响 `!` 开头的命令 |

```json
{
  "env": { "CLAUDE_CODE_USE_POWERSHELL_TOOL": "1" },
  "defaultShell": "powershell"
}
```

- `defaultShell` 在所有平台默认都是 `bash`，**Windows 上也不会自动变**，合法值只有 `"bash"` / `"powershell"`。
- PowerShell 工具会自动加 `-ExecutionPolicy Bypass`；公司要求遵守执行策略就设 `CLAUDE_CODE_POWERSHELL_RESPECT_EXECUTION_POLICY=1`。
- 验证：输入框里敲 `!$PSVersionTable.PSVersion`，看是 5.1 还是 7.x。

### 3.2 Hook 脚本

Hook 默认走 `sh -c`，Windows 上是 Git Bash（没装 Git Bash 才退回 PowerShell）。想写 `.ps1` 就在**单个 handler** 上显式声明：

```json
{
  "type": "command",
  "shell": "powershell",
  "command": "& \"$env:CLAUDE_PROJECT_DIR\\.claude\\hooks\\check.ps1\""
}
```

坑位（本地实测）：

| 项 | 结论 |
|---|---|
| `$env:CLAUDE_PROJECT_DIR` | ✅ 每个版本都对 |
| `${CLAUDE_PROJECT_DIR}` | ✅ v2.1.198 起改写成 `${env:...}`；单引号里不展开 |
| `$CLAUDE_PROJECT_DIR` | ❌ 解析成 `$null`，路径前缀丢失，只在 debug 日志里留一条警告 |
| `.ps1` 里有中文 | **必须存 UTF-8 with BOM**。PowerShell 5.1 无 BOM 时按系统 ANSI 码页读，轻则乱码、重则整段脚本解析失败 |
| `tool_input.file_path` | **总是反斜杠绝对路径**，即使 hook 跑在 Git Bash 下。比较前先归一化：bash 里 `${FILE_PATH//\\//}`，PowerShell 里 `-replace '\\','/'` |
| exec form（带 `args`） | 要求 `command` 是真可执行文件。`npx`、`eslint` 这类 `node_modules/.bin` 下的 `.cmd`/`.bat` shim **spawn 不起来**，要么用 shell form，要么直接 `{"command":"node","args":["...\\bin\\eslint.js"]}`。带 `args` 时 `shell` 字段被忽略 |
| `jq` | 官方 bash 例子依赖它。Windows 上 `winget install jqlang.jq`，或者改用 `ConvertFrom-Json` / `ConvertTo-Json` |

启动 PowerShell 脚本的 exec form 写法：

```json
{
  "type": "command",
  "command": "powershell.exe",
  "args": ["-NoProfile", "-ExecutionPolicy", "Bypass", "-File", "${CLAUDE_PROJECT_DIR}/.claude/hooks/check.ps1"]
}
```

`-NoProfile` 让 hook 起得快一点；想让公司执行策略生效就去掉 `-ExecutionPolicy Bypass`。

### 3.3 其它

- **执行策略报「禁止运行脚本」**：`Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`。
- **Git Bash 下敲 `claude -p "/fix-issue 1234"` 命令不触发**：MSYS 把 `/fix-issue` 当 Unix 路径吃掉了，加前缀 `MSYS_NO_PATHCONV=1`。PowerShell 里没这个问题。
- IDE 插件（IntelliJ / JetBrains）：`Settings` → `Plugins` → 搜 **Claude Code** → 安装重启，`Ctrl+Esc` 唤起，与终端共用同一份配置。

## 4. Codex

`C:\Users\<you>\.codex\config.toml` 里加一段 Windows 专属配置：

```toml
[windows]
sandbox = "elevated"
```

其它差异：

- 密钥传单次会话：`$env:CODEX_API_KEY = '...'; codex exec "..."; Remove-Item Env:CODEX_API_KEY`。
- Hook 配置可以让两个平台各写一条命令：`command`（POSIX）+ `commandWindows`（`py -3 "..."`）；Windows 上要确认 `py -3`、脚本路径和脚本依赖都在。
- 沙箱、审批策略、`AGENTS.md`、MCP 配置与平台无关，见 [`codex/README.md`](./codex/README.md)。

## 5. Pi

- **Windows 原生默认分工**：模型调用的 `bash` 工具走 Git Bash，`!` / `!!` 命令**永远**走 Bash（换不掉），`powershell` 工具要在 `defaultTools` 里启用后才存在。
- 想换成 PowerShell，写 `~/.pi/agent/settings.json`：

  ```json
  { "defaultTools": ["read", "powershell", "edit", "write", "grep", "find", "ls"] }
  ```

  增量写法：`["-bash", "+powershell"]`。**不要两个都留**，模型会按心情挑一个。
- 启动参数：`-NoProfile -NonInteractive -ExecutionPolicy Bypass`，优先 `pwsh.exe`，没有才退回 `powershell.exe`；公司强制的执行策略仍然优先。
- Bash 找不到时按 `shellPath` → `Program Files` 下 Git Bash → `PATH` 上的 `bash.exe` 顺序找。Cygwin / MSYS2 装歪了就显式指定（JSON 里反斜杠写两个）：

  ```json
  { "shellPath": "C:\\cygwin64\\bin\\bash.exe" }
  ```

- **快捷键差异**（Windows Terminal 占用部分 `Alt` 组合，Windows 和 WSL 用同一套替代键）：

  | 动作 | 通用默认 | Windows / WSL |
  |---|---|---|
  | 追加后续任务（排队） | `Alt+Enter` | `Ctrl+Q` |
  | 收回排队消息 | `Alt+Up` | `Alt+Q` |
  | 反向切模型 | `Shift+Ctrl+P` | `Alt+P` |
  | 粘贴图片 | `Ctrl+V` | `Alt+V` |

  实际生效的按键以 `/hotkeys` 为准。

- 其余（信任、工具白名单、`AGENTS.md`、模型接入）与平台无关，见 [`pi/README.md`](./pi/README.md)。

## 6. 通用的 Windows 坑

- **代码别放 `/mnt/c`**：跨文件系统读写慢一个数量级，`git status`、`npm install`、编辑器索引全都明显变慢。放 `~/code/...`。
- **换行符**：仓库里有 Windows 侧提交的文件时，`git config core.autocrlf` 与 `.gitattributes` 先说清楚，否则 diff 全是整文件变更。
- **编码**：脚本、配置一律 UTF-8。PowerShell 5.1 只对带 BOM 的文件稳定。
- **路径大小写**：Windows 不区分，Linux 区分。写路径比较/glob 时按 Linux 的规则来。
- **不要用 Windows 程序直接编辑 `~/` 里的文件**，用 VS Code 的 Remote-WSL。
- **interop 关掉之后** `.exe` 全失效（`explorer.exe .`、`code .`、Docker Desktop 集成），原因见 [`wsl.md`](./wsl.md) 第 3 节。
