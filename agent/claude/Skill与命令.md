# Skill 与命令

把一个任务的**流程、检查单和附带文件**打包成可复用能力。自定义斜杠命令是它的轻量形态，也在本篇。

> 基于 Claude Code 2.1.x、Linux / WSL。参数占位、`!` 注入、命令文件与 Skill 的同名优先级在本机 2.1.268 上用一次性测试项目实测过；其余按官方文档口径理解。版本变化后先看 `/help` 和 `claude --help`。「什么时候该用 Skill」见 [`README.md`](./README.md#扩展能力怎么选)。

## 1. Skill 是什么

Skill 是一组可复用的 Agent 工作规则，用来固定某类任务的触发条件、执行顺序、安全边界和验证方式。一个 Skill 就是一个包含 `SKILL.md` 的目录：

```text
pdf-tools/
├─ SKILL.md          # 必需：frontmatter + 正文指令
├─ scripts/          # 可选：可执行脚本，确定性的重复逻辑放这里
├─ references/       # 可选：按需读取的参考资料
└─ assets/           # 可选：输出要用的模板、图标、字体
```

关键是**三层按需加载**：

| 层级 | 什么时候进上下文 | 成本 |
|---|---|---|
| `name` + `description` | 一直在，每一轮都可见 | 一行描述，基本可以忽略 |
| `SKILL.md` 正文 | 命中触发条件、模型决定加载时 | 正文多长就多少 token |
| `references/`、`scripts/` 等 | 模型真正去读的时候 | 不读就不花钱 |

所以装一堆 Skill 但不触发，和不装差不多。两条实测细节：

- `SKILL.md` 正文**上限 1 MB**，超过直接报错不加载。
- 正文一旦加载，**本轮对话里会一直占着上下文**；只有 `references/` 里的文件随读随丢。这也是正文要写短的原因。

Skill 不提供新工具，它提供的是「这类活该怎么干」的流程知识。

## 2. 什么时候用

适合：重复出现、需要稳定流程的工作（怎么补单测、怎么发版、怎么审这类 PR）。

不适合：一次性的小任务不值得为了「走流程」额外加步骤；需要独立上下文或并行的活用 [Subagent](./Subagent.md)；要给模型加工具用 [MCP](./MCP.md)。

> 本仓库另有一套 `.agents/skills/`（`agent-perf`、`github-cli`、`kit-manage`），那是 Codex 和 Pi 的加载位置，**Claude Code 不读**。要在 Claude Code 里用，得放到第 3 节列的位置。

## 3. 从哪些位置加载

| 位置 | 作用范围 | 说明 |
|---|---|---|
| `~/.claude/skills/<名字>/SKILL.md` | 用户级 | 对所有项目生效 |
| `<项目>/.claude/skills/<名字>/SKILL.md` | 项目级 | **项目根和当前工作目录**都会扫 |
| 已启用插件里的 `skills/<名字>/SKILL.md` | 插件 | 名字带 `插件名:` 前缀 |
| `--plugin-dir <路径>` | 单次会话 | 本地调试插件用 |

几条容易踩的：

- **目录名就是 Skill 名**，也是 `/命令` 名。frontmatter 里的 `name` 靠自觉保持一致，写错不报错。
- 用户级路径跟着 `CLAUDE_CONFIG_DIR` 走（配了这个变量，`~/.claude/` 整个搬走）。
- 同一条路径既被用户级加载又被插件提供时，**插件那份会被跳过**。
- **同名不会后者覆盖前者**：不同来源可以同名并存，列表按名字成组，`skillOverrides` 也按名字整组生效。
- 企业策略里如果 `strictPluginOnlyCustomization` 带了 `"skills"`，用户级 / 项目级目录会被整体锁掉。

### 3.1 命令文件：`commands/<名字>.md`

自定义斜杠命令不是另一条技术路线，而是 **Skill 的轻量形态**：一个 Markdown 文件就是一条 `/名字`，文件名即命令名，正文就是展开后发给模型的提示词。两者共用同一套 frontmatter 和参数占位语法。

| 位置 | 作用范围 | 能提交给同事吗 |
|---|---|---|
| `~/.claude/commands/<名字>.md` | 所有项目 | 否 |
| `<项目>/.claude/commands/<名字>.md` | 单个项目 | 能 |
| 插件里的 `commands/<名字>.md` | 插件启用期间 | 能 |

- **优先级**：项目 > 个人 > 插件；低的同名那份被**静默忽略**。
- **名字来自路径**：`commands/sub/t2.md` → `/sub:t2`，子目录用 `:` 拼命名空间。frontmatter 的 `name` 不参与命名。
- **同名时 Skill 赢**：`commands/dup.md` 和 `skills/dup/SKILL.md` 并存时，`/dup` 用的是 Skill，命令那份没有任何提示就没了。所以「菜单里能看到 `/dup`」不等于你写的那份在生效。
- **升级动作**：`commands/review.md` → `skills/review/SKILL.md`，目录名即命令名，内容照搬，frontmatter 和参数语法都不用改。**别两边各留一份**——一定会漂移，而且命令那份会被静默忽略。

一句话：**一段提示词用命令文件，有流程、检查单、附带文件就该升级成 Skill。**

## 4. 写一个 Skill

### 4.1 目录与 frontmatter

`SKILL.md` 以 YAML frontmatter 开头，正文直接写指令。真正影响能否被触发的是 `description`，其余都能省：

| 字段 | 作用 |
|---|---|
| `name` | Skill 名，也是 `/命令` 名，通常与目录同名 |
| `description` | 一行摘要，**决定模型什么时候调用它** |
| `when_to_use` | 补充「什么场景该用它」，会并进 Skill 工具的描述 |
| `model` | 用哪个模型跑：`haiku` / `sonnet` / `opus` / `fable` 或完整 ID，`inherit` 跟随当前对话 |
| `allowed-tools` | 本文件生效期间给模型开的工具，逗号分隔字符串或 YAML 列表 |
| `disallowed-tools` | 本文件生效期间收回的工具；你发下一条消息时失效 |
| `argument-hint` | `/命令` 后面显示的占位提示，如 `<file>` |
| `disable-model-invocation` | `true`：模型不能自动调用，只有你能敲 `/名字` |
| `user-invocable` | `false`：对你隐藏 `/命令`，只有模型能调 |
| `effort` | 思考强度：`low` / `medium` / `high` / `xhigh` / `max` |
| `context` | `inline`（默认，展开在当前对话）或 `fork`（另起一个子 agent） |
| `agent` | `context: fork` 时用哪种子 agent |
| `background` | 只在 `context: fork` 下有效：fork 出来的子 agent 后台跑，完成后用任务通知回报 |
| `globs` | Glob 列表：只有模型碰到匹配文件时，这个 Skill 才加载 |
| `hooks` | 本 Skill 生效期间注册的 Hook，写法同 `settings.json` |
| `shell` | 正文里 `!` 命令块用哪个 shell（`bash` / `powershell`），默认 bash |
| `metadata` | 作者自用的自由键值，加载后原样保留 |

### 4.2 硬规则

- **寻址用的是目录名，不是 frontmatter 里的 `name`**。名称不能含括号、逗号和控制字符，不能有前后空格，不能以 `/` 开头，不能用 `*` 通配。
- **目录名和 `name` 不一致时没有任何告警**。目录 `foo/` 里写 `name: bar`，`/plugin validate` 照样通过，列表里显示 `foo`。只能靠纪律保持一致。
- **frontmatter 必须是键值映射**。写成数组时 `/plugin validate` 会报 `Frontmatter must be a YAML mapping (key: value pairs), got an array`，**但 Skill 依然会被加载**，只是 frontmatter 全丢、描述退化成正文里的文本。
- **缺 `description` 只是 warning**，Skill 照样出现在列表里，描述退化成正文内容。触发基本靠运气，等于没写。

> 三条合起来是同一个坑：**「列表里能看到」不等于「写对了」**。验收看 `/plugin validate` 有没有报错，而不是看 Skill 有没有出现。
>
> 引用附带文件时用**相对 Skill 目录的路径**，别依赖当前工作目录（用 `${CLAUDE_SKILL_DIR}`）。

### 4.3 `description` 怎么写才触发得上

`description` 是唯一的自动路由依据，必须同时说清**做什么**和**什么时候用**。

模型在这件事上偏保守——该用的时候经常想不起来（官方 skill-creator 管这叫 undertrigger）。所以宁可写得主动一点：把同事可能说的**原话**写进描述，比抽象概括有效。

```yaml
# 不行：给不出任何路由信息
description: Helps with documents.

# 可以
description: 把 Markdown 转成公司模板的 Word 文档。当用户提到「导出 Word」「转成 docx」
  「要一份能发出去的正式文档」时使用。
```

### 4.4 正文怎么写

- 用**命令式**写（「先读 X，再改 Y」），并且**解释为什么**。堆 `ALWAYS` / `MUST` 通常不如把道理说清楚。
- `SKILL.md` 只放流程、判断规则和指针；模板、长清单、API 字段表**下沉到 `references/`**，并在正文里写清「什么时候去读哪个文件」。
- 重复要写的确定性逻辑（解析、过滤、格式化）放进 `scripts/`，别在提示词里维护同一份逻辑。
- 删废话比加规则更值——正文加载后在整轮对话里都占上下文。

### 4.5 变量与参数

| 写法 | 含义 |
|---|---|
| `${CLAUDE_SKILL_DIR}` | 本 Skill 自己的目录。脚本、参考资料都用它定位 |
| `${CLAUDE_PROJECT_DIR}` | 当前项目目录 |
| `${CLAUDE_PLUGIN_ROOT}` | 插件根目录（插件提供的 Skill 用） |
| `${CLAUDE_SESSION_ID}` | 当前会话标识，落日志时才用得上 |
| `$ARGUMENTS` | `/名字` 后面跟的全部参数（整串） |
| `$0` / `$1` / `$2` | 第 1 / 2 / 3 个参数——**0 起，不是 1 起** |
| `$ARGUMENTS[0]` | 与 `$0` 等价；`[1]` 与 `$1` 等价 |

**参数占位的坑**（实测：`/名字 A1 A2 A3` 展开成 `$0`=A1、`$1`=A2、`$2`=A3、`$ARGUMENTS`=整串）：

- **`$0` 才是第一个参数。** 很多博客写成 1 起，照抄会取错一位。
- **参数不够时占位符原样保留字面量**：只传一个参数时 `$2` 就留着 `$2`，不会变成空。正文要交代清楚缺参数时怎么办。
- **没人接收的参数不会被吞**：正文里一个占位符都没写时，参数会以一行 `ARGUMENTS: <原文>` 追加在末尾。但别指望模型一定按它做事。
- **引号不会被剥**：`/名字 "hello world" second` 里 `$ARGUMENTS` 拿到的是带引号的原文。
- **字面 `$` 写成 `\$`**：`预算 $1.00` 里的 `$1` 会被当参数替换，写 `\$1.00` 才安全。

### 4.6 动态内容：先跑掉，再给模型

正文里可以内联执行 shell，**输出先替换进去，再发给模型**——省一轮工具调用，也避免模型跑错命令：

```markdown
当前分支是：!`git rev-parse --abbrev-ref HEAD`
```

也支持围栏块：

````markdown
```!
git rev-parse --abbrev-ref HEAD
```
````

要注意：

- **`!` 前面必须有空白或换行**：`` x=!`cmd` `` 不展开，`` x= !`cmd` `` 才展开。这是「写了没生效」最常见的原因。
- **默认走 bash，与平台无关**。Windows 上要跑 PowerShell 才需要写 `shell: powershell`（见 [`../windows.md`](../windows.md)）。
- **不受权限审批管辖**：它是提示词展开的一部分，不是工具调用——Skill 一旦被调用，那段 shell 就会跑，没有二次确认。**别放改状态、碰密钥、联网的命令。**
- 企业策略可以整体关掉（`disableSkillShellExecution`）。被关时那块会替换成 `[shell command execution disabled by policy]`，看到这句话就是被策略拦了。
- 模型看到内联 shell 输出时**可能当成「试图注入指令的文本」而拒绝执行**，这是它在防注入，属正常表现。
- **`@路径` 引文件不要依赖**：2.1.268 上复现不出来（相对路径、绝对路径、行首都试过，都是原样文本）。要引文件就用 `!` 注入（如 `` !`cat 路径` ``），或让模型自己读。

### 4.7 一个完整示例

放进 `~/.claude/skills/weekly-report/SKILL.md` 就能用（目录名和 `name` 一致）：

```markdown
---
name: weekly-report
description: 把一周的 git 提交和改动整理成周报。当用户说「写周报」「本周总结」「这周干了啥」时使用。
allowed-tools: Bash, Read
---

# 周报

按下面的顺序做，不要跳步。

1. 用 git log 拉出本周自己的提交（`--since="7 days ago" --author` 到你自己）。
2. 提交信息看不出做了什么时，去读对应的 diff，**不要凭提交信息猜**。
3. 按「功能 / 修复 / 杂项」三类归纳，每类最多 5 条，一条一句话，写清改了什么、为什么。
4. 不确定该不该写进去的先问用户，不要编。
5. 用中文输出，不要开场白，不要复述本指令。

这份内容会直接贴给主管：**把没做的事写成做了，比漏写严重得多。**
```

需要附带文件时再扩成这样：

```text
weekly-report/
├─ SKILL.md                    # 只放流程和判断规则
├─ references/
│  └─ templates.md             # 周报模板、历史样例、措辞禁区
└─ scripts/
   └─ collect_commits.sh       # 拉提交并按分类打标，输出结构化结果
```

正文里对应写两句：**要套模板时读 `references/templates.md`**；**先跑 `${CLAUDE_SKILL_DIR}/scripts/collect_commits.sh` 拿结构化结果**，再做归类和取舍——脚本负责确定性，Skill 负责判断。

## 5. 怎么调用

| 途径 | 说明 |
|---|---|
| 自动触发 | 你的说法命中 `description` 时，模型自己去读这个 Skill |
| 显式调用 | 敲 `/名字`，后面的参数作为 `$ARGUMENTS` 传进去 |
| Skill 工具 | 模型加载 Skill 正文的实际手段 |

- 插件提供的 Skill 带命名空间，写成 `插件名:skill名`；裸名同时匹配多个时会要求写全名。
- `/skills` 列出当前可用的 Skill（被 `skillOverrides` 关掉的也在里面，带状态标记）；用量和上下文成本在插件管理器的 **Stats** 标签页。
- 刚在磁盘上加了或改了 Skill，用 `/reload-skills` 拾取，不用重启会话；插件来源的改动用 `/reload-plugins`。
- 想让某个 Skill **只能你手动调**，写 `disable-model-invocation: true`；只想让模型调，写 `user-invocable: false`。

## 6. 验证与排查

**先跑这几条：**

```text
/skills                 列出可用 Skill
/context                看上下文总量，判断是不是 Skill 正文吃太多
/plugin validate .claude/skills    校验目录结构
claude --debug          看加载过程日志
```

`/plugin validate` 的输出分 **error**（例：frontmatter 不是键值映射）和 **warning**（例：缺 `description`）。它只校验结构，不执行行为——行为验证用 `claude plugin eval`。

**常见现象对照：**

| 现象 | 先查什么 |
|---|---|
| 列表里根本没有 | 路径是不是 `~/.claude/skills/<名字>/SKILL.md` 或 `<项目>/.claude/skills/<名字>/SKILL.md`；`SKILL.md` 文件名大小写 |
| 列表里的描述变成了正文内容 | frontmatter 没被当成键值对解析，或压根没写 `description`；跑 `/plugin validate` |
| 在列表里但从不自动触发 | `description` 没写清判断依据（4.3 节）；先手动敲 `/名字` 排除描述因素 |
| 敲 `/名字` 提示找不到 | Skill 名用的是**目录名**，不是 frontmatter 的 `name` |
| 改完不生效 | `/reload-skills`（插件改动用 `/reload-plugins`） |
| 显示为锁定、改不了 | frontmatter 写了 `disable-model-invocation`，或 `skillOverrides` 设成了 `off` |
| 我能敲 `/名字`，但模型不会自己用 | 有 `disable-model-invocation: true`，预期行为 |
| 参数取错了一位 | 占位符写成了 `$1` 却以为是第一个——`$0` 才是 |
| 钱数 / shell 变量被吃掉 | 字面 `$` 要写 `\$` |
| 正文里的 `!` 没展开 | `!` 前是不是紧贴别的字符；或那块变成了 `[shell command execution disabled by policy]` |
| `@file` 没带进文件内容 | 2.1.268 上复现不出来，改用 `!` 注入或让模型自己读 |
| 菜单里能看到 `/名字`，但生效的是另一份 | 有同名 Skill（Skill 赢）或另一个位置有同名命令，两者都**静默** |
| 反过来：模型会调，我敲不出来 | `user-invocable: false`，预期行为 |
| 内置 Skill 全不见了 | `disableBundledSkills` 或 `CLAUDE_CODE_DISABLE_BUNDLED_SKILLS` |
| 企业策略锁掉了自定义来源 | `strictPluginOnlyCustomization` 里含 `"skills"` |

**`skillOverrides`** 写在 settings 里，按 Skill 名字生效，用来临时控成本：

| 取值 | 效果 |
|---|---|
| `off` | 对你和模型都隐藏 |
| `name-only` | 只列名字、不带描述（省上下文，也基本不会被自动触发） |
| `user-invocable-only` | 对模型隐藏，只保留你敲 `/名字` |

## 7. 团队规范

- 先找已有 Skill 和仓库工具，再考虑新增实现。
- 一个 Skill 只解决一类清晰问题，不把多个无关工作流揉在一起。
- 不在 Skill 或文档中提交密码、Token、私钥及其他敏感信息。
- 不用扩大权限或跳过审批来掩盖范围不清的问题；先缩小任务和权限。`allowed-tools` 只开真正需要的。
- 需要修改文件时，明确修改范围、不要触碰的内容和验证命令。
- 脚本、模板和资源优先复用，不要在提示词里重复维护同一份逻辑。
- `.agents/` 下的 Agent 可读文件使用英文；面向团队的 `agent/` 文档可以使用中文。

**提交前检查：**

- 是否复用了已有 Skill？
- `description` 能不能让模型正确判断「该用 / 不该用」？
- 是否包含最小权限和敏感信息约束？
- 是否有一个最小可运行的验证方式（`/plugin validate` + 手动 `/名字` 跑一次）？
- 是否只改动了完成任务所需的文件？

## 8. 共享与插件分发

| 方式 | 生效范围 | 适合 | 代价 |
|---|---|---|---|
| `~/.claude/skills/<名字>/` | 只有你，所有项目 | 个人习惯、试验 | 换机器要重来，同事看不见 |
| `<仓库>/.claude/skills/<名字>/` | 跟着仓库走 | 团队共享的项目规则 | 每个仓库各存一份。**本仓库 `.claude/` 在 `.gitignore` 里**，直接放进去不会提交（`commands/` 同理） |
| 插件（marketplace） | 装了插件的所有人 | 跨项目复用、要版本和分发 | 多一层打包和发布流程 |

怎么选只看两件事：**要不要跨项目复用**、**要不要装到同事机器上**。只在自己一个仓库里用，第一种就够。

> 别和本仓库的 `.agents/skills/` 搞混：那是 Codex / Pi 的位置，Claude Code 不认。

插件这条路大致是：

```bash
claude plugin init my-skill             # 起骨架
claude plugin validate .claude/skills   # 校验结构
claude plugin eval                      # 跑行为验证
```

会话里：

```text
/plugin marketplace add <owner/repo 或 URL>
/plugin                 # 插件主菜单：浏览、安装、启用 / 禁用
/plugin help            # 看当前版本有哪些子命令
/reload-plugins         # 装完 / 改完让当前会话生效
```

插件带来的 Skill 在列表和调用里都带命名空间：`插件名:skill名`、`/插件名:skill名`。

**别为了一个任务装插件**：只要项目内的规则，用第 3 节的两个目录就够；确实要跨项目复用、要给同事装、要版本管理时，才值得走插件。

## 一页速查

```text
是什么      包含 SKILL.md 的目录；description 常驻，正文和附带文件按需加载
            正文上限 1 MB，加载后整轮对话都占上下文

位置        ~/.claude/skills/<名字>/SKILL.md          用户级，所有项目
            <项目>/.claude/skills/<名字>/SKILL.md     项目级（项目根 + 当前目录）
            插件 skills/<名字>/SKILL.md               带 插件名: 前缀
            --plugin-dir <路径>                       临时调试插件

写           调用名 = 目录名（frontmatter 的 name 不一致不报错）
            description 写清"做什么 + 什么时候用"，把用户原话写进去
            正文命令式、讲清为什么；细节下沉 references/；确定性逻辑放 scripts/
            allowed-tools 只开需要的；只手动调用写 disable-model-invocation: true

变量        ${CLAUDE_SKILL_DIR}   ${CLAUDE_PROJECT_DIR}   ${CLAUDE_PLUGIN_ROOT}
            ${CLAUDE_SESSION_ID}
            $ARGUMENTS        全部参数（整串）
            $0 $1 $2          第 1 / 2 / 3 个  ← 0 起！不是 1 起
            $ARGUMENTS[0]     等价于 $0
            参数不够时        占位符保留字面量，不变空
            没人接收          末尾自动追加 ARGUMENTS: <原文>
            字面 $            写 \$1.00

动态内容    !`命令`           内联执行，输出先替换再给模型（! 前必须空白或行首）
            ```! 围栏块       同上的块形式
            默认 bash         与平台无关；Windows 要 PowerShell 才写 shell: powershell
            不受审批管辖      被调用就会跑，别放改状态 / 碰密钥的命令
            @路径             2.1.268 复现不出来，别依赖；要引文件用 ! 注入

调用        自动匹配 description  |  /名字 后面跟参数  |  插件用 /插件名:skill名

命令文件    .claude/commands/<名字>.md 是 Skill 的轻量形态
            <项目>/.claude/commands/ > ~/.claude/commands/ > 插件（低的静默忽略）
            commands/sub/t2.md → /sub:t2（子目录 : 拼命名空间）
            同名时 Skill 赢，命令被静默忽略
            升级：commands/review.md → skills/review/SKILL.md，别两边各留一份

排查        /skills                        列表
            /context                       上下文总量
            /plugin validate .claude/skills 结构校验
            /reload-skills                 拾取磁盘改动（插件用 /reload-plugins）
            claude --debug                 加载日志

控成本      skillOverrides: off | name-only | user-invocable-only
            disableBundledSkills 关掉内置 Skill

共享        个人 → ~/.claude/skills/    项目 → <仓库>/.claude/skills/    跨项目 → 插件
            注意：.agents/skills/ 是 Codex / Pi 的位置，Claude Code 不读
```
