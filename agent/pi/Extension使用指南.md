# Pi Extension 使用指南

本文基于 Pi 0.87.x（Windows + PowerShell 环境实测）。API 签名逐个取自安装包内的 `dist/core/extensions/types.d.ts`，命令和默认值随版本变化，遇到不一致先执行 `pi --help`。

> ⚠️ 先说最要紧的一条：**扩展在 Pi 进程内执行，操作系统权限和 Pi 完全一样** —— 能读工作目录、能跑命令、能看会话历史和凭据。只加载你信得过的来源，详见第 9 节。

---

## 1. 扩展是什么，什么时候该用

扩展是给 Pi 加**可执行能力**的 TypeScript 模块：模型能调用的工具、`/` 命令、快捷键、CLI flag、生命周期事件钩子、模型 provider、终端 UI、会话状态。

Pi 的立场是"最小内核 + 一切靠扩展"，所以它给了一串从轻到重的机制，**优先用轻的**：

| 想解决的问题 | 用什么 | 为什么不用扩展 |
|---|---|---|
| 给某个目录加长期规则 | `AGENTS.md` | 不需要代码 |
| 把常用提示词变成 `/命令` | 提示词模板（`prompts/`） | 不需要代码 |
| 加任务专属说明和配套文件 | Skill（`skills/`） | 不需要代码 |
| 要加工具、拦工具、跑代码、加命令 | **扩展**（`extensions/`） | — |
| 接一个协议不兼容的模型服务 | 自定义 provider 扩展 | — |
| 打包分发上面这些 | Pi 包（`packages`） | — |

一句话判据：**"说明"能解决就用 `AGENTS.md` / 提示词模板 / Skill，"执行"才能解决才写扩展。**

一个扩展可以同时干很多事，但最实用的四类入口是：

- 加一个模型能调的工具（`registerTool`）
- 加一个你自己敲的 `/命令`（`registerCommand`）
- 在工具调用前拦截/改参数（`on("tool_call")`）
- 在会话节点上做副作用（`session_start` / `session_shutdown` / `agent_end`）

---

## 2. 放哪里、怎么加载

### 2.1 三个位置

| 位置 | 说明 |
|---|---|
| `<agent-dir>/extensions/`（默认 `~/.pi/agent/extensions/`） | 用户级，对所有项目生效 |
| `.pi/extensions/` | 项目级，**需要项目信任**（`/trust` 保存决定） |
| `settings.json` 的 `extensions` 数组 | 额外路径。用户设置里的相对路径按 agent 目录解析，项目设置里的按 `.pi` 目录解析；支持绝对路径和开头的 `~` |

单文件扩展直接放一个 `.ts`；多文件实现放一个目录，入口是 `index.ts` / `index.js`。要 npm 依赖就在旁边放 `package.json`。

Pi 用 jiti 加载，**本地 TypeScript 扩展不需要编译**，改完 `/reload` 就生效。

### 2.2 临时加载和临时关掉

```powershell
pi -e ./hello.ts             # 加载一个文件或目录，可重复
pi -e npm:@example/pi-tools  # 临时试一个包，不写进 settings
pi -ne                       # 关掉自动发现和配置里的扩展（显式 -e 的仍然加载）
pi -nt                       # 关掉所有内置 + 扩展 + 自定义工具
pi -e ./hello.ts -p "问题"    # 非交互模式也能加载
```

`-ne` 是排查"到底是哪个扩展在搞事"的主要手段（见第 8 节）。`-nt` 连扩展工具一起关掉。

### 2.3 settings.json 里的过滤写法

```json
{
  "extensions": [
    "extensions/*.ts",
    "!extensions/experimental-*.ts",
    "-extensions/legacy.ts"
  ]
}
```

规则：`!pattern` 排除 glob 匹配，`+path` 精确包含一个路径，`-path` 精确排除一个路径。用户级和项目级 settings 里的资源列表**都会加载**（项目级需要先过信任）。

### 2.4 生效方式

在会话里执行 `/reload` —— 它重载扩展、Skill、提示词模板、主题和上下文文件。**启动头部会列出本次实际加载了哪些资源和指令**，想知道"到底读进去没有"看那里最准。

> ⚠️ `/reload` 会替换掉整个扩展运行时。`await ctx.reload()` 之后的代码**不能复用旧运行时的状态**，需要会话级数据就在新上下文里重新取。

---

## 3. 最小可跑示例

三个例子覆盖 90% 的需求。每个都给了文件路径和验证方法。

### 3.1 加一个 `/hello` 命令

文件：`~/.pi/agent/extensions/hello-command.ts`

```typescript
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";

export default function (pi: ExtensionAPI) {
  pi.registerCommand("hello", {
    description: "打个招呼",
    handler: async (args, ctx) => {
      ctx.ui.notify(`Hello, ${args.trim() || "world"}!`, "info");
    },
  });
}
```

验证：重开 `pi` → 输入 `/hello 张三`。

要点：

- 工厂是 `export default function (pi: ExtensionAPI)`，**可以是 async**。async 工厂会被 await，所以启动期要拉配置或注册 provider 就写在里面。
- `handler` 的 `args` 是命令后面的原始字符串（含空格），不是数组。
- `ctx` 是 `ExtensionCommandContext`，比普通事件上下文多了会话控制方法（见第 4 节）。

### 3.2 加一个模型能调用的工具

文件：`~/.pi/agent/extensions/word-count.ts`

```typescript
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";
import { StringEnum } from "@earendil-works/pi-ai";
import { readFile } from "node:fs/promises";
import { resolve } from "node:path";
import { Type } from "typebox";

export default function (pi: ExtensionAPI) {
  pi.registerTool({
    name: "word_count",
    label: "Word Count",
    description: "统计一个文件的行数、单词数或字符数",
    promptSnippet: "word_count: 统计文件的行数/单词数/字符数",
    parameters: Type.Object({
      path: Type.String({ description: "文件路径，相对路径按工作目录解析" }),
      unit: StringEnum(["lines", "words", "chars"] as const),
    }),
    async execute(toolCallId, params, signal, onUpdate, ctx) {
      const file = resolve(ctx.cwd, params.path);
      const text = await readFile(file, "utf8");
      const stats = {
        lines: text.split(/\r?\n/).length,
        words: text.split(/\s+/).filter(Boolean).length,
        chars: text.length,
      };
      return {
        content: [{ type: "text", text: `${file}: ${params.unit} = ${stats[params.unit]}` }],
        details: stats,
      };
    },
  });
}
```

验证：`pi` → `用 word_count 工具统计 README.md 的单词数`。

要点（这几条踩了就很难查）：

- `parameters` 是 TypeBox schema。**字符串枚举必须用 `StringEnum`**（来自 `@earendil-works/pi-ai`），`Type.Union([Type.Literal(...)])` 在 Google 系 API 上不工作。
- 返回值**必须**有 `content`（给模型看的正文）和 `details`（给渲染和状态重建用）。没有结构化数据就写 `details: undefined`。
- 相对路径自己按 `ctx.cwd` 解析，工具不会替你换目录。
- `execute()` 里 `throw` 会变成一条失败的工具结果给模型；返回对象**不代表**失败。
- **同一个 assistant 消息里的多个工具调用可能并行执行**，不要假设"兄弟调用或结果一定存在"。共享内存状态（比如一个游戏光标）用 `executionMode: "sequential"`。
- 会改文件的工具，用 `withFileMutationQueue()` 把整段"读-改-写"包起来，否则并发下会互相覆盖。
- 大输出要截断，并告诉模型去哪读完整内容。
- **不写 `promptSnippet`，这个工具就不会出现在默认系统提示的 Available tools 段落里** —— 模型很可能不主动用。需要额外行为约束再加 `promptGuidelines`。
- 独立定义工具（赋值给变量、放进数组）时用 `defineTool()`，否则参数类型会被推宽成 `unknown`。
- 工具内部再调模型的话，把那段调用的 `usage` 一起返回，否则会话的 token 统计会少算。

### 3.3 拦一条危险命令

文件：`~/.pi/agent/extensions/confirm-destructive.ts`

```typescript
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";
import { isToolCallEventType } from "@earendil-works/pi-coding-agent";

export default function (pi: ExtensionAPI) {
  pi.on("tool_call", async (event, ctx) => {
    if (!isToolCallEventType("bash", event)) return;

    const cmd = event.input.command ?? "";
    if (!/\brm\s+-rf\b|\bsudo\b|\bformat\b/i.test(cmd)) return;

    if (!ctx.hasUI) return { block: true, reason: "非交互模式禁止破坏性命令" };

    const ok = await ctx.ui.confirm("危险命令", cmd);
    if (!ok) return { block: true, reason: "用户拒绝执行" };
  });
}
```

验证：让它执行 `rm -rf /tmp/whatever`，应该弹确认框而不是直接跑。

要点：

- `tool_call` 的返回类型是 `{ block?, reason?, terminate? }`。**要修改参数是原地改 `event.input`**，不是从返回值里给新参数。
- handler 抛异常 = 这条工具被**阻止**（fail-safe），不是放行。
- 内置工具事件用 `isToolCallEventType("bash" | "powershell" | "read" | "edit" | "write" | "grep" | "find" | "ls", event)` 收窄类型才有智能提示。
- 这只是"多一道防线"，**不是安全边界** —— 正则挡不住所有写法，真隔离靠容器（见第 9 节）。

---

## 4. API 速查表

工厂拿到的 `pi: ExtensionAPI`：

| 方法 | 用途 |
|---|---|
| `pi.on(event, handler)` | 订阅生命周期事件，返回取消订阅函数 |
| `pi.registerTool(toolDef)` | 注册模型可调用的工具 |
| `pi.registerCommand(name, { description?, getArgumentCompletions?, handler })` | 注册 `/` 命令 |
| `pi.registerShortcut(keyId, { description?, handler })` | 注册快捷键 |
| `pi.registerFlag(name, { type: "boolean" \| "string", default?, description? })` / `pi.getFlag(name)` | 注册/读取 CLI flag（会出现在 `pi --help` 里） |
| `pi.registerMessageRenderer(customType, renderer)` / `pi.registerEntryRenderer(customType, renderer)` | 自定义消息 / 会话条目的渲染 |
| `pi.registerMarkdownTransformer(fn)` | 渲染前改写用户和助手的 Markdown（只影响 TUI 显示） |
| `pi.sendMessage(msg, { triggerTurn?, deliverAs? })` | 发一条自定义消息（存进会话、参与上下文） |
| `pi.sendUserMessage(content, { deliverAs?, expandPromptTemplates? })` | 以用户身份发消息，**总会触发一轮** |
| `pi.appendEntry(customType, data?)` | 追加会话条目做状态持久化，**不进模型上下文** |
| `pi.setSessionName(name)` / `pi.getSessionName()` / `pi.setLabel(entryId, label?)` | 会话命名 / 打标签 |
| `pi.exec(command, args, options?)` | 执行外部命令 |
| `pi.getActiveTools()` / `pi.getAllTools()` / `pi.setActiveTools(names)` | 查看/切换当前生效的工具 |
| `pi.getCommands()` | 列出当前会话可用斜杠命令 |
| `pi.setModel(model)` → `Promise<boolean>` | 换会话模型（provider 没配好返回 `false`） |
| `pi.getThinkingLevel()` / `pi.setThinkingLevel(level)` | 读写思考等级 |
| `pi.registerProvider(...)` / `pi.unregisterProvider(name)` | 注册/覆盖模型 provider（初始加载期调用会排队，之后立即生效） |
| `pi.events` | 扩展之间通信的事件总线 |

`deliverAs` 取值 `"steer"`（改当前任务方向）/ `"followUp"`（追加后续任务）/ `"nextTurn"`（下一轮，仅 `sendMessage`）。

事件 handler 拿到的 `ctx: ExtensionContext`：

| 字段/方法 | 说明 |
|---|---|
| `ctx.ui` | 交互 UI（见下） |
| `ctx.mode` | `"tui" \| "rpc" \| "json" \| "print"` |
| `ctx.hasUI` | 有没有"能弹对话框"的客户端（TUI 和 RPC 为 true） |
| `ctx.cwd` | 工作目录 |
| `ctx.sessionManager` | 只读会话管理（`getBranch()` 取当前分支条目） |
| `ctx.modelRegistry` / `ctx.model` / `ctx.scopedModels` / `ctx.thinkingLevel` | 模型相关 |
| `ctx.isIdle()` / `ctx.hasPendingMessages()` / `ctx.signal` / `ctx.abort()` | 运行状态与控制 |
| `ctx.isProjectTrusted()` | 当前项目是否已信任 |
| `ctx.getContextUsage()` / `ctx.getSystemPrompt()` / `ctx.compact(options?)` | 上下文相关 |
| `ctx.shutdown()` | 请求有序退出 Pi |

`/命令` 的 handler 拿到的是 `ExtensionCommandContext`，**额外**有：

`getSystemPromptOptions()`、`waitForIdle()`、`newSession()`、`fork()`、`navigateTree()`、`switchSession()`、`reload()`。

> 这些方法**只能在命令里调** —— 从生命周期回调里调会死锁运行时。换会话会让旧 context 失效，切换前只保留纯数据，之后用 `withSession` 给的新 context。

`ctx.ui` 常用方法：

| 方法 | 用途 |
|---|---|
| `select(title, options, opts?)` / `confirm(title, message, opts?)` / `input(title, placeholder?, opts?)` | 对话框，分别返回字符串 / 布尔 / 字符串 |
| `notify(message, "info" \| "warning" \| "error")` | 通知 |
| `setStatus(key, text \| undefined)` | 底部状态栏文字 |
| `setWidget(key, content \| undefined, { placement })` | 编辑器上方/下方的小组件 |
| `setFooter(factory)` / `setHeader(factory)` / `setTitle(title)` | 页脚 / 页头 / 终端标题 |
| `setWorkingMessage(msg?)` / `setWorkingIndicator(opts?)` / `setHiddenThinkingLabel(label?)` | 流式过程中的提示与 thinking 折叠标签 |
| `custom(factory, { overlay?, overlayOptions?, onHandle? })` | 自带渲染和按键的组件（只支持 TUI） |
| `editor(title, prefill?)` / `setEditorText()` / `getEditorText()` / `pasteToEditor(text)` | 编辑器读写 |
| `addAutocompleteProvider(factory)` / `setEditorComponent(factory)` | 补全 / 自定义编辑器 |
| `theme` / `getTheme(name)` / `setTheme(name)` | 主题 |

provider 无关的嵌套模型调用用 `ctx.modelRegistry.streamSimple()`。

准确的事件、工具、结果类型以 `types.d.ts` 为准，别照着记忆写。

---

## 5. 生命周期事件清单

按用途分组（事件名就是 `pi.on()` 的第一个参数）：

**资源与信任**

- `project_trust` — 项目信任决定产生前触发。**只有用户级和命令行扩展能参与**。返回 `{ trusted: "yes" | "no" | "undecided", remember? }`。
- `resources_discover` — `session_start` 之后触发，允许扩展动态提供 skill / 提示词 / 主题路径。

**会话**

- `session_start` — 会话启动或恢复（重建状态写这里）。
- `session_info_changed`、`session_before_switch`、`session_before_fork`（返回 `{ cancel? }`）
- `session_before_compact`（返回 `{ cancel?, compaction? }` 可替换压缩结果）、`session_compact`、`session_compact_failed`
- `session_before_tree`、`session_tree`（会话树导航）
- `session_shutdown` — 释放资源写这里。

**上下文与 provider**

- `context` / `context_with_system` — 改写这一轮发给模型的 messages（返回 `{ messages }`）。`context` 不动 system 消息，Pi 用完会还原；`context_with_system` 只在需要自己掌控完整 transcript 时用，且必须保留 index 0 的 system 消息。
- `before_provider_request` / `before_provider_headers` / `after_provider_response` — 观察或改 provider 请求。
- `cache_warming_decision` — 覆盖空闲期 prompt cache 是否保持热：`{ action: "warm" }` 或 `{ action: "stop" }`，最后一个返回 action 的 handler 生效。

**Agent 与轮次**

- `before_agent_start` — 可追加一条自定义消息，或返回 `systemPrompt` 整体替换本轮系统提示（后续 handler 看到的都是替换后的文本）。
- `agent_start` / `agent_end`
- `turn_start` / `turn_end` — 可返回 `continue: true` 追加一次请求。
- `agent_before_settle` — **最后一个可动作的边界**，同样能 `continue: true` 做一次续跑。
- `agent_settled` — 最终状态，**只通知不能动**；集成需要知道"Pi 不会再自动继续了"时用这个。
- `ui_prompt_start` / `ui_prompt_end`

**消息**

- `message_start` / `message_update` / `message_end`（`message_end` 可返回替换后的 `message`，**必须保持原 role**）

**工具**

- `tool_execution_start` / `tool_execution_update` / `tool_execution_end` — 纯观察。
- `tool_call` — 拦、改参数、请求终止。
- `tool_result` — 改结果（`content` / `details` / `isError` / `usage`）；多个 handler 依次叠加，后面的能看到前面的改动。

**输入与 shell**

- `input` — 改写用户输入（`!{command}` 展开之类的技巧就在这）。
- `user_bash` — 接管 `!` 命令的执行。返回 `undefined` 交给下一个 handler 再交给本地执行；返回 `operations` 或 `result` 就终止传播；**handler 抛错会阻止这条命令**，不会落到本地执行。

**模型切换**

- `model_select`、`thinking_level_select`

### 几条硬规则

1. **不要在工厂里起进程、socket、watcher、定时器** —— 有些调用会加载扩展但不启动会话。长期资源放到 `session_start`，或放到真正用到它的命令/工具里。
2. 在 `session_shutdown` 里释放资源，并且**写成幂等**：取消、reload、换会话、进程退出都会汇聚到同一条清理路径，正常流程也不保证清理成功过。
3. `agent_end` 之后**仍可能有**自动重试、恢复、压缩或排队任务，别把它当"彻底结束"。要那个语义用 `agent_settled`。
4. `turn_end` / `agent_before_settle` 里**无条件 `return { continue: true }` 会死循环** —— 必须带条件。
5. handler 按扩展加载和注册顺序执行；`pi.on()` 返回取消订阅函数；**dispatch 进行中改注册不影响这一轮**。
6. 只有需要 `ctx.signal` 的嵌套工作才假设它在；命令和空闲期的会话事件通常没有 operation signal。

---

## 6. 状态与上下文

按"这个状态该怎么参与对话"来选存储：

| 状态性质 | 存哪 |
|---|---|
| 跟着当前分支走的工具状态 | 工具结果的 `details`（fork、回退自动跟着走） |
| 不想进模型上下文的持久数据 | `pi.appendEntry(customType, data)` |
| 要存下来并且发给模型的自定义内容 | `pi.sendMessage(...)` |
| 跨会话、跨项目的数据 | 外部存储（自己落盘） |

重建状态：在 `session_start` 里从 `ctx.sessionManager.getBranch()` 读当前分支的条目来重建。**不要遍历整个会话文件** —— 被放弃的分支代表另一条历史，照着它重建会串味。

自定义存储的内容要显示出来，就注册渲染器：消息用 `registerMessageRenderer`，会话条目用 `registerEntryRenderer`（自定义条目默认不进模型上下文）。

---

## 7. UI 与运行模式

Pi 有四种模式，**扩展在全部四种里都会加载**：

| 模式 | 怎么进 | UI 能力 |
|---|---|---|
| `tui` | 交互终端 | 完整终端 UI |
| `rpc` | `--mode rpc` | 支持 `select` / `confirm` / `input` / `notify` 等经 RPC Extension UI 协议转发，**不支持自定义终端组件** |
| `json` | `--mode json` | 没有 UI |
| `print` | `-p` | 没有 UI |

两条纪律：

1. **终端专属行为用 `ctx.mode === "tui"` 守**（自定义组件、裸终端输入、页脚页头这些）。
2. **对话框类交互用 `ctx.hasUI` 守**（TUI 和 RPC 都有）。不守的话，在 `pi -p` 或 CI 里 `ctx.ui.confirm()` 拿不到人回答，行为就不可预期了。
3. **工具和事件逻辑不要依赖渲染**，否则 JSON / print 模式直接坏掉。

`ctx.ui.custom()` 只在交互确实需要自己的渲染和按键处理时才用；组件、焦点、overlay、主题和性能细节看官方 `docs/tui.md`。

---

## 8. 调试与排查

### 8.1 几个开关

| 手段 | 作用 |
|---|---|
| `/reload` | 改完扩展/配置后重载 |
| 启动头部 | 本次实际加载了哪些资源 |
| `pi -ne` | 一次性关掉自动发现和配置的扩展（显式 `-e` 仍加载）—— 二分定位"是不是某个扩展干的" |
| `pi -nt` | 连扩展工具一起关掉，验证"工具消失后行为是否恢复" |
| `pi --help` | 顶部 help 会带上扩展注册的 long-form flag |
| `/debug` | 把终端渲染结果和会话消息写到 agent 目录的 `pi-debug.log` |
| `/session` | 看会话状态、上下文用量和费用 |

### 8.2 先知道报错怎么表现，别猜

- 事件 handler 出错：Pi 报告并**尽量继续**。
- `tool_call` handler 出错：这条工具被**阻止**（fail-safe）。
- 工具 `execute()` 出错：变成一条**失败的工具结果**给模型，Pi 不会崩。

### 8.3 症状对照

| 症状 | 常见原因 | 处置 |
|---|---|---|
| 扩展完全没加载 | 不在三个位置之一；`.pi/extensions` 没通过项目信任；被 `-ne` 关了 | 看启动头部 → `/trust` → 去掉 `-ne` |
| `/命令` 不在菜单里 | 注册时抛错；名字和已有命令冲突 | 看报错；换名字 |
| 模型不主动用这个工具 | 没写 `promptSnippet` | 补 `promptSnippet`，需要再加 `promptGuidelines` |
| 改了代码没生效 | 没 `/reload` | `/reload`；注意 reload 换掉整个运行时，旧状态不能复用 |
| 非交互模式下卡住或报错 | 在 json / print 模式里调了 UI | 用 `ctx.hasUI` / `ctx.mode === "tui"` 守 |
| fork 之后状态错乱 | 从全部会话条目重建了状态 | 改成从 `ctx.sessionManager.getBranch()` 重建 |
| 工具结果乱了/互相覆盖 | 并行执行共享了可变状态 | 该工具加 `executionMode: "sequential"`；文件改写包 `withFileMutationQueue()` |
| 模型不停继续、费用飙升 | `turn_end` / `agent_before_settle` 里无条件 `continue: true` | 给继续条件加上守卫 |

---

## 9. 安全

- **扩展 = Pi 进程内的代码 = 你的用户权限。** 它能读 `auth.json`、会话历史、工作目录里任何文件，能执行任何命令，能在你不知情时把上下文发到网络。装之前看代码，不是看它自称干什么。
- **项目本地扩展受"项目信任"闸门保护**，信任前不加载。有意思的是 `project_trust` 事件本身只允许**用户级和命令行**扩展参与 —— 项目扩展没有"自己批准自己"的机会。审查 `.pi/extensions/` 之后再给信任。
- `pi -na`（`--no-approve`）本次完全不加载信任闸门保护的项目本地资源，读别人的仓库时用。
- **不要把密钥、令牌写进扩展源码**，用环境变量。
- 扩展**不是安全边界**，也别指望它当沙箱。常规改动靠 git diff 审查；真要跑不可信或无人值守的任务，把 Pi 放进容器 / 虚拟机 / 沙箱，只暴露这次任务需要的目录、凭据和网络。
- 分发给别人时同理：包里的扩展能跑代码，包里的 skill 能让模型去跑程序。第三方包先看源码。

---

## 10. 分发：Pi 包

### 10.1 安装与管理

```powershell
pi install npm:@example/pi-tools@1.0.0
pi install git:github.com/example/pi-tools@v1
pi install ./local-package          # 本地路径，按解析后的绝对路径加载
pi install -l ./local-package       # 声明写进项目 .pi/settings.json（默认写用户级）
pi -e npm:@example/pi-tools         # 只这一次，不写进配置
pi list                             # 看已配置的包
pi config                           # TUI 里逐个开关资源，Tab 切 scope，--local 从项目覆盖开始
pi update --extensions              # 重新对齐已安装包的资源
pi remove <source>
```

npm 和 git 源都会固定版本（npm 带版本号、git 带 tag 或 commit）；反向的本地路径按声明它的 settings 文件所在目录解析。包声明写在项目 settings 里同样是**过了项目信任才读**。

### 10.2 做一个包

最简形式就是普通目录或 npm 包，用约定目录，Pi 自动发现 TypeScript / JavaScript 扩展、skill 目录、Markdown 提示词、JSON 主题：

```text
my-pi-package/
├── package.json
├── extensions/
├── skills/
├── prompts/
└── themes/
```

资源在别处就用 `package.json` 里的 `pi` 清单：

```json
{
  "name": "my-pi-package",
  "keywords": ["pi-package"],
  "pi": {
    "extensions": ["./src/extension.ts"],
    "skills": ["./resources/skills"],
    "prompts": ["./resources/prompts/*.md"],
    "themes": ["./resources/themes/*.json"]
  }
}
```

- `pi-package` keyword 让 npm 包能被 Pi 包画廊发现；可选的 `pi.image` / `pi.video` 提供预览。
- 运行时依赖放 `dependencies`。**Pi 提供的这几个包写 `peerDependencies` 且用 `"*"` 范围，不要打包**：`@earendil-works/pi-ai`、`@earendil-works/pi-agent-core`、`@earendil-works/pi-coding-agent`、`@earendil-works/pi-tui`、`typebox`。
- 每个包有独立的模块根：**不要指望两个包共用一个依赖实例**，也不要让一个包去解析另一个包没声明的依赖。
- 装包时可以用对象形式过滤要加载的资源（`!pattern` 排除、`+path` 精确包含、`-path` 精确排除）；过滤只能收窄包自己的声明，不能凭空引入资源。

---

## 11. 参考

**官方示例**（装完就在本地，按用途索引见该目录的 `README.md`）

```powershell
npm root -g     # 全局目录，示例在 <全局目录>\@earendil-works\pi-coding-agent\examples\extensions\
```

- 起步：`hello.ts`、`tools.ts`（交互式开关工具的 `/tools` 命令）
- 安全：`permission-gate.ts`、`protected-paths.ts`、`dirty-repo-guard.ts`、`project-trust.ts`、`sandbox/`、`gondolin/`
- 工具：`todo.ts`、`truncated-tool.ts`、`dynamic-tools.ts`（运行时启停工具）、`tool-override.ts`、`structured-output.ts`、`subagent/`
- 命令与 UI：`plan-mode/`、`preset.ts`、`status-line.ts`、`question.ts`、`custom-footer.ts`、`modal-editor.ts`
- 状态与渲染：`bookmark.ts`、`session-name.ts`、`message-renderer.ts`、`entry-renderer.ts`、`event-bus.ts`
- provider 与依赖：`custom-provider-anthropic/`、`custom-provider-gitlab-duo/`、`with-deps/`

**官方文档**（同一目录下的 `docs\`）

`extensions.md`（本文的一手依据）、`packages.md`、`custom-provider.md`、`tui.md`、`configuration.md`、`settings.md`、`security.md`、`rpc-extension-ui.md`

**类型声明**

`@earendil-works/pi-coding-agent` 的 `dist/core/extensions/types.d.ts` —— 事件、上下文、工具、结果类型的权威来源。文档和代码不一致时以它为准。

---

## 附录：一页速查

```text
位置      <agent-dir>/extensions/    默认 ~/.pi/agent/extensions/，对所有项目生效
          .pi/extensions/            项目级，需要项目信任
          settings.json "extensions": [...]   支持 !排除 +包含 -精确排除
加载      pi -e ./x.ts        pi -e npm:@x/y
关掉      pi -ne  关扩展     pi -nt  关全部工具（含扩展工具）
生效      /reload            启动头部看实际加载    pi --help 看扩展注册的 flag

骨架      export default function (pi: ExtensionAPI) { ... }    可以是 async
命令      pi.registerCommand("name", { description, handler: async (args, ctx) => {} })
工具      pi.registerTool({ name, label, description, promptSnippet,
                            parameters: Type.Object({...}),
                            async execute(id, params, signal, onUpdate, ctx) {
                              return { content: [...], details };
                            } })
钩子      pi.on("tool_call", (event, ctx) => ({ block: true, reason }))   // 改参数改 event.input
          pi.on("session_start" | "session_shutdown" | "before_agent_start"
                | "message_end" | "turn_end" | "input" | "user_bash" | ...)
状态      pi.appendEntry(type, data)   pi.sendMessage({...})   ctx.sessionManager.getBranch()
能力      pi.setActiveTools(names)  pi.setModel(m)  pi.setThinkingLevel(l)  pi.registerProvider(...)
UI        ctx.ui.notify/confirm/select/input/setStatus/setWidget/custom     ctx.hasUI  ctx.mode

注意      工厂里别起进程/定时器；session_shutdown 幂等释放；turn_end 无条件 continue 会死循环
          工具返回必须有 content + details；字符串枚举用 StringEnum
          改文件包 withFileMutationQueue()；共享状态用 executionMode: "sequential"
          非交互模式用 ctx.hasUI / ctx.mode 守 UI；agent_end 之后还可能有重试和压缩

安全      与 Pi 同权限执行；只装可信来源；项目扩展受项目信任闸门保护；真隔离靠容器

排查      /reload   pi -ne（二分定位）   pi -nt   /debug（pi-debug.log）   /session

文档      examples/extensions/README.md      docs/extensions.md      types.d.ts
```
