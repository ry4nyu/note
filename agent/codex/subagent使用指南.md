# Codex Subagent 使用指南

本文面向使用 Codex CLI 的团队成员，也适用于 Codex App 和 IDE 扩展。Subagent 是由主 Agent 委派出来、负责一个边界清晰任务的子 Agent；多个 Subagent 可以并行工作，主 Agent 最后汇总结果。

## 1. 什么时候使用

适合把任务拆成彼此独立的工作：

- 并行检查安全风险、测试缺口和可维护性；
- 分别探索前端、后端和数据库调用链；
- 同时分析多份日志、文档或测试失败信息；
- 让一个 Agent 只读调查，另一个 Agent 负责实现已确认的修复。

不适合拆分的情况：

- 任务很小，启动多个 Agent 的开销高于收益；
- 多个 Agent 会同时修改同一组文件；
- 后一个步骤必须等待前一个步骤的结论；
- 只是想让主 Agent 多想一会儿。

Subagent 会额外消耗 Token，因为每个子 Agent 都会独立进行模型推理和工具调用。先从两个或三个边界清晰的子任务开始。

## 2. 最简单的用法

直接告诉 Codex 如何拆分任务、是否等待全部结果，以及最终输出格式：

```text
并行检查当前分支相对 main 的改动：
1. 一个 subagent 检查安全风险；
2. 一个 subagent 检查测试缺口；
3. 一个 subagent 检查潜在行为回归。

等待全部结果后，按“严重程度、文件位置、原因、修复建议”汇总，不要直接修改文件。
```

实现类任务可以这样写：

```text
先让一个只读 subagent 定位登录失败的真实调用链，另一个 subagent 检查现有相关测试。
等两个结果返回后，由主 Agent 做最小修复并运行相关测试。不要让多个 Agent 同时修改同一文件。
```

任务提示词至少写清楚：

1. 每个 Subagent 的职责和边界；
2. 是否只读，是否允许修改文件；
3. 是否要等待全部结果；
4. 返回哪些证据，例如文件路径、符号、复现步骤和测试命令。

## 3. CLI 中查看和管理线程

在交互式 Codex CLI 中：

```text
/agent
```

`/agent` 用于查看和切换 Agent 线程。也可以直接告诉主 Agent：

```text
打开刚才负责安全检查的 subagent 线程，继续确认这个发现。
停止仍在运行的 subagent，并汇总已经完成的结果。
```

主线程负责需求、取舍和最终输出；Subagent 负责被委派的局部工作。不要把所有中间日志都复制回主线程，要求 Subagent 返回压缩后的结论和证据即可。

## 4. 内置 Agent

Codex 提供以下内置 Agent：

| 名称 | 用途 |
|---|---|
| `default` | 通用任务的默认 Agent |
| `worker` | 偏实现和修复的 Agent |
| `explorer` | 偏代码库阅读和调用链探索的 Agent |

通常不需要手动配置它们。只有当任务需要稳定的职责边界、不同的权限或不同的模型配置时，才创建自定义 Agent。

## 5. 自定义 Agent

### 5.1 文件位置

- 个人配置：`~/.codex/agents/`
- 项目配置：`.codex/agents/`

每个 TOML 文件定义一个 Agent。文件名可以与 Agent 名称相同，但真正的名称以 `name` 字段为准。

### 5.2 最小配置

```toml
name = "pr_explorer"
description = "只读探索代码库并整理 Pull Request 相关调用链。"
developer_instructions = """
保持只读。
定位入口、调用链和受影响文件，引用具体文件路径和符号。
不要修改代码，也不要在没有证据时提出确定性结论。
"""
```

这三个字段必填：

| 字段 | 作用 |
|---|---|
| `name` | Codex 调用或识别 Agent 时使用的名称 |
| `description` | 说明什么情况下应该使用它 |
| `developer_instructions` | 约束 Agent 的工作方式和输出格式 |

### 5.3 模型、权限和工具

自定义 Agent 可以覆盖常用会话配置，例如：

```toml
name = "reviewer"
description = "只读检查代码正确性、安全性和测试缺口。"
model = "gpt-6-sol"
model_reasoning_effort = "medium"
sandbox_mode = "read-only"
developer_instructions = """
优先报告真实的正确性、安全和回归风险。
每条发现都给出文件位置、影响、证据和最小修复方向。
不要修改文件，不要只报告代码风格问题。
"""
```

如果没有单独设置模型或推理强度，Subagent 默认继承主 Agent。自定义 Agent 中明确设置的值优先级更高；一次委派时显式指定的值优先级最高。

常见配置：

- `sandbox_mode = "read-only"`：适合探索、审查和文档核对；
- `sandbox_mode = "workspace-write"`：只给确实需要改文件的实现 Agent；
- `model`：为不同任务选择模型；
- `model_reasoning_effort`：在速度、Token 和推理深度之间取舍；
- `mcp_servers`、`skills.config`：只在该 Agent 确实需要额外工具或 Skill 时配置。

最小权限原则：只读任务使用只读沙箱；实现任务也只开放当前工作区，不要为了减少确认直接扩大权限。

## 6. 全局 Agent 设置

在 `config.toml` 中可以设置 Subagent 的全局默认行为：

```toml
[agents]
enabled = true
max_concurrent_threads_per_session = 4
default_subagent_model = "gpt-6-luna"
default_subagent_reasoning_effort = "high"
interrupt_message = true
```

字段说明：

| 字段 | 作用 |
|---|---|
| `agents.enabled` | 是否启用多 Agent 工具，默认启用 |
| `agents.max_concurrent_threads_per_session` | 单个会话最多同时运行的 Subagent 数量，不计主线程 |
| `agents.default_subagent_model` | 未单独指定时使用的模型 |
| `agents.default_subagent_reasoning_effort` | 未单独指定时使用的推理强度 |
| `agents.interrupt_message` | 是否把中断信息写入 Agent 可见上下文，默认启用 |

模型和推理强度只是默认值，不替代任务边界。轻量、重复、只读任务可以选择更快的模型；复杂的跨文件推理、实现和验证任务再使用更高推理强度。

## 7. 并行拆分的工作方式

推荐采用下面的顺序：

1. 主 Agent 明确目标、约束、验收标准和文件写入范围；
2. 并行启动互不依赖的只读调查；
3. 主 Agent 汇总并判断哪些结论有证据支持；
4. 只让一个 Agent 修改某一组文件；
5. 由主 Agent 检查 diff 并运行相关测试。

示例：

```text
调查“设置页保存失败”问题：
- code_mapper：只读定位前端入口、状态变化和后端接口调用；
- browser_debugger：只复现问题并记录控制台、网络请求和实际步骤；
- ui_fixer：等前两个结果后，再实现最小修复并运行相关测试。

三个 Agent 的写入范围不能重叠；最终由主线程汇总根因、修改文件和验证结果。
```

并行只适合独立任务。若任务存在严格先后关系，直接按顺序执行，避免为了“多 Agent”增加协调成本。

## 8. 权限、安全和结果检查

- Subagent 继承主会话的沙箱和审批策略；先在主会话选择正确的权限模式，再委派任务。
- CLI 中来自后台线程的审批请求可能在主线程显示；审批前确认请求来自哪个 Agent。
- 不要把密码、Token、私钥或内部敏感数据写入 Agent 配置和提示词。
- 并行写入同一文件容易产生冲突；将写入范围拆成互不重叠的文件，或只保留一个实现 Agent。
- 不要盲目接受 Subagent 的结论；要求文件位置、调用链、日志、复现步骤或测试结果等证据。
- 完成后仍要检查 `git diff`、`git status`，并运行与改动直接相关的测试。

## 9. 一页速查

```text
请求并行工作：  “spawn two agents / delegate this in parallel”
查看线程：      /agent
个人 Agent：     ~/.codex/agents/*.toml
项目 Agent：     .codex/agents/*.toml
全局配置：       [agents] in config.toml
只读调查：       sandbox_mode = "read-only"
实现修改：       sandbox_mode = "workspace-write"
完成检查：       git diff、git status、相关测试
```

官方文档：

- [Subagents](https://learn.chatgpt.com/docs/agent-configuration/subagents.md)
- [Codex 配置基础](https://learn.chatgpt.com/docs/config-file/config-basic)
- [权限与安全](https://learn.chatgpt.com/docs/agent-approvals-security.md)
