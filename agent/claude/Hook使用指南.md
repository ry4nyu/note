# Claude Code Hook 使用指南

本文面向本仓库团队成员，说明如何在 Claude Code 里配置、使用和排查 Hook。文中机制基于 Claude Code 2.1.x（Windows + PowerShell 环境），跟随版本变化，遇到不一致先敲 `/hooks` 看实际注册了什么，再对官方 Hooks 参考。

前置阅读：[`README.md`](./README.md)（安装、配置、权限模式）；Hook 在 Claude Code 整体能力里的位置、以及「什么时候该用 Hook 而不是 Skill」见 [`skill使用指南.md`](./skill使用指南.md) 第 2 节。Codex 的对应文档见 [`../codex/hook使用指南.md`](../codex/hook使用指南.md)，两边事件名和配置格式相近，但字段细节不同，别混用。

> **关于本文的事实来源**：事件清单、matcher 规则、退出码与 JSON 字段、默认超时值均逐条对过官方 `hooks` 参考与 `hooks-guide` 原文，不是凭印象写的。
>
> 但要说清楚两件事：
>
> - **机制描述（事件触发时机、决策字段、信任规则等）来自官方文档，没有在本机跑 Claude Code 复现过**，按参考值看待。版本迭代快，遇到不一致以 `/hooks` 和官方文档为准。
> - **第 9 节的示例脚本是本机实测的**：脚本按 UTF-8 with BOM 存盘后，在 Windows PowerShell 5.1 和 PowerShell 7 上分别跑过，喂样例 JSON 验证了退出码和输出 —— 拦 `.env` / `package-lock.json` 是 `exit 2`、普通文件是 `exit 0`，`Stop` 示例的 `stop_hook_active` 守卫和 JSON 输出都符合预期。第 10 节第 6 条的编码问题也是实测出来的。

---

## 1. Hook 是什么

Hook 是 Claude Code 生命周期上的**回调**：在会话开始、你提交提示词、工具调用前后、Agent 结束等节点，自动跑一条命令（或发个 HTTP 请求、调一个 MCP 工具、跑一次模型判定）。

它和 Skill 的区别是根本性的：**Skill 是给模型的流程知识，模型可以不照做；Hook 由 harness 直接执行，不经过模型判断，写了就一定跑。**

先分清 Claude Code 里几个「固化规则」的位置：

| 你想要的效果 | 该用哪个 | 为什么 |
|---|---|---|
| 每次都必须发生，不依赖模型判断 | **Hook（本文）** | 由 harness 执行，写了就一定跑 |
| 恒真的项目事实和约定 | `CLAUDE.md`（[`README.md`](./README.md) 第 6 节） | 每轮都在上下文里，代价是每轮都付 token |
| 成体系、要分文件或按路径生效的约定 | Rule（[`rule使用指南.md`](./rule使用指南.md)） | 自动加载，可用 `paths` 只在相关文件上生效 |
| 一类任务的流程、检查单、边界 | Skill（[`skill使用指南.md`](./skill使用指南.md)） | 平时只占一行描述，需要时才展开 |
| 要接入新工具或数据源 | MCP server（[`mcp使用指南.md`](./mcp使用指南.md)） | 提供的是工具本身 |

一句话版：**必须每次都发生的放 Hook，恒真的放 CLAUDE.md，按需的流程放 Skill。**

常见用途：

- 会话开始时注入项目状态（当前分支、未提交改动、正在做的需求）；
- 工具调用前拦住危险操作（`rm -rf`、改 `.env`、往项目外写）；
- 文件改完自动格式化、跑 lint；
- Claude 停下来等你确认时弹个系统通知；
- 收尾前要求它先跑一遍测试；
- 会话结束时保存日志、清理临时文件。

> **Hook 是辅助控制，不是完整的安全边界。** 命令 Hook 以**你的完整用户权限**运行，写错了比不写更危险。真正的硬边界仍然是权限系统、沙箱和操作系统权限 —— 见第 11 节。

---

## 2. 配在哪

Hook 写在 JSON 配置文件里。位置决定作用范围：

| 位置 | 作用范围 | 能提交给同事吗 |
|---|---|---|
| `~/.claude/settings.json` | 你所有项目 | 否，只在本机 |
| `<项目>/.claude/settings.json` | 单个项目 | **能**，跟着仓库走 |
| `<项目>/.claude/settings.local.json` | 单个项目 | 否，Claude Code 往里写设置时会自动 gitignore |
| 托管策略设置（企业下发） | 全组织 | 是，管理员控制 |
| 插件的 `hooks/hooks.json` | 插件启用期间 | 是，跟插件一起分发 |
| Skill 的 frontmatter `hooks:` | 该 Skill 被调用后的**整个会话** | 是，写在 Skill 文件里 |
| Subagent 的 frontmatter `hooks:` | 该子 agent 运行期间 | 是，写在 agent 文件里 |

几条容易踩的：

- **跨层是合并不是覆盖**。用户级、项目级、本地级各自追加自己的 hook，不会替换掉别的层。同一条 hook 在多个设置文件里重复定义，只会跑一次；但插件/Skill 里那份算独立的一份，会各跑一次。
- **`disableAllHooks` 能关掉自己的 hook，关不掉托管层的**。托管策略设的 hook 只能由托管层关（第 11 节）。
- 直接改设置文件的 hook，文件监听器通常会自动拾取；没生效就重启一次会话强制重新加载。
- ⚠️ **本仓库的 `.claude/` 在 `.gitignore` 里**（与 [`skill使用指南.md`](./skill使用指南.md) 第 8 节同一个坑）。往 `.claude/settings.json` 里写 hook 是能跑的，但**不会提交**，同事拉不到。要共享得先决定怎么处理这条忽略规则。

---

## 3. 最小示例：会话开始时注入项目状态

### 3.1 写配置

在项目根目录建 `.claude/settings.json`：

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup|resume",
        "hooks": [
          {
            "type": "command",
            "shell": "powershell",
            "command": "& \"$env:CLAUDE_PROJECT_DIR\\.claude\\hooks\\session-context.ps1\"",
            "statusMessage": "读取项目状态",
            "timeout": 10
          }
        ]
      }
    ]
  }
}
```

注意 `"shell": "powershell"` —— Hook 默认走 `sh -c`（Windows 上是 Git Bash），这里显式改成 PowerShell，省得在两种语法之间来回切。`$env:CLAUDE_PROJECT_DIR` 见第 10 节，**这里不要写裸的 `$CLAUDE_PROJECT_DIR`**。

### 3.2 写脚本

建 `.claude/hooks/session-context.ps1`：

```powershell
$branch = git rev-parse --abbrev-ref HEAD 2>$null

if (-not $branch) {
  Write-Output "当前目录不是 Git 仓库：$PWD"
} else {
  Write-Output "当前分支：$branch"
  $dirty = git status --porcelain 2>$null | Select-Object -First 10
  if ($dirty) {
    Write-Output "工作区有未提交改动，改动会混进本次会话："
    Write-Output $dirty
  } else {
    Write-Output "工作区干净。"
  }
}
```

`SessionStart` 是少数几个「stdout 直接进上下文」的事件之一，所以纯文本 `Write-Output` 就够了，不用拼 JSON。

等价的 bash 版本（配置里去掉 `"shell": "powershell"`，脚本换成）：

```bash
#!/bin/bash
branch=$(git rev-parse --abbrev-ref HEAD 2>/dev/null)
if [ -z "$branch" ]; then
  echo "当前目录不是 Git 仓库：$PWD"
else
  echo "当前分支：$branch"
  git status --porcelain | head -10
fi
```

> 加那句「不是 Git 仓库」的判断不是为了好看：`git` 失败时 `$branch` 是空字符串，不加判断就会输出「工作区干净」—— **hook 输出谎报状态，比不输出更糟**。写 hook 时对失败路径要显式处理，别让它静默变成一句好消息。

> ⚠️ **`.ps1` 脚本里有中文，就存成 UTF-8 with BOM**（VS Code 右下角编码 → `Save with Encoding` → `UTF-8 with BOM`）。否则在 Windows PowerShell 5.1 下中文会乱码、甚至整个脚本解析失败 —— 本地实测过，见第 10 节第 6 条。这是本文所有 PowerShell 例子的前提。

### 3.3 验证

```text
/hooks
```

`/hooks` 是**只读**的浏览器：列出所有事件、每个事件上配了几个 hook、每个 hook 来自哪个文件、完整命令是什么。看到 `SessionStart` 那一行带着计数就说明注册成功。要改只能去改 JSON，或者直接让 Claude 帮你改。

然后**重启会话**（`SessionStart` 只在会话开始时触发），确认上下文里出现了分支信息。启动时的 `SessionStart` 是后台跑的：你可以立刻打字，但你的第一句要等它跑完才发给模型。

---

## 4. 结构：事件 → matcher → handler

一个 hook 配置是三层嵌套：

1. **事件（event）**：响应哪个生命周期节点，如 `PreToolUse`、`Stop`；
2. **matcher group**：过滤什么时候触发，如「只在 Bash 工具上」；
3. **handler**：命中后跑什么。

### 4.1 handler 的五种类型

每个 handler 的 `type` 决定它怎么跑：

| `type` | 做什么 | 结果怎么回传 |
|---|---|---|
| `command`（最常用） | 跑一条 shell 命令/脚本 | stdin 收 JSON，靠**退出码 + stdout/stderr** 回传 |
| `http` | 把事件 JSON POST 到一个 URL | 靠 HTTP 状态码 + 响应体回传（响应体用同一套 JSON 格式） |
| `mcp_tool` | 调一个**已连接** MCP 工具（不会触发 OAuth 或重连） | 工具的文本输出当作 command 的 stdout 处理 |
| `prompt` | 发一次单轮模型判定，让模型回 `{ok: true/false}` | 结构化 JSON |
| `agent` | 拉起一个能读文件、能搜索的子 agent 做验证 | 同上（**实验特性**，生产别依赖） |

常用字段：

| 字段 | 说明 |
|---|---|
| `command` | 要执行的命令。**带 `args` 时按可执行文件直接 spawn（exec form），不带时交给 shell（shell form）** |
| `args` | 参数数组。写了就切到 exec form，不走 shell —— 引号、`$`、反引号都是字面量 |
| `timeout` | 超时秒数，见 4.4 |
| `statusMessage` | hook 跑的时候转圈上显示的话 |
| `shell` | `"bash"` 或 `"powershell"`。默认 bash（Windows 上没装 Git Bash 时默认 PowerShell） |
| `async` | `true` 则丢后台跑，不阻塞 Claude。**只有 `command` 支持** |
| `asyncRewake` | 后台跑，且退出码为 2 时把 Claude 叫醒 |
| `if` | 权限规则语法的额外过滤，见 4.3 |
| `once` | 跑成功一次后移除。**只在 Skill frontmatter 里有效**，设置文件里写了会被忽略 |

> **exec form 和 shell form 的区别不是小事**。exec form 里每个 `args` 元素原样作为一个参数，不做 shell 展开，引号也是字面量。Windows 上 exec form 要求 `command` 是个真可执行文件（`.exe`）——`npx`、`eslint` 这类装在 `node_modules/.bin` 的 `.cmd`/`.bat` shim **不能**被直接 spawn，要么用 shell form，要么写成 `{"command": "node", "args": ["...\\node_modules\\eslint\\bin\\eslint.js"]}`。

### 4.2 matcher 怎么写

`matcher` 怎么求值，取决于它里面有哪些字符：

| matcher 值 | 按什么求值 | 例子 |
|---|---|---|
| `"*"`、`""`、省略 | 全部匹配 | 该事件每次触发都跑 |
| 只含字母、数字、`_`、`-`、空格、`,`、`\|` | **精确匹配**（可以用 `\|` 或 `,` 分隔多个） | `Bash` 只匹配 Bash 工具；`Edit\|Write` 匹配两者之一；`code-reviewer` 只匹配这个 agent |
| 含任何其它字符 | **未锚定的 JavaScript 正则** | `^Notebook` 匹配所有以 Notebook 开头的工具 |

未锚定这点最容易出事：**`Edit.*` 会同时匹配 `Edit` 和 `NotebookEdit`**，要整串匹配必须自己加 `^...$`。

不同事件匹配的字段也不一样：

| 事件 | matcher 过滤什么 | 可用值 |
|---|---|---|
| `PreToolUse`、`PostToolUse`、`PostToolUseFailure`、`PermissionRequest`、`PermissionDenied` | 工具名 | `Bash`、`Edit\|Write`、`mcp__.*` |
| `SessionStart` | 会话怎么开始的 | `startup`、`resume`、`clear`、`compact`、`fork` |
| `SessionEnd` | 会话为什么结束 | `clear`、`resume`、`logout`、`prompt_input_exit`、`other` |
| `Notification` | 通知类型 | `permission_prompt`、`idle_prompt`、`auth_success` 等 |
| `PreCompact`、`PostCompact` | 谁触发的压缩 | `manual`、`auto` |
| `SubagentStart`、`SubagentStop` | agent 类型 | `general-purpose`、`Explore`、自定义 agent 名 |
| `ConfigChange` | 配置来源 | `user_settings`、`project_settings`、`local_settings`、`policy_settings`、`skills` |
| `FileChanged` | **要监视的文件名**（不是事件字段） | `.envrc\|.env` |
| `CwdChanged`、`UserPromptSubmit`、`PostToolBatch`、`Stop`、`TeammateIdle`、`TaskCreated`、`TaskCompleted`、`WorktreeCreate`、`WorktreeRemove`、`MessageDisplay` | 不支持 matcher | 写了会被静默忽略 |

**MCP 工具怎么匹配**：MCP 工具就是普通工具，名字形如 `mcp__<server>__<tool>`（例：`mcp__filesystem__read_file`）。要匹配某个 server 的全部工具，**`.*` 必须写**：

```text
mcp__memory__.*           ✓ 匹配 memory server 的所有工具
mcp__memory               ✗ 只含精确匹配字符，被当成整串比较，什么都匹配不到
mcp__.*__write.*          ✓ 任意 server 上名字以 write 开头的工具
mcp__brave-search__.*     ✓ server 名里有连字符也这么写
```

### 4.3 `if`：比 matcher 更细的过滤

`matcher` 只能按工具名过滤，`if` 能同时看工具名和参数，用的是**权限规则的语法**：

```json
{
  "type": "command",
  "if": "Bash(git *)",
  "command": "..."
}
```

- `"Bash(git *)"`：Bash 子命令匹配 `git *` 时才跑。前导的 `VAR=value` 赋值会被剥掉，`&&`/`;` 分开的每条子命令、`$()` 和反引号里的内容都会检查。
- `"Edit(*.ts)"`：只对 TypeScript 文件跑。
- **一个 `if` 只能写一条规则**，没有 `&&`、`||`、列表语法。要多个条件就写多个 handler。

⚠️ **`if` 只在 tool 事件上求值**（`PreToolUse`、`PostToolUse`、`PostToolUseFailure`、`PermissionRequest`、`PermissionDenied`）。**在其它事件上，带 `if` 的 hook 永远不跑** —— 这个失败方式很安静，容易查半天。

> `if` 是**尽力而为**的：当 Claude Code 无法判断一条 Bash 输入会执行什么（比如 `$TOOL git push` 这种变量当命令名），它会**照样跑**你的 hook。所以别用 `if` 当硬性的准入检查，那是权限系统的活。

### 4.4 默认超时

| 类型 | 默认 | 例外 |
|---|---|---|
| `command`、`http`、`mcp_tool` | 600 秒 | `UserPromptSubmit`、`PreModelSwitch`、`PostModelSwitch` 降到 30 秒；`MessageDisplay` 降到 10 秒 |
| `prompt` | 30 秒 | |
| `agent` | 60 秒 | |
| `SessionEnd`（任意类型） | **共 1.5 秒预算** | 单个 hook 设了更长的 `timeout`，预算会抬到跟它一致，最高 60 秒 |

超时会被取消并**丢弃输出**，所以在大多数事件上「超时」= 这个 hook 没有任何决策。`async: true` 的 hook 不受 `timeout` 约束（`asyncRewake` 受）。

---

## 5. 输入：stdin 收到的 JSON

命令 Hook 从 **stdin** 收一个 JSON 对象；HTTP Hook 收的是同一个 JSON 作为 POST body。每个事件都会带上这些公共字段：

| 字段 | 含义 |
|---|---|
| `session_id` | 当前会话 ID |
| `prompt_id` | 当前这轮用户提示词的 UUID（第一条输入之前没有） |
| `transcript_path` | 会话记录 JSON 的路径。**它是异步写的，可能落后于内存里的对话**，本轮最后一条消息未必在里面 |
| `cwd` | hook 被调用时的工作目录 |
| `scratchpad_dir` | 会话的临时工作目录（没有就是空） |
| `permission_mode` | 当前权限模式：`default`、`plan`、`acceptEdits`、`auto`、`dontAsk`、`bypassPermissions`。注意**界面上叫 Manual 的那个，传过来是 `default`**，不是 `manual` |
| `hook_event_name` | 触发的事件名 |
| `effort` | 思考强度，在工具调用语境下的事件（`PreToolUse`、`PostToolUse`、`Stop`、`SubagentStop`）里会有 |

在子 agent 里（或 `--agent` 启动时）还会多两个：`agent_id`（区分子 agent 调用和主线程调用）、`agent_type`（agent 名）。

工具事件额外给 `tool_name`、`tool_input`、`tool_use_id`；`PostToolUse` 还有 `tool_response` 和 `duration_ms`。一个 `PreToolUse` 的输入长这样：

```json
{
  "session_id": "abc123",
  "transcript_path": "C:\\Users\\你的名字\\.claude\\projects\\...\\transcript.jsonl",
  "cwd": "C:\\Users\\你的名字\\IdeaProjects\\your-project",
  "permission_mode": "default",
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": { "command": "npm test", "description": "Run test suite" },
  "tool_use_id": "toolu_01ABC123..."
}
```

### 5.1 路径占位符与环境变量

| 占位符 / 变量 | 含义 |
|---|---|
| `${CLAUDE_PROJECT_DIR}` | **会话启动时**的项目根。exec form 里会替换进 `command` 和每个 `args` 元素；shell form 里要用双引号包起来 |
| `$env:CLAUDE_PROJECT_DIR`（PowerShell） / `$CLAUDE_PROJECT_DIR`（bash） | 同名环境变量，脚本里直接读也行 |
| `${CLAUDE_PLUGIN_ROOT}` / `${CLAUDE_PLUGIN_DATA}` | 插件安装目录 / 插件的持久数据目录 |
| `$CLAUDE_ENV_FILE` | 一个文件路径，**只对 `SessionStart`、`Setup`、`CwdChanged`、`FileChanged` 可用**。往里面追加 `export` 语句，后续的 Bash 命令就能看到这些环境变量 |
| `$CLAUDE_EFFORT` | 当前思考强度 |

⚠️ **worktree 里这两个东西会分叉**：`${CLAUDE_PROJECT_DIR}` 仍指向**会话启动时的项目根**（不会跟着进 worktree），而 `cwd` 字段跟着 Claude 走 —— 进了 worktree 就是 worktree 根。需要知道「Claude 现在在哪个目录干活」就读 `cwd`。

> 没有 `$CLAUDE_MODEL` 这个变量。只有 `SessionStart` 的输入里可能带 `model` 字段，而且不一定有；会话中途 `/model` 切换它也不会更新。

---

## 6. 输出：退出码与 JSON

### 6.1 退出码

| 退出码 | 含义 |
|---|---|
| `0` | 成功，没有异议。**这不等于批准**：`PreToolUse` 返回 0，工具调用照样走正常权限流程 |
| `2` | **阻断**。理由写 stderr（或在 JSON 里给 `reason`）。在能阻断的事件上，**JSON 也覆盖不了它** |
| 其它（包括 `1`） | 不阻断。**Unix 惯例的 `1` 在这里不阻断**，这是最容易踩的一个坑 —— 想拦就必须 `exit 2` |

⚠️ 脚本路径写错、文件不存在时，shell 会以 127 之类的码退出，结果只是一条**非阻断**的 `<hook 名> hook error` 提示，动作照常执行。**配策略类 hook 时一定要看第一次运行有没有这条提示**，否则等于门没锁上你还以为锁了。

### 6.2 JSON 输出怎么被解析

stdout 要「**以 `{` 开头、以 `}` 结尾**」（忽略首尾空白）才会被当 JSON 解析；否则整段当纯文本。常见事故是 shell profile 里有一句无条件的 `echo`，输出变成：

```text
Shell ready on arm64
{"decision": "block", "reason": "Not allowed"}
```

这在退出码 0 时**什么都不提示**，只在 debug 日志里留一行。修法是给 profile 里的 echo 加交互判断（`if [[ $- == *i* ]]; then ... fi`），或者用 exec form 绕开 shell。

另外两条：

- JSON 校验不通过、或本该是 JSON 却解析失败，都是**非阻断错误**：动作照做，transcript 里显示一条 `<hook 名> hook error`。
- `additionalContext`、`systemMessage`、`initialUserMessage` 以及纯 stdout，**每个字符串上限 10,000 字符**。超了会被写进一个文件，只把路径和前 2000 字符预览给 Claude —— 所以**必须让 Claude 看到的东西别超过这个量**。

### 6.3 通用字段

| 字段 | 默认 | 说明 |
|---|---|---|
| `continue` | `true` | 设 `false` 则 hook 跑完后 Claude **彻底停止处理**，优先级高于各事件的决策字段 |
| `stopReason` | 无 | `continue: false` 时给你看的消息。它留在对话里，所以对话继续的话 Claude 也能看到 |
| `systemMessage` | 无 | 给你看的警告。部分事件会丢弃它，或送到别处 |
| `terminalSequence` | 无 | 让 Claude Code 替你发一个终端转义序列（桌面通知、窗口标题、响铃），见 6.5 |
| `suppressOutput` | `false` | **官方明确「接受但无效果」**。网上不少文章说它能静音 hook 输出，是错的 —— 成功的 hook stdout 本来就不会进 transcript，只进 debug 日志 |

### 6.4 决策字段：按事件分组

| 事件 | 用什么字段 | 关键值 |
|---|---|---|
| `UserPromptSubmit`、`UserPromptExpansion`、`PostToolUse`、`PostToolUseFailure`、`PostToolBatch`、`Stop`、`SubagentStop`、`ConfigChange`、`PreCompact` | 顶层 `decision` | `{"decision": "block", "reason": "..."}`。`Stop`/`SubagentStop` 还能用 `hookSpecificOutput.additionalContext` 给「非报错」的反馈 |
| `PreToolUse` | `hookSpecificOutput` | `permissionDecision`：`allow` / `deny` / `ask` / `defer`；配 `permissionDecisionReason`。还能用 `updatedInput` 改写参数 |
| `PermissionRequest` | `hookSpecificOutput` | `decision.behavior`：`allow` / `deny`（`exit 2` 在这个事件上**不生效**，只能用 JSON） |
| `PermissionDenied` | `hookSpecificOutput` | `retry: true` 告诉模型可以重试被拒的调用 |
| `SessionStart`、`SubagentStart`、`PostModelSwitch` | 只能加上下文 | `additionalContext`。`SessionStart` 还认 `initialUserMessage`、`sessionTitle`、`watchPaths`、`reloadSkills`。**不能阻断** |
| `Setup`、`Notification`、`SessionEnd`、`PostCompact`、`InstructionsLoaded`、`StopFailure`、`CwdChanged`、`DirectoryAdded`、`FileChanged` | 无 | 没有决策控制，用来做副作用（日志、清理） |
| `WorktreeCreate` | 打印路径 | 命令 Hook 把 worktree 路径打印到 stdout；失败或缺路径则创建失败 |
| `Elicitation`、`ElicitationResult` | `hookSpecificOutput` | `action`：`accept` / `decline` / `cancel` |

⚠️ **`PreToolUse` 的顶层 `decision` / `reason` 已经废弃**，必须写进 `hookSpecificOutput.permissionDecision` / `permissionDecisionReason`。老写法（`"approve"` / `"block"`）对应 `"allow"` / `"deny"`。

**多个 hook 同时命中时**：全部**并行**跑完再合并结果，谁都不能阻止别人跑。`PreToolUse` 的最终决策取**最严**的那个：`deny` > `defer` > `ask` > `allow`。`additionalContext` 则是**每个 hook 的都会合并**给 Claude。多个 hook 都用 `updatedInput` 改同一个工具时，**最后跑完的那个生效**，而并行顺序不确定 —— 所以别让两个 hook 改同一个工具的输入。

### 6.5 `additionalContext` 与桌面通知

`additionalContext` 把一段文本塞进 Claude 的上下文，Claude Code 会用 system reminder 包起来插在 hook 触发的那个位置：`SessionStart` 是对话开头、`UserPromptSubmit` 是紧随你这条提示词、工具事件是贴着工具结果、`Stop` 是这一轮末尾（对话会继续，让 Claude 能据此行动）。

写的时候**写成事实陈述，不要写成系统指令**：「这个仓库用 `bun test`」读起来是项目信息；写成「系统指令：必须执行 bun test」可能触发 Claude 的提示词注入防御，它会反过来把这段拿给你看。

Hook 跑在没有控制终端的会话里，`/dev/tty` 写不了，想发桌面通知就返回 `terminalSequence`，由 Claude Code 替你写出去。允许的序列只有 OSC `0`/`1`/`2`（窗口标题）、OSC `9`（Windows Terminal、ConEmu、WezTerm 等）、OSC `99`（Kitty）、OSC `777`（Ghostty、Warp、urxvt）和裸 BEL，别的一律被忽略（包括改光标、改颜色、OSC 52 写剪贴板）。它只在交互式会话里生效，`-p` 模式和 SDK 里会被忽略。

---

## 7. 事件一览

共 33 个事件。「能否阻断」列指**退出码 2 或对应 JSON 能否拦下这个动作**。

| 事件 | 触发时机 | 能否阻断 |
|---|---|---|
| `SessionStart` | 会话开始或恢复时 | 否，stderr 只给你看 |
| `Setup` | `claude --init-only`，或 `-p` 模式下的 `--init` / `--maintenance` | 否，输出和退出码都被丢弃 |
| `UserPromptSubmit` | 你提交提示词、Claude 处理之前 | **能**，拦下并抹掉这条提示词 |
| `UserPromptExpansion` | 手敲的 `/命令` 展开成提示词之前 | **能**（`PreToolUse` 覆盖不到这条路径） |
| `PreToolUse` | 工具调用执行前 | **能** |
| `PermissionRequest` | 工具调用需要权限决策时 | 否，但可以用 JSON 的 `decision.behavior` 允许/拒绝 |
| `PermissionDenied` | auto 模式拒绝了工具调用时 | 否（拒绝已经发生）；可用 `retry: true` 让模型重试 |
| `PostToolUse` | 工具调用**成功后** | 否，只能把 stderr 给 Claude 看 |
| `PostToolUseFailure` | 工具调用**失败后** | 否，同上 |
| `PostToolBatch` | 一批并行工具调用全部结束、下一次模型请求前 | **能**，停掉 agentic loop |
| `Notification` | Claude Code 发通知时 | 否 |
| `MessageDisplay` | 助手消息文本上屏时 | 否（只能改屏显，transcript 和 Claude 看到的仍是原文） |
| `SubagentStart` | 子 agent 被创建时 | 否，stderr 只给你看 |
| `SubagentStop` | 子 agent 结束时 | **能**，不让它停 |
| `TaskCreated` | 通过 `TaskCreate` 创建任务时 | **能**，回滚这次创建 |
| `TaskCompleted` | 任务被标记完成时 | **能**，不让它标记完成 |
| `Stop` | Claude 回复结束时 | **能**，不让它停、继续对话。**用户中断不触发**，API 错误走 `StopFailure` |
| `StopFailure` | 因 API 错误结束这一轮时 | 否，输出和退出码都被忽略 |
| `TeammateIdle` | agent team 的队友即将空闲时 | **能**，不让它空闲 |
| `InstructionsLoaded` | `CLAUDE.md` 或 `.claude/rules/*.md` 载入上下文时 | 否，退出码被忽略 |
| `ConfigChange` | 会话中配置文件发生变化时 | **能**（`policy_settings` 除外） |
| `CwdChanged` | 工作目录变化时（比如 Claude 执行了 `cd`） | 否，stderr 只给你看 |
| `DirectoryAdded` | 会话中通过 `/add-dir` 追加了工作目录 | 否，目录已经加上了 |
| `FileChanged` | 被监视的文件在磁盘上变化时 | 否，发生在变化之后 |
| `WorktreeCreate` | 创建 worktree 时 | **能**，任何非 0 退出码都会让创建失败 |
| `WorktreeRemove` | 移除 worktree 时 | **能**，非 0 退出码让移除失败 |
| `PreCompact` | 上下文压缩前 | **能**，拦下这次压缩 |
| `PostCompact` | 压缩完成后 | 否，stderr 只给你看 |
| `PreModelSwitch` | 应用模型切换前 | **能**，拦下切换 |
| `PostModelSwitch` | 会话模型变化后 | 否，模型已经换了 |
| `Elicitation` | MCP server 在工具调用中请求用户输入时 | **能**，拒绝这次请求 |
| `ElicitationResult` | 用户回应 MCP 请求后、回传 server 前 | **能**，拦下回应（变成 decline） |
| `SessionEnd` | 会话终止时 | 否，输出被丢弃，stderr 只给你看 |

只支持 `command` / `http` / `mcp_tool`、不支持 `prompt` / `agent` 的事件：`SessionStart`、`Setup`、`ConfigChange`、`CwdChanged`、`DirectoryAdded`、`Elicitation`、`ElicitationResult`、`FileChanged`、`InstructionsLoaded`、`MessageDisplay`、`Notification`、`PostCompact`、`PreCompact`、`PreModelSwitch`、`PostModelSwitch`、`SessionEnd`、`StopFailure`、`SubagentStart`、`WorktreeCreate`、`WorktreeRemove`。

---

## 8. 常用事件详解

### 8.1 `SessionStart`

会话开始或恢复时触发，**每次会话都跑，所以要快**。只支持 `command` 和 `mcp_tool`。

输入里多一个 `source`：`startup`（新会话）、`resume`（`--resume` / `--continue` / `/resume`）、`clear`（`/clear` 后）、`compact`（压缩后）、`fork`（从已有会话 fork 出来）。可能带 `model`，也可能没有 —— 读之前先判断。

**输出**（都用 `hookSpecificOutput`）：

| 字段 | 作用 |
|---|---|
| `additionalContext` | 会话开头注入上下文 |
| `initialUserMessage` | 当作本会话第一条用户消息（`-p` 模式下有用，没有提示词也能发起第一轮） |
| `sessionTitle` | 直接设会话标题，等于 `/rename`。`clear` / `compact` 时忽略 |
| `watchPaths` | 绝对路径数组，本次会话要监视这些路径的 `FileChanged` 事件 |
| `reloadSkills` | `true` 则在 hook 跑完后重新扫描 skill / command 目录。用在「hook 自己装了 skill」的场景 |

另外，`SessionStart` 能拿到 `$CLAUDE_ENV_FILE`，往里追加 `export` 就能给后续的 Bash 命令持久化环境变量：

```powershell
if ($env:CLAUDE_ENV_FILE) {
  Add-Content $env:CLAUDE_ENV_FILE 'export NODE_ENV=production'
}
```

> 恢复会话（`--continue` / `--resume`）时它**会再跑一次**，`source` 是 `resume` —— 所以它可以用来刷新会过期的上下文。但中途事件（`PostToolUse`、`UserPromptSubmit`）注入的文本在被恢复时是**重放存档**，不会重跑 hook，时间戳、commit SHA 这类值会变旧。

### 8.2 `UserPromptSubmit`

你提交提示词、模型处理之前触发。默认超时只有 30 秒，而且它**卡住就卡住整个会话**，别在这里做慢活。

两条加上下文的路子（都是退出码 0）：纯文本 stdout，或者 JSON 里的 `additionalContext`。两者都不显示在 transcript 里，只是作为 system reminder 给 Claude 读。

要拦下提示词：

```json
{
  "decision": "block",
  "reason": "给用户看的拦截理由（不会进上下文）",
  "hookSpecificOutput": {
    "hookEventName": "UserPromptSubmit",
    "additionalContext": "给 Claude 的补充上下文"
  }
}
```

它**不能改写提示词**，只能拦掉或加料。

### 8.3 `PreToolUse`

工具调用执行前触发。匹配任何工具名（`Bash`、`PowerShell`、`Edit`、`Write`、`Read`、`Glob`、`Grep`、`Agent`、`WebFetch`、`WebSearch`、`AskUserQuestion`、`ExitPlanMode` 以及 MCP 工具）。

⚠️ 两个它**不覆盖**的场景：

- **在提示词里用 `@文件` 引用**：内容是拼提示词时直接插进去的，**没有任何工具调用**，所以 `PreToolUse` 不触发（连匹配 `Read` 的也不触发）。要拦这类路径只能靠 `Read` 的 deny 规则。
- `EndConversation` 工具。

决策字段：

| 字段 | 说明 |
|---|---|
| `permissionDecision` | `allow` 跳过权限提示（但**不能**覆盖设置里的 deny 规则）；`deny` 拦下（理由给 Claude 看）；`ask` 弹给你确认；`defer` 优雅退出、留给之后恢复（只在 `-p` 模式有意义） |
| `permissionDecisionReason` | `allow` / `ask` 时给**你**看，`deny` 时给 **Claude** 看，`defer` 时忽略 |
| `updatedInput` | 改写工具参数。**是整个替换**，所以没改的字段也要一起带上；改完之后权限规则是按**新参数**判的 |
| `additionalContext` | 贴着工具结果给 Claude 加一段 |

`exit 2` 的效果等同于 `deny`：Claude 看到 stderr 作为拒绝理由。

### 8.4 `PostToolUse`

工具调用**成功后**触发。工具事件里唯一「事后」的那个 —— 副作用已经发生了，它**撤销不了任何东西**。

| 字段 | 说明 |
|---|---|
| `decision: "block"` + `reason` | 把理由附在工具结果旁边（Claude 仍然看得到原始输出） |
| `updatedToolOutput` | **替换**工具输出。值要匹配该工具的输出结构（`Bash` 是 `{stdout, stderr, interrupted, isImage}`）；内置工具结构不对会**被忽略**、仍用原始输出 |
| `additionalContext` | 贴着工具结果加一段上下文 |
| `classifierContext` | 只给 auto 模式的分类器看的一条短注记（Claude 看不到），2000 字符上限 |

要拦「还没发生」的事只能用 `PreToolUse`。另外 `PostToolUse` 只对 `Edit|Write` **工具调用**生效；如果是 `Bash` 命令或 Claude Code 之外的进程改了同一个文件，得用 `FileChanged`。

### 8.5 `Stop`

Claude 回复结束时触发。**注意它是「每一轮回复结束」都触发，不是「任务完成」时触发**，所以别在这里写只有收尾才该做的事。

输入里几个关键字段：

- **`stop_hook_active`**：因为 stop hook 而继续的回合里为 `true`。**用它防死循环**，见 8.5 下面的例子。
- `last_assistant_message`：Claude 最后一条回复的文本。**要拿这轮最终输出就读它，别去读 `transcript_path`** —— 记录文件是异步写的，Stop 那一刻未必包含最后一条消息。
- `background_tasks` / `session_crons`：用来区分「会话真干完了」和「只是停下来等后台任务把它唤醒」。

输出：`{"decision": "block", "reason": "..."}` 不让它停，`reason` 会变成它的下一条指令；或者用 `hookSpecificOutput.additionalContext` 给「非报错」的反馈 —— 对话同样继续，但 transcript 里标的是 `Stop hook feedback` 而不是 hook error。

**连续阻断 8 次后会被强制放行**，并在末尾给一条警告。合法场景真的需要更多轮，用 `CLAUDE_CODE_STOP_HOOK_BLOCK_CAP` 抬上限。

### 8.6 `PreCompact` / `PostCompact`

`PreCompact` 在压缩前触发，matcher 是 `manual`（`/compact`）或 `auto`（撞到自动压缩窗口）。可以 `exit 2` 或 `decision: "block"` 拦下压缩。

拦自动压缩的后果分两种：如果压缩是**提前预防性**触发的，跳过就跳过了，对话继续跑；如果压缩是为了**从上下文超限错误里恢复**，那底层错误会直接冒出来、当前请求失败。所以别无条件拦自动压缩。

`PostCompact` 在压缩完成后触发，输入里带 `compact_summary`（生成的摘要）。它没有决策控制，适合用来记录摘要或同步外部状态。

### 8.7 `SessionEnd`

会话终止时触发，用 `reason` 区分退出原因：

| `reason` | 含义 |
|---|---|
| `clear` | 执行了 `/clear` |
| `resume` | 交互式 `/resume` 切了会话 |
| `logout` | 登出 |
| `prompt_input_exit` | 在提示词输入框可见时退出 |
| `other` | 其它 |

它没有决策控制、输出会被丢弃，只能做清理和记日志。**预算是所有 `SessionEnd` hook 共享 1.5 秒**，超了就来不及跑完：要给某个 hook 更多时间就在它自己身上设 `timeout`（预算会自动抬到跟它一致，最高 60 秒），或者用 `CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS` 一次性设总预算（毫秒）。

### 8.8 `Notification`

Claude Code 要发通知时触发，matcher 是通知类型，常用的两个：

- `permission_prompt`：Claude 需要你批准工具调用，且这个提示**已经等了大约 6 秒**；
- `idle_prompt`：Claude 回复完约 60 秒、你一直没打字。

也就是说 **`permission_prompt` 不是「一需要权限就通知」**，你想要「立刻反应」应该用 `PermissionRequest` 事件。

`Notification` 的 `systemMessage`、`continue` 都会被丢弃，**只有 `terminalSequence` 还有效** —— 所以它最实际的用法是转发到外部系统或弹桌面通知。

### 8.9 其余事件

`Setup`（CI / 脚本里的一次性准备）、`SubagentStart` / `SubagentStop`、`TaskCreated` / `TaskCompleted`、`TeammateIdle`、`PermissionRequest` / `PermissionDenied`、`UserPromptExpansion`、`PostToolBatch` / `PostToolUseFailure`、`InstructionsLoaded`、`ConfigChange`、`CwdChanged`、`DirectoryAdded`、`FileChanged`、`WorktreeCreate` / `WorktreeRemove`、`PreModelSwitch` / `PostModelSwitch`、`Elicitation` / `ElicitationResult`、`MessageDisplay`、`StopFailure` 这些用得少，需要时对照官方 Hooks 参考的对应小节。

---

## 9. 六个实战例子

### 9.1 拦住改敏感文件

`.claude/settings.json`：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "shell": "powershell",
            "command": "& \"$env:CLAUDE_PROJECT_DIR\\.claude\\hooks\\protect-files.ps1\""
          }
        ]
      }
    ]
  }
}
```

`.claude/hooks/protect-files.ps1`：

```powershell
$callInput = [Console]::In.ReadToEnd() | ConvertFrom-Json
$path = $callInput.tool_input.file_path -replace '\\', '/'   # Windows 上是反斜杠，先归一化

$blocked = @('(^|/)\.env', '/package-lock\.json$', '/\.git/')
foreach ($pattern in $blocked) {
  if ($path -match $pattern) {
    [Console]::Error.WriteLine("已阻止：$path 命中受保护规则 $pattern")
    exit 2
  }
}
exit 0   # 没有异议，走正常权限流程
```

bash 版本：

```bash
#!/bin/bash
INPUT=$(cat)
FILE_PATH=$(jq -r '.tool_input.file_path // empty')
FILE_PATH="${FILE_PATH//\\//}"

for pattern in '(^|/)\.env' '/package-lock\.json$' '/\.git/'; do
  if [[ "$FILE_PATH" =~ $pattern ]]; then
    echo "已阻止：$FILE_PATH 命中受保护规则 $pattern" >&2
    exit 2
  fi
done
exit 0
```

**验证**：让 Claude 给 `.env` 加一行注释，应该在调用前被拦下，并把 `已阻止：...` 当作反馈给到它。

> 这是**额外防线**，不是硬边界 —— 它用你的权限跑，而且只覆盖 `Edit`/`Write` 工具。真正的硬边界仍然是权限规则（`deny` 规则）和操作系统权限。

### 9.2 改完自动格式化

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "shell": "powershell",
            "command": "& \"$env:CLAUDE_PROJECT_DIR\\.claude\\hooks\\format-edited.ps1\"",
            "timeout": 60
          }
        ]
      }
    ]
  }
}
```

`.claude/hooks/format-edited.ps1`：

```powershell
$callInput = [Console]::In.ReadToEnd() | ConvertFrom-Json
$path = $callInput.tool_input.file_path

switch -Regex ($path) {
  '\.java$'                  { mvn -q spotless:apply "-DspotlessFiles=$path" }
  '\.(ts|tsx|js|json|md)$'   { npx prettier --write $path }
}
exit 0   # 格式化失败不阻断；要看结果就写日志或改成 exit 2
```

**成功时对话里什么都不显示** —— 这是设计如此，不是没跑。要确认就看文件是不是被改了，或者开 debug 日志。

> `npx` 在 Windows 上是 `.cmd` shim，只能走 shell form（这里没写 `args`，就是 shell form）。要改用 exec form 得写成 `{"command": "node", "args": ["...\\node_modules\\prettier\\bin\\prettier.cjs", "--write", "<路径>"]}`。

### 9.3 压缩后把关键约定塞回去

上下文被压缩成摘要，容易丢细节。用 `SessionStart` 的 `compact` matcher 在每次压缩后重新注入：

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "compact",
        "hooks": [
          {
            "type": "command",
            "shell": "powershell",
            "command": "Write-Output '提醒：包管理用 bun 不用 npm；提交前跑 bun test；当前迭代是鉴权重构。'"
          }
        ]
      }
    ]
  }
}
```

`SessionStart` 的纯 stdout 会进上下文，所以一句话的注入直接 `Write-Output` 就行，不用拼 JSON。想换成动态内容就把 `echo` 换成 `git log --oneline -5` 这类命令。

> 恒真不变的约定应该写进 `CLAUDE.md`（不跑脚本、加载更快）；这个位置适合放**会变**的东西。

### 9.4 需要你确认时弹系统通知

```json
{
  "hooks": {
    "Notification": [
      {
        "matcher": "permission_prompt|idle_prompt",
        "hooks": [
          {
            "type": "command",
            "shell": "powershell",
            "command": "& \"$env:CLAUDE_PROJECT_DIR\\.claude\\hooks\\notify.ps1\""
          }
        ]
      }
    ]
  }
}
```

`.claude/hooks/notify.ps1`：

```powershell
$callInput = [Console]::In.ReadToEnd() | ConvertFrom-Json
$msg = if ($callInput.message) { $callInput.message } else { '需要你的确认' }

# OSC 9：Windows Terminal / ConEmu / WezTerm 支持；BEL 结尾
$seq = "$([char]27)]9;$msg$([char]7)"
@{ terminalSequence = $seq } | ConvertTo-Json -Compress
```

**验证**：让 Claude 做一件需要批准的事，然后切走窗口，约 6 秒后应该收到通知。如果没出现，先在 Windows Terminal 里手动 `printf '\033]9;test\007'` 试一下你的终端认不认这个序列。

### 9.5 收尾前要求先跑测试

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "shell": "powershell",
            "command": "& \"$env:CLAUDE_PROJECT_DIR\\.claude\\hooks\\stop-check.ps1\""
          }
        ]
      }
    ]
  }
}
```

`.claude/hooks/stop-check.ps1`：

```powershell
$callInput = [Console]::In.ReadToEnd() | ConvertFrom-Json

# 已经在因为 stop hook 而继续了：直接放行，否则会来回死循环
if ($callInput.stop_hook_active) { exit 0 }

$msg = $callInput.last_assistant_message
if ($msg -notmatch '测试|test|单测') {
  @{
    hookSpecificOutput = @{
      hookEventName     = 'Stop'
      additionalContext = '还没看到测试结果。先跑一遍相关单测，再把结论写出来。'
    }
  } | ConvertTo-Json -Compress
}
exit 0
```

两个必须注意的点：

1. **`stop_hook_active` 守卫不能省**。不写的话，hook 让 Claude 继续 → Claude 又结束 → hook 又拦 → …… 直到撞上连续 8 次的上限，最后以一条警告收场。
2. **上面这个判断很粗**（就是看最后一条回复里有没有"测试"两个字）。真要用来管流程，改成检查某个标记文件、CI 结果或测试报告更靠谱，否则 Claude 随手写一句"我会补测试"就绕过去了。

### 9.6 长任务丢后台

`mvn test` 这种要跑几分钟的别让 hook 同步等：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "shell": "powershell",
            "command": "& \"$env:CLAUDE_PROJECT_DIR\\.claude\\hooks\\run-tests-async.ps1\"",
            "async": true
          }
        ]
      }
    ]
  }
}
```

`.claude/hooks/run-tests-async.ps1`：

```powershell
$callInput = [Console]::In.ReadToEnd() | ConvertFrom-Json
$path = $callInput.tool_input.file_path
if ($path -notmatch '\.java$') { exit 0 }

$output = & mvn -q test 2>&1 | Out-String
$msg = if ($LASTEXITCODE -eq 0) {
  "$path 改动后单测通过。"
} else {
  "$path 改动后单测失败：$($output.Substring(0, [Math]::Min(2000, $output.Length)))"
}

@{ hookSpecificOutput = @{ hookEventName = 'PostToolUse'; additionalContext = $msg } } | ConvertTo-Json -Compress
```

几条限制：

- `async` **只有 `command` 类型支持**，而且**不能阻断、不能控制行为**（`decision`、`permissionDecision`、`continue` 都无效）—— 它要控制的动作早就做完了。
- 结果**在下一轮对话时才送到**；会话闲着就得等你下次打字。
- 每次触发都起一个独立后台进程，不合并。
- `async` 的 hook **不受 `timeout` 约束**；`-p` 模式结束时会杀掉还在跑的后台 hook。
- 想让后台任务失败时**立刻叫醒** Claude，用 `asyncRewake: true`：它在退出码 2 时会立即把 stderr 作为 system reminder 推给 Claude（这个受 `timeout` 约束）。

---

## 10. Windows / PowerShell 注意

**1）`shell` 是逐个 hook 设的**

```json
{
  "type": "command",
  "shell": "powershell",
  "command": "Write-Host '文件已写入'"
}
```

不写 `shell` 就走 `sh -c`（Windows 上是 Git Bash；没装 Git Bash 才用 PowerShell）。设成 `"powershell"` 不需要开 `CLAUDE_CODE_USE_POWERSHELL_TOOL`，Claude Code 会直接拉起 PowerShell —— 优先 `pwsh.exe`（PowerShell 7+），没有才退回 `powershell.exe`（5.1）。**带了 `args`（exec form）时 `shell` 会被忽略。**

**2）路径占位符的三种写法，只有一种不会出错**

| 写法 | 结果 |
|---|---|
| `$env:CLAUDE_PROJECT_DIR` | ✅ 每个版本都对 |
| `${CLAUDE_PROJECT_DIR}` | ✅ v2.1.198 起会被改写成 PowerShell 的 `${env:...}` 形式。**注意**：它在双引号里能展开，在单引号里不行（单引号在 PowerShell 里就是不展开变量） |
| `$CLAUDE_PROJECT_DIR` | ❌ **会解析成 `$null`** —— PowerShell 把它当成未定义的局部变量，路径就缺了前缀。Claude Code 不做这个改写，只在 debug 日志里记一条警告 |

推荐的稳妥写法（shell form）：

```json
{
  "type": "command",
  "shell": "powershell",
  "command": "& \"$env:CLAUDE_PROJECT_DIR\\.claude\\hooks\\check.ps1\""
}
```

**3）文件路径是反斜杠的绝对路径**

`tool_input.file_path` 在 Windows 上**总是绝对路径、且用反斜杠**，即使 hook 跑在 Git Bash 下也一样。所以：

- 拿 `/src/` 这种正斜杠片段去比较，**永远匹配不上**，hook 会静默放行；
- 比较前先归一化：PowerShell 里 `-replace '\\', '/'`，bash 里 `FILE_PATH="${FILE_PATH//\\//}"`；
- 归一化之后按**路径片段**匹配（`/src/`），别用 `^` 锚开头 —— 它是绝对路径。

**4）启动 PowerShell 脚本的常规姿势**

```json
{
  "type": "command",
  "command": "powershell.exe",
  "args": ["-NoProfile", "-ExecutionPolicy", "Bypass", "-File", "${CLAUDE_PROJECT_DIR}/.claude/hooks/check.ps1"]
}
```

`-NoProfile` 让 hook 启动快一点（不加载你的 profile），`-ExecutionPolicy Bypass` 绕开执行策略限制。这是 **exec form**，`${CLAUDE_PROJECT_DIR}` 在 `args` 里是原样替换、不用加引号。想让公司执行策略生效就把 `-ExecutionPolicy Bypass` 去掉。

**5）`.cmd` / `.bat` 别指望 exec form**

`npx`、`eslint`、`tsc` 这些装在 `node_modules/.bin` 的是 `.cmd` shim，**不是可执行文件**，exec form 里 spawn 不起来。要么用 shell form，要么直接走 `node`：

```json
{ "command": "node", "args": ["${CLAUDE_PROJECT_DIR}/node_modules/typescript/bin/tsc", "--noEmit"] }
```

**6）脚本里有中文就必须存成 UTF-8 with BOM（本地实测）**

Claude Code 在 Windows 上优先用 `pwsh.exe`（PowerShell 7+），没有才退回 `powershell.exe`（5.1）。两者读 `.ps1` 的默认编码不一样：

- **PowerShell 7**：默认按 UTF-8 读，无 BOM 也正常。
- **Windows PowerShell 5.1**：无 BOM 时按系统 ANSI 码页读。脚本里的中文字符串会乱码，**严重的直接整个脚本解析失败**（本地实测：本文 7 段 PowerShell 示例存成无 BOM 的 UTF-8 后，5.1 下有 4 段报语法错误、另一段把中文写花了；同一批文件用 `pwsh` 7 跑全部正常）。

所以只要脚本里有中文（本文的例子都有），**存成 UTF-8 with BOM**：VS Code 里右下角编码 → `Save with Encoding` → `UTF-8 with BOM`。这样 5.1 和 7 都能正确读。

**7）依赖 `jq` 的例子**

官方文档里的 bash 例子大量用 `jq`。Windows 上要么 `winget install jqlang.jq` 装上，要么就用 PowerShell 的 `ConvertFrom-Json` / `ConvertTo-Json`（本文的 PowerShell 例子都是后者，没有额外依赖）。

---

## 11. 安全与边界

- **命令 Hook 以你的完整用户权限运行**，能改、能删、能读你账号能碰到的任何东西。加进配置前先自己跑一遍看看它到底干了什么。
- **workspace trust 的行为要记清楚**：
  - **交互式会话**：在你确认那个目录（或它的父目录）可信之前，**所有**设置文件里的 hook 都被扣住 —— 包括你自己的 `~/.claude/settings.json`。
  - **`claude -p` / SDK 会话**：**不弹信任框，直接把目录当成可信**。也就是说，克隆一个陌生仓库然后 `claude -p` 过去，它 `.claude/settings.json` 里提交的 hook 会直接在你机器上跑。
  - 所以：**对陌生仓库跑 `-p` 之前先看它的 `.claude/` 目录**，或者 `--settings '{"disableAllHooks": true}'` 关掉这一次的 hook。
- **`PreToolUse` 在权限模式判定之前触发**：返回 `deny` 在 `bypassPermissions` 模式下**也拦得住**（这是它作为策略闸门的价值所在）。但反过来不成立：返回 `allow` **不能**覆盖设置里的 deny 规则，也不能跳过标记了 `requiresUserInteraction` 的 MCP 工具确认框。**Hook 只能收紧，不能放宽。**
- **单向的拦截，不是完整的沙箱**：`PreToolUse` 覆盖不到 `@文件` 引用和 `EndConversation`；`PostToolUse` 根本撤销不了已经发生的副作用；hook 正则也拦不住精心构造的命令。要硬边界就用权限规则 + 沙箱，hook 只做额外一层。
- **密钥不进 hook 配置**：设置文件、Skill、hook 脚本里都不写 Token / API Key / 私钥，走环境变量。设置文件里的 hook 是明文，而且 `.claude/settings.json` 是**会被提交**的。
- **别把敏感内容写进日志**：hook 的 stdout 会进 debug 日志，`additionalContext` 会进 transcript。
- **关掉 hook**：设置里 `"disableAllHooks": true` 临时全关（`false` 写在项目级可以覆盖用户级的 `true`）；单次运行可以用 `claude --settings '{"disableAllHooks": true}'`，它优先级最高。**没有「关掉某一个 hook 但保留配置」的办法**，只能删掉那条。托管策略层设的 hook 只能由托管层关。
- **多人共用的仓库慎加阻断类 hook**：`exit 2` 的误判会直接卡住同事的活。团队共享的 hook 建议先只做记录（写日志），跑一段时间确认不误伤再改成阻断。

---

## 12. 排查

**先跑这几条：**

```text
/hooks                                    看实际注册了哪些 hook、来自哪个文件、命令是什么（只读）
claude --debug-file C:\temp\claude.log    把日志写到固定路径，另开一个窗口 tail
/debug                                    会话中途开启日志并看日志路径
```

`CLAUDE_CODE_DEBUG_LOG_LEVEL=verbose` 能看到更细的匹配日志（命中了几个 hook、matcher 比较结果）。会话记录视图 `Ctrl+O` 里能看到 hook 的运行结果：成功的话**什么都不显示**（除非它返回了 `systemMessage` 或 Stop 反馈）；阻断的话显示理由；非阻断错误显示一条 `<hook 名> hook error`。

**常见现象对照：**

| 现象 | 先查什么 |
|---|---|
| Hook 完全没触发 | `/hooks` 里有没有它、在不在**正确的事件**下；matcher 拼写（**大小写敏感**）；是不是搞错了事件（`PreToolUse` 在前、`PostToolUse` 在后）；非 tool 事件上写了 `if` 会导致**永不运行**（4.3 节） |
| `/hooks` 里一个 hook 都没有 | JSON 是不是合法（**不允许尾逗号、不允许注释**）；文件位置对不对（`.claude/settings.json` 是项目级、`~/.claude/settings.json` 是用户级）；文件监听失灵就重启会话 |
| 脚本改了没反应 | hook 配置文件改了通常会自动拾取；脚本文件本身改了不受影响（每次都是重新执行），但如果命令/路径变了要重启 |
| 输出 `PreToolUse hook error: ...` | 脚本意外非 0 退出。手工喂一段样例 JSON 试：`echo '{"tool_name":"Bash","tool_input":{"command":"ls"}}' \| ./my-hook.ps1`，然后看退出码 |
| 报 `command not found` / 路径不存在 | 用绝对路径或 `${CLAUDE_PROJECT_DIR}`；加 `"args": []` 切 exec form 可以完全避开 shell 引号问题 |
| 报 `jq: command not found` | 装 jq，或者改用 PowerShell 的 `ConvertFrom-Json` |
| **JSON 写了但不生效，也没有任何报错** | ① stdout 前面被别的东西污染了（profile 里的 `echo`），不再以 `{` 开头；② 字段**层级写错了** —— 比如 `permissionDecision` 必须放在 `hookSpecificOutput` 里，放顶层会被静默忽略。用 `claude --debug` 搜日志里的 `Hook JSON output had unrecognized keys` |
| 提示里带 JSON 校验/解析错误 | 说明 stdout 被当成 JSON 但没通过校验。别手拼字符串，用 `ConvertTo-Json` / `jq` 生成，让引号和反斜杠被正确转义（6.2 节） |
| Stop hook 反复不让停 | 没查 `stop_hook_active`（9.5 节）。连续 8 次后会被强制放行，要更多轮就设 `CLAUDE_CODE_STOP_HOOK_BLOCK_CAP` |
| Hook 跑到一半没了 | 撞上超时被取消，输出被丢弃。看 4.4 的默认值，长任务改 `async` 或调 `timeout` |
| Windows 上脚本里项目路径是空的 | 写了裸的 `$CLAUDE_PROJECT_DIR`，PowerShell 解析成 `$null`。改 `$env:CLAUDE_PROJECT_DIR`（第 10 节） |
| PowerShell 脚本报语法错误 / 中文变乱码 | `.ps1` 没存成 UTF-8 with BOM，5.1 按 ANSI 码页读了（第 10 节第 6 条） |
| 路径匹配怎么都不中 | `file_path` 是反斜杠的绝对路径，先归一化再比较（第 10 节） |
| 点了 `deny` 还是跑了 | 有别的 hook 返回了更宽松的决策？不会 —— `deny` 优先级最高，反过来是 `allow` 盖不住 deny 规则。更可能是脚本退出码不是 2（**`1` 不阻断**） |
| 陌生仓库里 `-p` 直接跑了它的 hook | 交互式会扣住、`-p` 不会。跑之前先看它的 `.claude/settings.json`，或 `--settings '{"disableAllHooks": true}'`（第 11 节） |

---

## 一页速查

```text
是什么      生命周期回调，由 harness 执行、不经过模型判断；必须每次都发生的放这里
            恒真的放 CLAUDE.md，按需的流程放 Skill，加工具的放 MCP

配在哪      ~/.claude/settings.json            用户级，所有项目
            <项目>/.claude/settings.json       项目级（可提交）
            <项目>/.claude/settings.local.json 本地，自动 gitignore
            插件 hooks/hooks.json              Skill / Subagent 的 frontmatter
            跨层是合并不是覆盖；disableAllHooks 关不掉托管层的
            ⚠️ 本仓库 .claude/ 在 .gitignore 里，写了不会提交

结构        事件 → matcher group → handler(type: command|http|mcp_tool|prompt|agent)
            matcher 只含字母数字 _- 空格 , | → 精确匹配；含其它字符 → 未锚定的正则
            （Edit.* 会匹配 Edit 和 NotebookEdit，要 ^Edit$ 才是整串）
            MCP 工具：mcp__<server>__<tool>，匹配整个 server 必须写 mcp__memory__.*
            if: "Bash(git *)" / "Edit(*.ts)" 权限规则语法，且只在 tool 事件上求值
            超时：command/http/mcp_tool 600s，UserPromptSubmit 等 30s，prompt 30s，
                  agent 60s，SessionEnd 共 1.5s（可抬到 60s）

输入        stdin 收 JSON：session_id / transcript_path / cwd / permission_mode
            （界面的 Manual 传过来是 default）/ hook_event_name / prompt_id
            工具事件加 tool_name / tool_input / tool_use_id；PostToolUse 加 tool_response
            ${CLAUDE_PROJECT_DIR}（会话启动时的项目根，worktree 里不变）
            $CLAUDE_ENV_FILE（只有 SessionStart/Setup/CwdChanged/FileChanged 有）

输出        退出码 0 = 无异议（≠ 批准）；2 = 阻断（JSON 也盖不住）；其它包括 1 = 不阻断
            stdout 必须 「以 { 开头、以 } 结尾」才是 JSON，否则当纯文本
            JSON 校验失败 / 解析失败 = 非阻断错误，动作照做
            通用字段：continue / stopReason / systemMessage / terminalSequence
            ⚠️ suppressOutput 官方明确「接受但无效果」
            顶层 decision: "block" —— UserPromptSubmit / PostToolUse / Stop / PreCompact 等
            hookSpecificOutput —— PreToolUse 的 permissionDecision: allow|deny|ask|defer
                                  + updatedInput；PermissionRequest 的 decision.behavior
            ⚠️ PreToolUse 的顶层 decision/reason 已废弃
            多个 hook 并行跑，决策取最严：deny > defer > ask > allow
            additionalContext / stdout 上限 10000 字符

常用事件    SessionStart   会话开始/恢复/压缩后；stdout 和 additionalContext 都进上下文
            UserPromptSubmit  提交提示词前，能拦；默认 30s，卡住会卡整个会话
            PreToolUse     工具调用前，allow/deny/ask/defer + 改写参数
                           在权限模式判定之前触发，能拦 bypassPermissions
                           但覆盖不到 @文件 引用（没有工具调用）
            PostToolUse    成功后，撤销不了任何东西；updatedToolOutput 只改 Claude 看到的
            Stop           每轮回复结束都触发；必查 stop_hook_active；连续 8 次强制放行
            PreCompact     能拦压缩；别无条件拦自动压缩
            SessionEnd     只能做副作用，共享 1.5s 预算
            Notification   要「立刻反应」用 PermissionRequest，permission_prompt 是等了 6s 才发

安全        hook 以你的完整用户权限运行；PreToolUse 能收紧不能放宽（allow 盖不住 deny 规则）
            交互式会话未确认信任时全部 hook 被扣住；claude -p 不弹信任框、直接当可信
            → 陌生仓库跑 -p 之前先看它的 .claude/
            密钥不进 hook 配置，走环境变量；别把敏感内容写进 stdout（会进 debug 日志）

排查        /hooks                     看注册了什么、来自哪个文件（只读）
            claude --debug-file <路径> 日志写固定路径，另开窗口 tail
            /debug                     会话中途开启日志
            Ctrl+O                     会话记录视图，看 hook 运行结果
            常见：JSON 不生效 → 先看 stdout 有没有被 profile 的 echo 污染、
                  字段层级对不对（permissionDecision 必须嵌在 hookSpecificOutput 里）
                  路径匹配不中 → file_path 是反斜杠绝对路径，先归一化
                  PowerShell 里路径为空 → 别写裸 $CLAUDE_PROJECT_DIR
                  策略 hook 没拦住 → 退出码得是 2，1 不阻断
                  .ps1 报语法错误/中文乱码 → 存成 UTF-8 with BOM
                  （5.1 读无 BOM 文件按 ANSI 解，详见第 10 节第 6 条）
```

官方文档：[Hooks 参考](https://docs.claude.com/en/docs/claude-code/hooks)、[Hooks 指南](https://docs.claude.com/en/docs/claude-code/hooks-guide)
