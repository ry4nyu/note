# Subagent

把一件边界清楚的事委派给**在自己的独立上下文窗口里**运行的子代理，只把结论收回主对话。

> 基于 Claude Code 2.1.x、Linux / WSL。字段表和内置 agent 清单来自本机 2.1.268 的 `claude --help` 和 `claude.exe` 里的 schema；表中英文描述是程序原文。什么时候该用 Subagent 见 [`README.md`](./README.md#扩展能力怎么选)。

## 1. Subagent 是什么

Subagent 由主对话委派出去，在**独立的上下文窗口**里干一件边界清楚的事。干完只把**最终结论**交回来，中间的 grep / read / 试错过程留在它自己那边，不占主对话的上下文。

它和 Skill 最本质的区别：Skill 是「教你该怎么干」，正文加载后**一直在主对话里占上下文**；Subagent 是「派个人去干」，**只把结论带回来**。

三条必须先记住的边界：

- **子代理不继承主对话的历史**。它拿到的是你（或主模型）写给它的那段 prompt，之前聊过什么它不知道。**该给的背景要显式写进委派提示词**。
- **子代理不能再往下委派**。它不能自己 spawn 子代理，拆分只能由主对话做。
- **结果要自己核**。子代理的结论是「另一个模型的判断」，不是事实。要求它给文件位置、调用链、复现步骤、测试命令，别直接采信。

## 2. 什么时候用

适合拆成子代理的活，共同点是**边界独立**：

- 并行检查安全风险、测试缺口、行为回归——三件互不依赖的事；
- 分别摸清前端、后端、数据库的调用链；
- 一次性读大量文件 / 大范围搜索，把成堆的文件内容挡在主对话之外；
- 固定角色的活：只做 code review 的、只写单测的、只做只读调查的。

不适合拆的：

- 任务很小，起一个子代理的协调开销高于收益；
- **多个子代理会改同一批文件**——并行写入必然冲突；
- 后一步必须等前一步的结论，直接按顺序做；
- 只是想让主模型「多想一会儿」——那该调 `effort`，不是拆代理。

**成本要说清楚**：每个子代理都是一次独立的模型推理 + 工具调用，**Token 是实打实多花的**。好处是用主对话的上下文换来的——主对话只装结论，不装过程。判断标准是：**这活的过程量是不是大到值得用一个独立窗口去装**。过程量大、结论短，才划算。

先从两三个边界清晰的子任务开始。

## 3. 从哪些位置加载

| 位置 | 作用范围 | 说明 |
|---|---|---|
| `~/.claude/agents/<名字>.md` | 用户级 | 对所有项目生效 |
| `<项目>/.claude/agents/<名字>.md` | 项目级 | 目录会**递归**扫描 |
| 已启用插件里的 `agents/<名字>.md` | 插件 | 名字带 `插件名:` 前缀 |
| `--agents '<json>'` | 单次会话 | 命令行直接给 JSON，不落盘 |

几条容易踩的：

- **`.claude/` 在本仓库的 `.gitignore` 里**。把子代理放进项目级 `.claude/agents/` **不会被提交**。要共享得先处理忽略规则，或走插件分发。
- **真正的名字以 frontmatter 的 `name` 为准**（这点和 Skill 相反——Skill 寻址用目录名）。手动敲名字调用时容易搞混，两处保持一致最省事。
- **同名会覆盖内置 agent**。想给内置的 `Explore` 换模型，写一个同名文件即可。
- **定义文件所在目录不受信任时，它的 `hooks` 和 `mcpServers` 会被静默跳过**，启动日志里是 `Skipping frontmatter ... the folder its definition file came from is not trusted`。这是安全设计，不是 bug。

### 3.1 内置 agent

不用自己定义也能用，主模型会按 `description` 自动挑：

| agentType | 模型 | 工具 | 用途 |
|---|---|---|---|
| `claude` | inherit | 全部 | Catch-all；未指定名字时的默认 |
| `general-purpose` | inherit | 全部 | 复杂多步任务；**委派时省略 `subagent_type` 就用它** |
| `Explore` | inherit | 只读 | 定位代码。**只「找到」，不做 review / audit**；可以让它按 `medium` 或 `very thorough` 的广度搜 |
| `Plan` | inherit | 只读 | 设计实现方案、识别关键文件和取舍，plan mode 里用 |
| `claude-code-guide` | **haiku** | Read / WebFetch / WebSearch | 回答 Claude Code、SDK、API 的用法问题 |
| `statusline-setup` | **sonnet** | Read / Edit | 配置状态栏 |
| `fork` | inherit | 全部，`maxTurns` 200 | 显式传 `subagent_type: "fork"` 时**继承完整对话上下文**；从不是默认选项 |

- 内置 agent 的默认模型：`claude-code-guide` 用 **haiku**，`statusline-setup` 用 **sonnet**，其余 `inherit`（跟随当前对话）。
- `Explore` 早期固定用 haiku，2.1.198 起改为继承主会话模型（有封顶）。想强制它用便宜模型，就写一个同名的用户级 agent 覆盖掉。
- 还有几个内部用途、明确「不面向直接 spawn」的（`workflow-subagent`、`comment-thread-analyst`、`coordinator` 等），列在列表里但没有手工使用的意义。
- `CLAUDE_AGENT_SDK_DISABLE_BUILTIN_AGENTS` 可以把内置 agent 整体关掉。

## 4. 写一个 Subagent

### 4.1 最小示例

放进 `~/.claude/agents/code-reviewer.md` 就能用：

```markdown
---
name: code-reviewer
description: 只读审查代码改动，找出正确性、安全和回归风险。当用户要求 review 代码、检查本次改动、提交前把关时使用。
tools: Read, Grep, Glob
model: inherit
---

你是代码审查员。只读，不要修改任何文件。

1. 先看 `git diff` 和 `git status`，确认审查范围是哪些文件。
2. 逐处检查：边界条件、错误处理、并发、资源释放、安全（注入、越权、密钥硬编码）。
3. 每条发现必须给出**文件路径 + 行号 + 触发条件 + 影响**。给不出触发条件的，不要报。
4. 只报真实缺陷，不报代码风格偏好。不确定的明确说「不确定」。
5. 用中文输出，按严重程度排序，不要开场白。
```

### 4.2 字段表

`name` 和 `description` **必填**，其余都能省。英文描述是程序里的原文：

| 字段 | 必填 | 说明 |
|---|---|---|
| `name` | ✅ | Agent identifier. Required —— Agent 工具和 `--agent` 用它寻址。 |
| `description` | ✅ | When to use this agent. Required —— 显示在 Agent 工具列表里。 |
| `model` | | Model override. `inherit` 跟随当前对话。别名 `haiku` / `sonnet` / `opus` / `fable` 或完整 ID。 |
| `tools` | | Tools available. **Replaces the default set.** —— 是**替换**，不是追加。 |
| `disallowedTools` | | Tools removed from the default set. **Ignored if `tools` is set.** |
| `effort` | | Thinking effort: `low`、`medium`、`high`、`xhigh`、`max`（部分版本也接受整数）。 |
| `permissionMode` | | Agent 运行时的权限模式，取值同 [`日常使用.md`](./日常使用.md) 第 4 节。 |
| `maxTurns` | | Maximum conversation turns before the agent stops. 防止跑飞的兜底。 |
| `skills` | | Skills preloaded for this agent. 想让子代理一上来就会用某个 Skill，写这里。 |
| `mcpServers` | | MCP servers to connect when this agent runs. 只给确实需要的子代理开。 |
| `hooks` | | Hooks registered while this agent runs. 写法见 [`Hook.md`](./Hook.md)；子代理场景还有 `SubagentStart` / `SubagentStop`。 |
| `memory` | | Memory scope: `user`、`project`、`local`。 |
| `background` | | If true, runs in the background by default. |
| `isolation` | | **`worktree` 在临时 git worktree 里跑。** 需要子代理改文件又怕污染工作区时用。 |
| `initialPrompt` | | Auto-submitted first message **when this agent runs as the main session**（`--agent` 或设置里选定）。**作为子代理 spawn 时不读。** |
| `color` | | 程序里标为 `@internal`，面向 agents UI 的显示色。 |
| `observer` / `observerMessage` | | 标为 `@internal`，用不到。 |

`tools` 写法两种都行：逗号分隔字符串（`tools: Read, Grep, Glob`）或 YAML 列表。

### 4.3 硬规则

- **`tools` 是替换不是追加。** 写了 `tools: Read, Grep, Glob`，这个子代理就**只剩**这三个——想让它跑测试，得把 `Bash` 也写上。
- **`tools` 和 `disallowedTools` 别一起写**，后者会被忽略。要么白名单，要么黑名单，二选一。
- **`name` 不能含 `:`**，冒号是插件命名空间的保留符号（`插件名:agent名`）。
- **`description` 缺失或没写清，主模型就不会正确委派**——和 Skill 的 `description` 是同一类问题。
- **`initialPrompt` 在子代理场景下不生效**，它是「这个 agent 当主会话跑」时用的。

### 4.4 `description` 怎么写

`description` 是主模型判断「要不要派给它」的**唯一依据**，标准就是程序里的原话：**When to use this agent**。要写**什么时候用**，不是写它是什么。

```yaml
# 不行：只是个标签，主模型判断不出该不该派
description: 代码审查代理

# 可以：写清触发场景，把同事可能说的原话写进去
description: 只读审查代码改动，找出正确性、安全和回归风险。当用户要求 review 代码、
  检查本次改动、提交前把关时使用。
```

模型在这件事上偏保守（该用的时候想不起来用）。**把用户可能说的原话写进描述，比抽象概括有效。**

### 4.5 正文怎么写

- 用**命令式**写，并且**解释为什么**。堆 `ALWAYS` / `MUST` 通常不如把道理讲清楚。
- **显式写清边界**：只读还是可写、允许改哪些路径、哪些东西不许碰。子代理没有主对话的那些隐性约束。
- **规定输出格式和证据要求**：要文件路径、行号、复现步骤、命令。这是子代理最容易糊弄的地方。
- **别把所有中间日志交回来**——那等于没省上下文。要求它返回压缩后的结论 + 证据。

## 5. 两个完整示例

> 下面两个都**只是文档里的代码块，本仓库不会落盘**——`.claude/` 在 `.gitignore` 里。要用就复制到自己的 `~/.claude/agents/`。

### 5.1 只读审查：`code-reviewer`

```markdown
---
name: code-reviewer
description: 只读审查代码改动，找出正确性、安全和回归风险。当用户要求 review 代码、
  检查本次改动、提交前把关时使用。
tools: Read, Grep, Glob, Bash
model: inherit
maxTurns: 30
---

你是代码审查员。**只读** —— 不要修改、创建或删除任何文件。

1. 先用 `git status` 和 `git diff` 确定审查范围，不要凭想象扩大范围。
2. 逐处检查：边界条件、错误处理、空值、并发、资源释放、异常吞掉、安全（注入、
   越权、密钥硬编码）、以及**这次改动可能破坏的既有行为**。
3. 每条发现给出：**文件路径 + 行号 + 触发条件 + 影响**。给不出具体触发条件的
   不要报 —— 那说明你还没确认。
4. 只报真实缺陷，不报代码风格偏好。不确定的明确标注「不确定」。
5. 用中文输出，按严重程度排序。不要开场白，不要复述本指令。
```

关键点：`tools` 里给了 `Bash`（要跑 `git diff`）但**没有** `Edit` / `Write`，从工具层面保证它改不了东西——比在提示词里写「请不要修改」可靠。

### 5.2 跨文件调查：`call-chain-explorer`

```markdown
---
name: call-chain-explorer
description: 只读追踪跨文件、跨模块的调用链和数据流，定位入口、经过的函数和落库点。
  当需要搞清楚"某个功能到底怎么跑起来的"、或大范围搜索但不确定关键词时使用。
tools: Read, Grep, Glob
model: inherit
background: false
---

你负责把一条调用链摸清楚，只读。

1. 先找**入口**（Controller / 路由 / 消息消费者 / 定时任务），再顺着往下走。
2. 每一步都引用**具体文件路径 + 符号名**，不要用"某处""大概在 service 层"这种说法。
3. 区分**实际调用**和**动态分发**（回调、事件、AOP、反射）—— 后者要说明你是怎么确认的。
4. 输出一条有序链路：入口 → 中间层 → 落库/外部调用，每步一句话说明做了什么。
5. 找不到的地方明确说"未找到"，**不要用推测补齐链路**。
```

关键点：`background: false` 让主对话等结果（调查型任务通常下一步就要用）；正文里「区分实际调用和动态分发」和「不要用推测补齐链路」针对子代理最容易犯的两个错——**把约定当事实、把没找到的地方用想象补上**。

## 6. 怎么调用

| 途径 | 说明 |
|---|---|
| 自动委派 | 你的说法命中某个子代理的 `description` 时，主模型自己去派 |
| 显式点名 | 提示词里说清用哪个（「用一个只读子代理去查…」），或直接给 `subagent_type` |
| `--agent <名字>` | 让某个 agent **作为当前主会话**运行（不是 spawn 成子代理）；此时 `initialPrompt` 才会被读 |
| `--agents '<json>'` | 单次会话直接给 JSON 定义，键是 agent 名 |

几条实测口径：

- **不指定 `subagent_type` 时，默认用 `general-purpose`。**
- **委派时可以临时覆盖模型**：Agent 工具带一个 `model` 参数，接受 `sonnet` / `opus` / `haiku` / `fable` 别名或完整 ID，**优先级高于定义文件里的 `model`**。唯一例外是 `fork`——它忽略 `model` 覆盖。
- **`fork` 是特殊选项**：显式传 `subagent_type: "fork"` 时子代理**继承完整对话上下文**（其它子代理都不继承）。它从不是默认，必须点名。
- **`background` 控制后台还是当前轮等**：设 `true`，默认后台跑、完成后用任务通知回报；设 `false` 则当前轮等结果。
- **可以并行**：多个子代理在一条消息里同时发起就会并发执行。
- **别重复劳动**：把搜索委派出去之后，主对话**不要再自己搜一遍同样的东西**。这是最常见的浪费。
- **`--forward-subagent-text`**：配合 `claude -p --output-format stream-json` 时，把子代理的文本和思考块也转发出来，便于在 CI / 日志里观察。

> **别和 `claude agents` 搞混**：`claude agents` 管理的是**后台会话**（`claude --bg` 起的那些，用 `claude attach <id>` 接回来），和这里的子代理是两回事。

## 7. 安全、排错与结果检查

- **子代理继承主会话的权限与审批策略**。先把主会话的权限模式调对，再委派；不要为了让它少问几次而放宽权限。
- **并行写入同一文件会冲突**。要么把写入范围拆成互不重叠的文件，要么**只保留一个**有写权限的子代理。
- **`--agents` 在 safe mode 下会被忽略**（日志：`--agents: ignored in safe mode (user-supplied custom agents are disabled)`）。
- **不受信任的目录，其定义的 `hooks` / `mcpServers` 会被跳过**。
- **不要把密码、Token、私钥写进 agent 定义或委派提示词**。agent 文件是明文，而且可能被提交或分享。
- **不要盲信子代理的结论**。要求文件位置、调用链、日志、复现步骤或测试结果；给不出证据的按「未验证」处理。
- **完成后自己收口**：检查 `git diff` / `git status`，跑与改动直接相关的测试。子代理说「改好了」不等于改好了。

**常见现象对照：**

| 现象 | 先查什么 |
|---|---|
| 我定义的 agent 不出现在列表里 | 路径是不是 `~/.claude/agents/<名字>.md` 或 `<项目>/.claude/agents/<名字>.md`；`name` 是否含 `:` |
| 定义了但从不被自动委派 | `description` 没写清「什么时候用」；先在提示词里显式点名，排除描述因素 |
| 子代理说改不了文件 | `tools` 是**替换**语义，是不是漏写了 `Edit` / `Write` |
| 同时写了 `tools` 和 `disallowedTools`，屏蔽没生效 | `disallowedTools` 在 `tools` 存在时被忽略，二选一 |
| 定义的 `hooks` 不生效 | 定义文件所在目录不受信任，被静默跳过 |
| 子代理「不知道」前面聊过什么 | 预期行为——它不继承主对话历史，背景要写进委派提示词 |
| `--agents` 里给的 agent 不见了 | 会话是不是 safe mode |
| 子代理结论看着对但落到代码上不对 | 要求它给证据（路径 / 行号 / 复现命令），给不出的重新派 |

> **关于 `/agents`**：交互式向导已在 **2.1.198 移除**。现在两条路：**直接让 Claude 帮你写 / 改** `~/.claude/agents/*.md`，或者**自己手写**那个 Markdown 文件。

## 8. 团队规范

- 先看内置 agent（3.1 节）能不能用，再考虑自己定义。
- 一个 agent 只解决一类清晰问题，不把 review、实现、验证揉成一个。
- **只读任务就用 `tools` 白名单把写工具摘掉**，不要只在提示词里写「请不要修改」。
- 不给子代理开放超出任务需要的工具和 MCP server。
- 不在 agent 定义或委派提示词里写密码、Token、私钥。
- 需要改文件时，明确修改范围、不要触碰的内容和验证命令；**并行时写入范围不能重叠**。
- 内部用途、写明「不面向直接 spawn」的内置 agent 不要手工调用。

推荐的任务提示词：

```text
并行调查当前分支相对 main 的改动：
1. 一个只读子代理查正确性和行为回归；
2. 一个只读子代理查测试缺口；
3. 一个只读子代理查安全风险。

三个都只读，不要修改文件。等全部结果返回后，按"严重程度、文件位置、原因、
修复建议"汇总，并标注每条结论有没有证据支撑。
```

**提交前检查：**

- 是否复用了内置 agent 或已有定义？
- `description` 能不能让主模型正确判断「该派 / 不该派」？
- `tools` 是不是最小必要集？只读任务有没有把写工具摘干净？
- 输出的证据要求（路径 / 行号 / 复现步骤）写清楚了吗？
- 是否包含最小权限和敏感信息约束？
- 是否只改动了完成任务所需的文件？

## 一页速查

```text
是什么      独立上下文窗口的子代理；只把结论交回主对话
            不继承主对话历史、不能再往下派、结果要自己核

位置        ~/.claude/agents/<名字>.md        用户级，所有项目
            <项目>/.claude/agents/<名字>.md   项目级（递归扫描）
            插件 agents/<名字>.md             带 插件名: 前缀
            --agents '<json>'                 单次会话，不落盘
            ⚠️ 本仓库 .claude/ 在 .gitignore 里，放进去不会提交

字段        name / description  必填（description = When to use）
            model               inherit | haiku | sonnet | opus | fable | 完整 ID
            tools               白名单，替换默认集合（要跑测试别忘了 Bash）
            disallowedTools     黑名单，tools 存在时被忽略
            effort maxTurns skills mcpServers hooks memory background
            isolation: worktree 用临时 git worktree 隔离文件改动
            initialPrompt       只在 agent 当主会话跑时生效，子代理不读

内置        claude / general-purpose（省略 subagent_type 时用这个）
            Explore / Plan（只读）  claude-code-guide（haiku）  statusline-setup（sonnet）
            fork（显式点名，继承完整上下文）

调用        自动按 description 委派 | 提示词里显式点名 | subagent_type 指定
            background: true 后台跑 / false 当前轮等结果
            委派时的 model 参数 > 定义里的 model（fork 例外，忽略覆盖）
            可并行；委派出去后自己不要再搜一遍同样的东西

排查        claude --help                      看 --agent / --agents 口径
            claude agents / attach <id>        管后台会话（≠ 子代理，别混）
            safe mode 下 --agents 被忽略
            目录不受信任时 hooks / mcpServers 被静默跳过
            /agents 向导已在 2.1.198 移除，改为直接编辑 .claude/agents/*.md

完成检查    git diff、git status、相关测试；子代理的结论要有证据
```

官方文档：[Create custom subagents](https://code.claude.com/docs/en/sub-agents)
