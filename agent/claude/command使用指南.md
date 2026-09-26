# Claude Code Command 使用指南

本文面向本仓库团队成员，说明如何在 Claude Code 里写、用、排查**自定义斜杠命令**（`.claude/commands/*.md`）。文中机制基于 Claude Code 2.1.x（Windows + PowerShell 环境），命令和默认行为随版本变化，遇到不一致先敲 `/` 看菜单里到底列出了什么。

前置阅读：[`README.md`](./README.md)（安装、配置、权限模式）；Command 与 Skill 的关系、以及「什么时候该用哪个」见 [`skill使用指南.md`](./skill使用指南.md) 第 2 节。Codex 目前没有对应篇目；Pi 的「提示词模板（`prompts/`）」是同类机制，见 [`../pi/Extension使用指南.md`](../pi/Extension使用指南.md)。

> **关于本文的事实来源**：命令名解析、子目录命名空间、参数替换、同名优先级、`!` 注入语法均在本机 Claude Code **2.1.268** 上用一次性测试项目实测（`claude -p` + 临时 `.claude/commands/` 目录）。其余标注「官方文档口径」的内容没有本地复现，按参考值看待；标注「二进制核对」的是从 `claude.exe` 里读出的字符串，行为见过但没做端到端复现。

---

## 1. Command 是什么

自定义命令就是**一个 Markdown 文件**。文件名决定命令名，正文就是你敲 `/名字` 之后发给模型的那段提示词：

```text
.claude/commands/fix-issue.md   →   /fix-issue
```

它的价值在于**把你会反复敲的一段提示词固化下来**：不用每次重新解释"按我们团队规范做"，不用每次把背景贴一遍，也不用担心这次少说了一句。

### 1.1 先搞清楚它和 Skill 的关系

这是本篇最容易写错、也最值得先说明白的一点：

> **官方已经把自定义命令并入了 Skill 机制（2.1.88 起）。** `.claude/commands/foo.md` 和 `.claude/skills/foo/SKILL.md` 产生的是**同一个 `/foo`**；两者同名时 **Skill 生效，命令被静默忽略**（第 2.3 节，本地实测）。

换句话说，命令不是一条平行技术路线，而是 **Skill 的轻量形态**：一个 md 文件 vs 一个目录，共用同一套 frontmatter、同一套调用路径、同一个"技能"工具。本机二进制里 `disable-model-invocation` 的描述原文就是"模型不能通过 **Skill 工具** 调用它" —— 命令和 Skill 走的是同一条路。

所以别把这篇读成"命令比 Skill 好"，正确的读法是：**先用命令，长到该有流程和附带文件了，再升级成 Skill**（第 7 节给了判据和迁移动作）。

### 1.2 和其他机制怎么分工

沿用 [`Hook使用指南.md`](./Hook使用指南.md) 第 1 节那张表：

| 你想要的效果 | 该用哪个 | 为什么 |
|---|---|---|
| 每次都必须发生，不依赖模型判断 | **Hook**（[`Hook使用指南.md`](./Hook使用指南.md)） | 由 harness 执行，写了就一定跑 |
| 恒真的项目事实和约定 | `CLAUDE.md`（[`README.md`](./README.md) 第 6 节） | 每轮都在上下文里，代价是每轮都付 token |
| 一类任务的流程、检查单、边界 | **Skill**（[`skill使用指南.md`](./skill使用指南.md)） | 平时只占一行描述，需要时才展开 |
| **一句话的固定提示词、要带参数** | **自定义命令（本文）** | 点一下就展开，不用每次重打 |
| 要接入新工具或数据源 | MCP server（[`mcp使用指南.md`](./mcp使用指南.md)） | 提供的是工具本身 |

一句话版：**必须每次都发生的放 Hook，恒真的放 CLAUDE.md，按需的流程放 Skill，一句话的提示词放命令。**

> **判断标准就看一条**：这个东西展开之后是"一段提示词"，还是"一套流程 + 检查单 + 附带文件"。前者用命令，后者用 Skill。

常见用途：

- `/commit-msg`：按团队的提交信息格式生成 commit message；
- `/fix-issue 1234`：拉 issue、定位、改、跑测试，一条龙；
- `/review-mine`：review 自己当前分支的改动，按团队口径给意见；
- `/standup`：把昨天到今天的提交整理成站会发言。

---

## 2. 放哪、叫什么名字

### 2.1 三个位置

| 位置 | 作用范围 | 能提交给同事吗 |
|---|---|---|
| `~/.claude/commands/<名字>.md` | 你所有项目 | 否，只在本机 |
| `<项目>/.claude/commands/<名字>.md` | 单个项目 | **能**，跟着仓库走 |
| 插件里的 `commands/<名字>.md` | 插件启用期间 | 是，跟插件一起分发 |
| `.claude/skills/<名字>/SKILL.md` | 同上（**等价位置**，见 1.1） | 同上 |

**优先级**：项目 > 个人 > 插件（官方文档口径，未逐条复现）。高优先级同名时低优先级那份**被静默忽略**，没有任何告警 —— 这点和 Skill 一样，只能靠纪律。

### 2.2 名字怎么来的

**名字来自文件路径，不是 frontmatter 里写的 `name`。**

| 文件 | 命令名 |
|---|---|
| `.claude/commands/fix-issue.md` | `/fix-issue` |
| `.claude/commands/sub/t2.md` | `/sub:t2` |
| `.claude/commands/a/SKILL.md` | `/a` |

子目录用 **`:`** 拼成命名空间 —— 这条**本地实测过**（`.claude/commands/sub/t2.md` 在会话里确实以 `sub:t2` 出现）。值得提醒的是：**官方文档页上写的还是旧行为**（"子目录只影响命令描述，不影响命令名"），和实际实现对不上。以 `/` 菜单里显示的为准。

命名空间是好东西：按模块分组（`/db:migrate`、`/fe:component`），同名冲突也就自然避开了。

> frontmatter 里的 `name:` 对命令名没有影响。写了不一致**不会有任何报错**，只会在你按 `name` 找不到命令时浪费你十分钟。

### 2.3 同名会怎样

- **命令 vs 命令**（不同位置）：按 2.1 的优先级取一个，另一个静默失效。
- **命令 vs Skill**：**Skill 赢**。本地实测：同时放 `.claude/commands/dup.md` 和 `.claude/skills/dup/SKILL.md`，`/dup` 用的是 Skill 那份描述，命令那份**没有任何提示就没了**。

> ⚠️ 两种"静默失效"合起来是同一个坑：**菜单里能看到 `/dup`，不等于你写的那份在生效。** 改完命令发现"怎么没变化"，第一件事是查有没有同名 Skill。

### 2.4 本仓库的 `.claude/` 在忽略名单里

和 [`skill使用指南.md`](./skill使用指南.md) 第 8 节、[`Hook使用指南.md`](./Hook使用指南.md) 第 2 节是同一个坑：

```text
.gitignore:
.claude/
.agent/
ai_tmp/
.codegraph/
.agents/
```

往 `.claude/commands/` 里放命令**能跑，但不会提交**，同事拉不到。要共享得先决定怎么处理这条忽略规则（第 8 节）。

---

## 3. 最小示例

### 3.1 写一个不接参数的

建 `~/.claude/commands/standup.md`：

```markdown
---
description: 把最近的提交整理成站会发言
---

按下面的顺序做，不要跳步。

1. 用 `git log --since="yesterday" --author=<我自己>` 拉出最近的提交。
2. 提交信息看不出做了什么时，去读对应 diff，**不要凭提交信息猜**。
3. 按「在做 / 做完 / 卡住」三块输出，每块最多 3 条，一条一句话。
4. 不确定的不要编，直接说"不确定"。
5. 用中文输出，不要开场白。
```

### 3.2 写一个接参数的

建 `<项目>/.claude/commands/fix-issue.md`：

```markdown
---
description: 按团队规范修一个 GitHub issue
argument-hint: [issue-number]
allowed-tools: Read, Edit, Bash(gh issue view:*), Bash(gh issue comment:*), Bash(mvn test:*)
---

修 issue #$0。

1. 先 `gh issue view $0` 读清楚现象和复现步骤。
2. 定位到代码后，**先说清你打算改哪几个文件、为什么**，等我确认再动手。
3. 改完跑 `mvn test -Dtest=<相关测试类>`，把结果贴出来。
4. 不要顺手重构不相关的地方。
```

`argument-hint` 只影响你敲 `/` 时菜单里显示的占位提示（二进制核对：原文是 "Placeholder text shown after the slash command name"），不影响参数**怎么传**。

### 3.3 调用和验证

```text
/fix-issue 1234
```

敲 `/` 会弹出可搜索的菜单，输入几个字过滤；带 `description` 的会显示描述。

**改完不生效怎么办**：

```text
/reload-skills
```

它的定义原文是 "Pick up skills added or changed on disk during this session"（二进制核对）—— 因为命令和 Skill 共用加载器，用它来拾取命令改动是合理推断，但**这条我没做端到端复现**（非交互模式没法在一个会话里改文件再看效果）。**不确定就重启会话，一定生效。**

> **Windows 上的一个坑（本地实测）**：在 **Git Bash** 里用 `claude -p "/fix-issue 1234"`，`/fix-issue` 会被 MSYS 的路径转换当成 Unix 路径吃掉，命令根本不触发。加 `MSYS_NO_PATHCONV=1` 前缀即可。PowerShell 里没有这个问题。

---

## 4. frontmatter 字段

字段 schema 和 Skill 是**同一套**，完整清单见 [`skill使用指南.md`](./skill使用指南.md) 第 4.1 节。对命令来说，常改的是这几个：

| 字段 | 作用 | 来源 |
|---|---|---|
| `description` | 菜单和 `/help` 里显示的一行说明 | 官方口径；菜单实测显示无误 |
| `argument-hint` | 命令名后面显示的占位提示，如 `[issue-number]` | 二进制核对 |
| `allowed-tools` | 本命令生效期间给模型开的工具。**支持逗号分隔字符串或 YAML 列表**；`Bash(git status:*)` 这种细粒度写法见 [`README.md`](./README.md) 第 9 节 | 二进制核对 |
| `model` | 用哪个模型跑这条命令 | Skill 篇 |
| `disable-model-invocation` | 设 `true`：**模型不能自己调**，只有你能敲 `/名字` | 二进制核对 |
| `user-invocable` | 设 `false`：**对你隐藏 `/命令`**，只有模型能调 | 二进制核对 |
| `shell` | 正文里 `!` 命令块用哪个 shell：`bash` / `powershell`，**默认 bash 且与平台无关** | 二进制核对 |
| `context` / `agent` | 让命令在独立子 agent 里跑（`context: fork`），见第 7 节 | Skill 篇 |
| `hooks` | 本命令生效期间注册 hook | Skill 篇 |

后两个字段的完整语义在 Skill 篇里，这里不重复。

**`disable-model-invocation: true` 值得单独说。** 命令**默认是模型也能调的** —— 也就是说，模型判断你这句话适合用某条命令时，会自己去调它。对大多数场景这是好事，但对"会发版 / 会写远端 / 会发消息"这类命令不是：你不想它自作主张。这类命令请显式写 `disable-model-invocation: true`，把它锁成"只能人手敲"。

> **有一条不要用**：`arguments`。二进制里它的描述原文是 "@internal — typed variant of argument-hint; argument-hint is the documented form"，即**内部字段，文档形式是 `argument-hint`**。网上有些教程拿它声明具名参数（第 5.3 节），能不能用得看版本，**别写进团队共享的命令里**。

---

## 5. 参数

### 5.1 全部参数：`$ARGUMENTS`

`/fix-issue 1234` → 正文里的 `$ARGUMENTS` 展开成 `1234`。多词时是整串：`/arg2 A1 A2 A3` → `A1 A2 A3`（本地实测）。

### 5.2 单个参数：`$0`、`$1`、`$ARGUMENTS[n]`

**⚠️ 索引是 0 起的**，`$0` 才是第一个参数。这是本篇最值得记住的一条，因为很多博客写的是 1 起。

本地实测（`/arg2 A1 A2 A3`）：

| 正文里写 | 展开成 |
|---|---|
| `$0` | `A1` |
| `$1` | `A2` |
| `$2` | `A3` |
| `$3`（没有第 4 个参数） | **原样保留 `$3`** |
| `$ARGUMENTS[0]` / `[1]` / `[2]` | `A1` / `A2` / `A3` |
| `$ARGUMENTS` | `A1 A2 A3` |

重点看第 4 行：**索引占位符在没有对应参数时保持字面量，不会展开成空**。所以命令正文里写 `$1` 而用户只传了一个参数时，模型看到的是字面的 `$1` —— 这比"静默变成空字符串"安全（至少你看得出没传），但也意味着**命令要自己交代清楚缺参数时怎么办**。

> 引用多词参数要加引号，`/arg "hello world" second` —— 这层是 shell 风格的分词（官方文档口径）。**实测时 `$ARGUMENTS` 会把引号原样带进去**（`"hello world" second`），别指望它帮你剥引号。

### 5.3 具名参数（慎用）

官方文档描述了用 frontmatter 声明具名参数：

```yaml
---
arguments: [issue, branch]
---
修 $issue，目标分支 $branch。
```

规则（官方文档口径，**未本地复现**）：具名按出现位置顺序对应；具名占位符没有对应参数时展开成**空字符串**（注意和第 5.2 节索引占位符的行为**相反**）。但如第 4 节所说，二进制把这个字段标成 `@internal`，**不建议写进团队共享的命令**——用 `$0` / `$1` 更稳。

### 5.4 参数没人接的情况

如果你传了参数，但正文里**没有任何占位符接收它**，Claude Code 会在正文末尾追加一行：

```text
ARGUMENTS: <你输入的原文>
```

本地实测（一条不含任何占位符的命令，`/nop hello world` → 追加 `ARGUMENTS: hello world`）。所以参数不会"被吞掉"，但**也别指望模型一定按那行做事** —— 该写的占位符还是老实写。

### 5.5 字面量 `$`

正文里要写钱数、要写 shell 变量这类字面 `$`，得转义成 `\$`，否则 `$1.00` 里的 `$1` 会被当参数替换掉：

```markdown
预算 \$1.00 以内          # ✓ 不会把 $1 当参数
预算 $1.00               # ✗ $1 会被替换成第二个参数
```

二进制里确实有 `\$` 的转义处理路径（`(?<!\\)\$`）。**但这条我没做出干净的行为复现**：`claude -p` 的返回值要经过模型回显，而模型看到 `$1` 会"自作主张"按它以为的语义改写（实测中它反复把 0 起的 `$1` 当成第一个参数输出），把观测污染了。所以按"建议一律写 `\$`"来办，成本为零。

---

## 6. 动态内容：把命令变成"活的"

### 6.1 注入 shell 命令的输出

正文里可以内联执行命令，**输出会先替换进去，再发给模型**：

```markdown
当前分支是：!`git rev-parse --abbrev-ref HEAD`
```

支持两种写法（二进制核对，两个正则都读到了）：

| 写法 | 形式 |
|---|---|
| 内联 | `` !`命令` `` |
| 代码块 | ```` ```! ```` 开头的围栏块 |

上面那句在模型眼里就是"当前分支是：`main`" —— 它不需要自己去跑 `git`，省一轮工具调用，也避免了它跑错命令。

几条要注意的：

- **默认走 bash（Windows 上是 Git Bash），且不随平台变**。二进制里 `shell` 字段的描述原文是 "Shell for `!`-command blocks: `bash` or `powershell`. Defaults to bash regardless of platform"。要在里面写 PowerShell，得显式加 `shell: powershell`，否则你会看到两种语法打架。
- **这一步不受 Claude Code 权限审批管辖**：它是命令展开的一部分，不是一次工具调用。所以**别在命令正文里塞会改状态、会联网、会碰密钥的命令** —— 这条命令一旦被调用，它就会跑，没有任何二次确认。
- **企业策略可以关掉它**：二进制里有 `disableSkillShellExecution` 策略开关，被关掉时替换成的原文是 `[shell command execution disabled by policy]`。如果你的命令里 `!` 那块变成了这句话，就是被策略拦了，不是命令写错了。
- 本地实测踩到过：模型看到内联的 shell 命令时，**可能把它当成"试图注入指令的文本"而拒绝执行**并跟你解释一遍。这是它在防注入，属正常表现。真需要让它跑，把 `!` 注入的结果用在它必须依赖的地方（而不是让它"照抄一行"）会更自然。

### 6.2 引入文件内容：`@路径`

```markdown
按 @docs/api-convention.md 里的规范写。
```

本地实测：`@note.txt` 会把该文件内容带进命令上下文，模型确实读到了里面的标记字符串。

和 Skill 一样，**路径最好相对命令文件自己写**；依赖当前工作目录的写法在换目录后就不灵了。

### 6.3 会话 ID

`${CLAUDE_SESSION_ID}` 展开成当前会话标识（Skill 篇第 4.5 节的变量表同源）。日常写命令基本用不到，做日志落盘时才有点用。

---

## 7. 该写成 Command、Skill 还是 Subagent

### 7.1 判据

| 你手里的东西 | 写成 |
|---|---|
| 一段提示词 + 一两个参数，一个文件装得下 | **Command** |
| 有流程、有检查单、要带模板/脚本/参考资料 | **Skill**（`skills/<名字>/SKILL.md` + `references/` + `scripts/`） |
| 要把一堆调查结论**汇总完再回主对话**，不想让它挤占上下文 | **Skill + `context: fork`**，或直接 Subagent |
| 团队好几条命令共享同一批约定 | 先抽成 **Skill**，命令只留一层薄壳 |

最后一行是实际会遇到的：几条命令开始互相抄同一段"团队规范"，就该把那一段抽成 Skill，命令里留个指针。

### 7.2 升级成 Skill 的动作

```text
.claude/commands/review.md   →   .claude/skills/review/SKILL.md
```

目录名成为命令名，文件内容搬过去即可 —— 同一套 frontmatter，同一套参数语法。**别两边各留一份**：两份说明一定会漂移，而且如第 2.3 节所述，同名时命令那份会被静默忽略，你以为改了其实没生效。

### 7.3 什么时候用 `context: fork`

正文会长、探索过程会塞满上下文的命令（比如"翻遍仓库找所有调用点"），在 frontmatter 里加：

```yaml
---
context: fork
agent: Explore
---
```

这样它在独立子 agent 里跑，主对话只拿到结论。代价是你看不到中间过程，命令的正文要**明确写清要返回什么**，否则回来的结论会很空。

---

## 8. 团队共享与安全

### 8.1 怎么让同事用上

先说结论：**本仓库现在共享不出去**。`.claude/` 在 `.gitignore` 里（2.4 节），放进去的命令不会提交。三条路：

| 方式 | 生效范围 | 代价 |
|---|---|---|
| 改 `.gitignore`，提交 `.claude/commands/` | 跟着仓库走 | 要确认团队接受把 agent 配置提交进仓库 |
| 每人放 `~/.claude/commands/` | 只有自己 | 换机器要重来，同事看不见 |
| 打包成插件分发 | 装了插件的所有人 | 多一层打包发布流程 |

命令本身很轻，**建议先走第一条**：把 `.claude/commands/` 从忽略名单里放出来，提交几条团队共用的。如果不愿意动忽略规则，那就得接受"命令是个人资产"。

### 8.2 安全

- **命令正文里的 `!` 是直接执行的，没有审批**（6.1 节）。共享命令要评审 —— 同事拉下仓库、敲一个 `/命令`，里面那段 shell 就跑起来了，和 `.mcp.json` 是同一类"别人替你决定执行什么"的东西。
- **`allowed-tools` 只开需要的**。别图省事写 `allowed-tools: Bash`（等于放行所有命令），要按 `Bash(mvn test:*)` 这种细粒度写。
- **会改外部状态 / 会花钱 / 会发消息的命令，一律加 `disable-model-invocation: true`**，至少保证它是你主动敲的。
- **密钥不进命令正文**。命令文件是要提交、要共享的，Token / 私钥 / 内部地址都不该出现。

---

## 9. 排查

**先做这三件事：**

```text
/            敲斜杠，看菜单里到底有没有这条命令、名字长什么样
/reload-skills   拾取磁盘上新增/修改的命令（不确定就重启会话）
/hooks       确认不是被 hook 拦了
```

**常见现象对照：**

| 现象 | 先查什么 |
|---|---|
| 菜单里根本没有 | 路径对不对（`.claude/commands/<名字>.md`，或 `~/.claude/commands/`）；**本仓库 `.claude/` 被 gitignore 不影响加载**，但如果文件其实没建在这台机器上就是没有 |
| 敲 `/名字` 提示找不到 | 命令名是**文件路径**派生的：子目录要写成 `/sub:名字`，不是 `/名字`（2.2 节） |
| 菜单里能看到，但生效的是另一份 | 有同名 Skill（Skill 赢），或另一个位置有同名命令（优先级高的赢），两者都是**静默**的（2.3 节） |
| 改了命令没反应 | `/reload-skills`；还不行重启会话。改了 frontmatter 里的 `name:` 是没用的，改文件名才是改命令名 |
| 参数没传进去 | 占位符是不是写成了 `$1` 却只传了一个参数（`$1` 是**第二个**，第一个是 `$0`）；或者压根没写占位符（那样只会在末尾追加 `ARGUMENTS: ...`，5.4 节） |
| 钱数 / shell 变量被吃掉 | 字面 `$` 要写 `\$`（5.5 节） |
| `!` 那块没执行 | 正文里是不是写成了 `[shell command execution disabled by policy]` —— 那是被策略开关关了（6.1 节），不是命令写错 |
| `!` 里的 PowerShell 语法报错 | `shell` 字段默认是 `bash`，要用 PowerShell 得显式写 `shell: powershell`（6.1 节） |
| 模型拒绝执行命令里的 `!` 那行 | 它在防注入，属正常；把注入结果用在它真正需要的地方，别让它"照抄一行"（6.1 节） |
| 在 Git Bash 里 `claude -p "/命令 参数"` 没反应 | MSYS 把 `/命令` 当路径转了，加 `MSYS_NO_PATHCONV=1`（3.3 节） |

---

## 一页速查

```text
是什么     一个 Markdown 文件 = 一条斜杠命令；文件名即命令名
            官方已并入 Skill 机制：.claude/commands/x.md 与 .claude/skills/x/SKILL.md
            产生同一个 /x，同名时 Skill 赢（命令静默失效）

位置        ~/.claude/commands/<名字>.md          你所有项目，不可提交
            <项目>/.claude/commands/<名字>.md     项目级，可提交
            插件 commands/<名字>.md               跟插件走
            优先级 项目 > 个人 > 插件（官方口径）

命名        .claude/commands/fix-issue.md   →  /fix-issue
            .claude/commands/sub/t2.md      →  /sub:t2     （子目录 : 拼命名空间，实测）
            名字来自路径，不是 frontmatter 的 name（写了不一致不报错）

参数        $ARGUMENTS        全部参数（整串）
            $0 $1 $2          第 1 / 2 / 3 个  ← 0 起！不是 1 起
            $ARGUMENTS[0]     同上，写法等价
            参数不够时        索引占位符原样保留字面量，不展开成空
            没人接收          正文末尾自动追加 ARGUMENTS: <原文>
            字面 $            写 \$1.00（$1 会被替换）

动态内容    !`命令`           内联执行，输出先替换再给模型（默认 bash，与平台无关）
            ```! 围栏块       同上的块形式
            shell: powershell  要让 ! 走 PowerShell 必须显式写
            ! 不受审批管辖     命令被调就会跑，别放改状态/碰密钥的命令
            @路径             引入文件内容
            策略关了会变成 [shell command execution disabled by policy]

frontmatter description / argument-hint / allowed-tools / model
            常用  disable-model-invocation: true  锁成只能人手敲（默认模型也能调）
                  user-invocable: false          对你隐藏，只给模型用
            ⚠️ arguments 是 @internal，别用

升级成 Skill  commands/review.md → skills/review/SKILL.md
            正文会长 / 要带模板脚本 / 要 fork 出去 就该升级
            同名优先 Skill ⇒ 别两边各留一份

共享        ⚠️ 本仓库 .claude/ 在 .gitignore 里，写了不会提交（同 Skill / Hook 篇）
            命令正文里的 ! 是直接执行的，共享命令要评审

排查        /            菜单里有没有、叫什么
            /reload-skills  拾取磁盘改动；不确定就重启
            名字对不上 → 子目录要写 /sub:名字；生效的是别人 → 查同名 Skill
```
