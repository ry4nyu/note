# Codex Hook 使用指南

本文介绍 Codex Hooks 的基本配置、常用场景和 Windows 下的最小实践。Hooks 的具体行为可能随 Codex 版本变化，遇到不一致时先查看官方文档和执行 `/hooks`。

## 1. Hook 是什么

Hook 是 Codex 生命周期中的回调。它可以在会话、用户输入、工具调用、上下文压缩或 Agent 结束等节点运行命令，或者调用已经连接的 MCP 工具。

常见用途：

- 会话开始时加载项目约定；
- 执行命令前阻止危险操作；
- 文件修改后运行检查；
- 会话结束时保存日志或清理临时文件；
- 在停止前要求 Codex 再做一次检查。

Hook 是辅助控制，不是完整的安全边界。真正的权限控制仍应使用沙箱、审批策略和操作系统权限。

## 2. 配置文件位置

Codex 会从各个配置层加载 Hook：

```text
用户级：~/.codex/hooks.json
用户级：~/.codex/config.toml
项目级：<项目>/.codex/hooks.json
项目级：<项目>/.codex/config.toml
```

本指南优先使用项目级 `.codex/hooks.json`，便于和项目一起版本控制。项目 Hook 只有在项目的 `.codex/` 配置层受信任时才会加载。

同一事件匹配到的多个 Hook 都会运行；多个命令 Hook 会并发启动，不能依赖其中一个 Hook 阻止其他 Hook 启动。一个配置层不要同时使用 `hooks.json` 和 `config.toml` 内嵌的 `[hooks]`，否则 Codex 会合并并给出警告。

## 3. 最小示例：会话开始时注入提示

### 3.1 创建 Hook 配置

在项目根目录创建 `.codex/hooks.json`：

```json
{
  "description": "项目生命周期 Hooks",
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup|resume|clear|compact",
        "hooks": [
          {
            "type": "command",
            "command": "python3 \"$(git rev-parse --show-toplevel)/.codex/hooks/session_start.py\"",
            "commandWindows": "py -3 \"$(git rev-parse --show-toplevel)/.codex/hooks/session_start.py\"",
            "statusMessage": "加载项目提示",
            "timeout": 10
          }
        ]
      }
    ]
  }
}
```

### 3.2 创建命令脚本

创建 `.codex/hooks/session_start.py`：

```python
import json
import sys

event = json.load(sys.stdin)
print(f"当前项目目录：{event['cwd']}")
print("修改文件前先阅读项目规则，并只做与需求直接相关的改动。")
```

命令 Hook 从 stdin 接收一个 JSON 对象。对于 `SessionStart`，脚本写到 stdout 的普通文本会作为额外开发者上下文传给 Codex。

### 3.3 审查并信任

启动 Codex 后执行：

```text
/hooks
```

在 Hook 列表中审查并信任新增或变更的 Hook。Codex 会按 Hook 定义的哈希记录信任状态；Hook 内容改变后需要重新审查。非托管 Hook 在未信任前会被跳过。

## 4. 常用生命周期事件

| 事件 | 触发时机 | 是否适合阻止操作 |
|---|---|---|
| `SessionStart` | 会话启动、恢复、清空或压缩后 | 否，主要用于注入上下文 |
| `SessionEnd` | 主会话结束 | 否，适合保存记录和清理 |
| `UserPromptSubmit` | 用户消息提交前 | 是，可阻止消息 |
| `PreToolUse` | 工具调用前 | 是，可拒绝或改写支持的调用 |
| `PermissionRequest` | Codex 即将请求权限时 | 是，可允许或拒绝请求 |
| `PostToolUse` | 工具调用完成后 | 不能撤销已发生的副作用 |
| `PreCompact` | 上下文压缩前 | 可阻止压缩 |
| `PostCompact` | 上下文压缩后 | 通常用于补充上下文 |
| `SubagentStart` | 子 Agent 启动时 | 不能阻止启动 |
| `SubagentStop` | 子 Agent 即将结束时 | 可要求继续处理 |
| `Stop` | 主 Agent 准备停止时 | 可要求继续处理 |
| `Interrupt` | 用户中断活动中的主会话时 | 不能阻止中断 |

## 5. Matcher 怎么写

`matcher` 是正则表达式，用于筛选事件。省略、写空字符串或写 `"*"` 表示匹配全部。

常见匹配值：

```text
工具调用：Bash
文件修改：apply_patch、Edit 或 Write
MCP 工具：mcp__filesystem__read_file
会话来源：startup|resume|clear|compact
压缩来源：manual|auto
```

`UserPromptSubmit`、`Stop` 和 `Interrupt` 当前不使用 matcher；配置了也不会进行筛选。

## 6. 工具调用前拦截危险命令

下面的配置让 `PreToolUse` 检查所有 Bash 调用：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "^Bash$",
        "hooks": [
          {
            "type": "command",
            "command": "python3 \"$(git rev-parse --show-toplevel)/.codex/hooks/pre_tool_use.py\"",
            "commandWindows": "py -3 \"$(git rev-parse --show-toplevel)/.codex/hooks/pre_tool_use.py\"",
            "statusMessage": "检查命令安全性",
            "timeout": 10
          }
        ]
      }
    ]
  }
}
```

脚本 `.codex/hooks/pre_tool_use.py`：

```python
import json
import re
import sys

event = json.load(sys.stdin)
command = event.get("tool_input", {}).get("command", "")

if re.search(r"\brm\s+-rf\b|\bformat\b", command, re.IGNORECASE):
    print(json.dumps({
        "hookSpecificOutput": {
            "hookEventName": "PreToolUse",
            "permissionDecision": "deny",
            "permissionDecisionReason": "命令包含项目策略禁止的破坏性操作。"
        }
    }))
```

`PreToolUse` 还可以返回 `permissionDecision: "allow"` 和 `updatedInput` 来改写支持的工具调用。Hook 只能作为额外防线；不要用简单正则替代沙箱和人工审批。

## 7. Hook 的输入和输出

所有命令 Hook 都从 stdin 接收一个 JSON 对象，常见字段包括：

```json
{
  "session_id": "会话 ID",
  "transcript_path": "会话记录路径或 null",
  "cwd": "会话工作目录",
  "hook_event_name": "当前事件名",
  "model": "当前模型",
  "permission_mode": "当前权限模式"
}
```

工具相关事件还会提供：

```json
{
  "tool_name": "Bash",
  "tool_use_id": "工具调用 ID",
  "tool_input": { "command": "待执行命令" },
  "tool_response": "工具输出"
}
```

输出规则：

- 普通文本是否生效取决于事件；例如 `SessionStart` 的 stdout 会注入上下文，而 `PreToolUse` 的普通文本会被忽略。
- 需要控制行为时输出 JSON；字段必须符合对应事件的格式。
- 对 `PreToolUse`、`UserPromptSubmit`、`PostToolUse` 等事件，退出码 `2` 并向 stderr 写入原因也可用于阻止或反馈。
- 不要把密码、Token、私钥或完整敏感会话内容写入日志、stdout 或 Hook 配置。

## 8. 超时、异步和 MCP Hook

- `timeout` 单位是秒；大多数 Hook 默认 600 秒。
- `SessionEnd` 和 `Interrupt` 默认 1 秒，允许的最大值是 3 秒。
- 设置 `async: true` 可让命令在后台运行，但 `SessionEnd` 始终同步执行。
- `mcp_tool` Hook 只能调用已经连接的 MCP Server，不会自动启动或重连 Server。
- `mcp_tool` Hook 使用结构化输入，支持 `${tool_input.file_path}` 这类事件字段占位符。
- `prompt` 和 `agent` 类型目前会被解析但跳过；优先使用 `command` 或 `mcp_tool`。

## 9. 常见问题

### Hook 没有运行

依次检查：

1. 配置文件是否位于当前生效的 `.codex/` 或用户级 `.codex/`；
2. JSON 或 TOML 格式是否正确；
3. 是否执行 `/hooks` 并信任了当前版本的 Hook；
4. `matcher` 是否匹配实际事件值；
5. Windows 下 `py -3`、脚本路径和脚本依赖是否可用；
6. 脚本是否从 stdin 读取 JSON，而不是等待交互输入。

### 项目 Hook 没有出现

确认项目 `.codex/` 配置层已受信任。未受信任项目不会加载项目本地 Hook，但仍可能加载用户级和系统级 Hook。

### Hook 改了但仍使用旧行为

Hook 内容改变后需要在 `/hooks` 中重新审查并信任。也可以重启 Codex 后再次检查 Hook 状态。

### 如何临时关闭 Hooks

在 `config.toml` 中设置：

```toml
[features]
hooks = false
```

`hooks` 是当前规范键名；`codex_hooks` 仅作为已弃用别名保留。

## 10. 建议

- 从一个事件、一个脚本开始，确认输入输出后再增加规则；
- 项目 Hook 使用 Git 根目录解析脚本路径，不要依赖启动 Codex 时所在的子目录；
- Hook 脚本保持短小、可审查、可重复执行；
- 将阻止规则放在 `PreToolUse`，将检查和提示放在 `PostToolUse`；
- 修改 Hook 后重新检查信任状态，并用一个无害命令做验证；
- 让 Hook 失败时给出明确原因，不要静默吞掉错误。

官方文档：[Codex Hooks](https://developers.openai.com/codex/hooks/)
