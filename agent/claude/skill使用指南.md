# Claude Code Skill 使用指南

本文面向本仓库团队成员，说明如何在 Claude Code 里选择、使用和维护 Skill。文中机制基于 Claude Code 2.1.x（Windows + PowerShell 环境），命令和默认行为随版本变化，遇到不一致先看 `claude --help` 和 `/help`。

前置阅读：[`README.md`](./README.md)（安装、配置、权限模式）；Skill 在 Claude Code 整体能力里的位置见该文第 10 节。需要独立上下文预算或并行的活用子代理，见 [`subagent使用指南.md`](./subagent使用指南.md)。Codex / Pi 的对应文档见 [`../codex/skill使用指导.md`](../codex/skill使用指导.md)、[`../pi/skill使用指导.md`](../pi/skill使用指导.md)。

---

## 1. Skill 是什么

Skill 是一组可复用的 Agent 工作规则，通常用于固定某类任务的触发条件、执行顺序、安全边界和验证方式。一个 Skill 就是一个包含 `SKILL.md` 的目录，可以顺带打包脚本、参考资料和模板：

```text
pdf-tools/
├─ SKILL.md          # 必需：frontmatter + 正文指令
├─ scripts/          # 可选：可执行脚本，重复要写的确定性逻辑放这里
├─ references/       # 可选：按需读取的参考资料
└─ assets/           # 可选：输出要用的模板、图标、字体
```

关键点是**三层按需加载**：

| 层级 | 什么时候进上下文 | 成本 |
|---|---|---|
| `name` + `description` | 一直在，每一轮都可见 | 一行描述，基本可以忽略 |
| `SKILL.md` 正文 | 命中触发条件、模型决定加载时 | 正文多长就多少 token |
| `references/`、`scripts/` 等 | 模型真正去读的时候 | 不读就不花钱 |

所以装一堆 Skill 但不触发，和不装它们差不多。两条实测细节：

- `SKILL.md` 正文**上限 1 MB**，超过直接报错不加载（正常写法远到不了这个量级）。
- 正文一旦加载，**本轮对话里会一直占着上下文**；只有 `references/` 里的文件是随读随丢。这也是 4.4 节反复强调"正文写短"的原因。

Skill 本身不提供新工具，它提供的是"这类活该怎么干"的流程知识。

---

## 2. 什么时候用 Skill

Claude Code 里能"固化规则"的地方不止一处，先分清再动手：

| 你想要的效果 | 该用哪个 | 为什么 |
|---|---|---|
| 每次都必须发生，不依赖模型判断 | **Hook**（`settings.json` 的 `hooks`） | 由 harness 执行，不经过模型，写了就一定跑 |
| 恒真的项目事实和约定 | **CLAUDE.md** | 每轮都在上下文里，代价是每轮都付 token |
| 成体系、要分文件或按路径生效的约定 | **Rule**（[`rule使用指南.md`](./rule使用指南.md)） | 自动加载，可用 `paths` 只在相关文件上生效 |
| 一类任务的流程、检查单、边界 | **Skill** | 平时只占一行描述，需要时才展开 |
| 需要独立上下文预算 / 要并行 | **子 agent（Subagent）**（见 [`subagent使用指南.md`](./subagent使用指南.md)） | 单独的上下文窗口，不挤占主对话 |
| 要接入新工具或数据源 | **MCP server**（见 [`mcp使用指南.md`](./mcp使用指南.md)） | 提供的是工具本身，Skill 只能教模型怎么用 |
| 一句话的固定提示词 | **自定义命令**（`.claude/commands/*.md`，见 [`command使用指南.md`](./command使用指南.md)） | 纯参数化展开，不带流程 |

一句话版：**必须每次都发生的放 Hook，恒真的放 CLAUDE.md，按需的流程放 Skill。**

Skill 适合重复出现、需要稳定流程的工作（怎么补单测、怎么发版、怎么审这类 PR）；一次性的小任务不值得为了"走流程"额外加步骤。

> 本仓库另有一套 `.agents/skills/`（`agent-perf`、`github-cli`、`kit-manage`），那是 Codex 和 Pi 的加载位置，**Claude Code 不读**。要在 Claude Code 里用，得放到第 3 节列的位置。

---

## 3. 从哪些位置加载

| 位置 | 作用范围 | 说明 |
|---|---|---|
| `~/.claude/skills/<名字>/SKILL.md` | 用户级 | 对所有项目生效 |
| `<项目>/.claude/skills/<名字>/SKILL.md` | 项目级 | **项目根和当前工作目录**都会扫一遍 |
| 已启用插件里的 `skills/<名字>/SKILL.md` | 插件 | 随插件启用/禁用，名字带 `插件名:` 前缀 |
| `--plugin-dir <路径>` | 单次会话 | 本地调试插件用，不写进配置 |

几条容易踩的：

- **目录名就是 Skill 名**，也是 `/命令` 名。frontmatter 里的 `name` 一致性靠自觉，写错不报错（见 4.2）。
- 用户级路径跟着 `CLAUDE_CONFIG_DIR` 走（配了这个变量，`~/.claude/` 整个搬走）。
- 同一条路径既被用户级加载又被插件提供时，**插件的会被跳过**，启动日志里是 `Skipping plugin skill '<名字>' — <路径> is a user-level skill already surfaced by the skills directory loader`。
- **同名不会后者覆盖前者**：几个不同来源的 Skill 可以同名并存，列表按名字成组，`skillOverrides`（第 6 节）也是按名字整组生效。
- 企业策略里如果 `strictPluginOnlyCustomization` 带了 `"skills"`，上面这些用户级/项目级目录会被整体锁掉，只剩插件和策略来源。

---

## 4. 自己写一个 Skill

### 4.1 目录与 frontmatter

`SKILL.md` 以 YAML frontmatter 开头，正文直接写指令。可用的字段如下——真正影响能否被触发的是 `description`，其余都能省：

| 字段 | 作用 |
|---|---|
| `name` | Skill 名，也是 `/命令` 名，通常与目录同名 |
| `description` | 一行摘要，**决定模型什么时候调用它**；列表和 Skill 工具里显示的就是它 |
| `when_to_use` | 补充"什么场景该用它"，会并进 Skill 工具的描述 |
| `model` | 用哪个模型跑：`haiku` / `sonnet` / `opus` / `fable` 或完整 ID，`inherit` 表示跟随当前对话 |
| `allowed-tools` | 本文件生效期间给模型开的工具，逗号分隔字符串或 YAML 列表 |
| `disallowed-tools` | 本文件生效期间从模型收回的工具；你发下一条消息时失效 |
| `argument-hint` | `/命令` 后面显示的占位提示，如 `<file>` |
| `disable-model-invocation` | 设为 `true`：模型不能自动调用，只有你能敲 `/名字` |
| `user-invocable` | 设为 `false`：对你隐藏 `/命令`，只有模型能调 |
| `effort` | 思考强度：`low` / `medium` / `high` / `max` 或整数 |
| `context` | `inline`（默认，展开在当前对话）或 `fork`（另起一个子 agent） |
| `agent` | `context: fork` 时用哪种子 agent |
| `background` | 只在 `context: fork` 下有效：fork 出来的子 agent 后台跑、完了用任务通知回报；设 `false` 则当前轮等结果 |
| `globs` | Glob 列表：只有模型碰到匹配的文件时，这个 Skill 才会加载 |
| `hooks` | 本 Skill 生效期间注册的 hook，写法同 `settings.json` 的 `hooks` |
| `shell` | 正文里 `!` 命令块用哪个 shell（`bash` / `powershell`），默认 bash |
| `metadata` | 给作者自己用的自由键值，加载后原样保留 |

### 4.2 硬规则

- **寻址用的是目录名，不是 frontmatter 里的 `name`**。名称不能含括号、逗号和控制字符，不能有前后空格，不能以 `/` 开头，不能用 `*` 通配。
- **目录名和 `name` 不一致时没有任何告警**（实测：目录 `foo/` 里写 `name: bar`，`/plugin validate` 通过，列表里显示的是 `foo`）。校验帮不了你，这只能靠纪律——两处保持一致。
- **frontmatter 必须是键值映射**。写成数组时 `/plugin validate` 会报 `Frontmatter must be a YAML mapping (key: value pairs), got an array`——**但 Skill 依然会被加载**，只是 frontmatter 全丢、描述退化成正文里的文本。
- **缺 `description` 只是 warning**（`No description in frontmatter…`），Skill 照样出现在列表里，描述同样退化成正文内容。触发基本靠运气，等于没写。

> 三条合起来是同一个坑：**"列表里能看到"不等于"写对了"**。验收看 `/plugin validate` 有没有报错，而不是看 Skill 有没有出现。
>
> 引用附带文件时用**相对 Skill 目录的路径**，别依赖当前工作目录（见 4.5 的 `${CLAUDE_SKILL_DIR}`）。

### 4.3 `description` 怎么写才触发得上

`description` 是唯一的自动路由依据，必须同时说清**做什么**和**什么时候用**。

模型在这件事上偏保守——该用的时候经常想不起来用（官方 skill-creator 管这叫 undertrigger）。所以宁可写得主动一点：把同事可能说的**原话**写进描述里，比抽象概括有效得多。

```yaml
# 不行：给不出任何路由信息
description: Helps with documents.

# 可以
description: 把 Markdown 转成公司模板的 Word 文档。当用户提到「导出 Word」「转成 docx」
  「要一份能发出去的正式文档」时使用。
```

### 4.4 正文怎么写

- 用**命令式**写（"先读 X，再改 Y"），并且**解释为什么**。堆 `ALWAYS` / `MUST` 的硬规则，效果通常不如把道理说清楚。
- `SKILL.md` 只放流程、判断规则和指针；模板、长清单、API 字段表这些**下沉到 `references/`**，并在正文里写清"什么时候去读哪个文件"。
- 重复要写的确定性逻辑（解析、过滤、格式化）放进 `scripts/`，别在提示词里维护同一份逻辑。
- 删废话比加规则更值——正文加载后在整轮对话里都占着上下文（第 1 节）。

### 4.5 变量与参数

| 写法 | 含义 |
|---|---|
| `${CLAUDE_SKILL_DIR}` | 本 Skill 自己的目录。脚本、参考资料都用它定位 |
| `${CLAUDE_PROJECT_DIR}` | 当前项目目录 |
| `${CLAUDE_PLUGIN_ROOT}` | 插件根目录（插件提供的 Skill 用） |
| `$ARGUMENTS` | `/名字` 后面跟的全部参数 |
| `$ARGUMENTS[0]`、`$1` | 第 1 个参数；`$2` 是第 2 个，以此类推 |

### 4.6 一个完整示例

放进 `~/.claude/skills/weekly-report/SKILL.md` 就能用（注意目录名和 `name` 一致）：

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

要长到需要附带文件时，再扩成这个样子：

```text
weekly-report/
├─ SKILL.md                    # 只放流程和判断规则
├─ references/
│  └─ templates.md             # 周报模板、历史样例、措辞禁区
└─ scripts/
   └─ collect_commits.ps1      # 拉提交并按分类打标，输出结构化结果
```

正文里对应写两句：**要套模板时读 `references/templates.md`**；**先跑 `${CLAUDE_SKILL_DIR}/scripts/collect_commits.ps1` 拿结构化结果**，再在正文里做归类和取舍——脚本负责确定性，Skill 负责判断。

---

## 5. 怎么调用

三种途径，不用刻意选：

| 途径 | 说明 |
|---|---|
| 自动触发 | 你的说法命中 `description` 时，模型自己去读这个 Skill |
| 显式调用 | 敲 `/名字`，后面跟的参数会作为 `$ARGUMENTS` 传进去 |
| Skill 工具 | 模型加载 Skill 正文的实际手段，自动触发和插件里包装的都是它 |

- 插件提供的 Skill 带命名空间，写成 `插件名:skill名`；裸名同时匹配到多个时会要求写全名。
- `/skills` 列出当前可用的 Skill（被 `skillOverrides` 关掉的也在里面，带状态标记）；每个 Skill 的用量和上下文成本在插件管理器的 **Stats** 标签页。
- 刚在磁盘上加了或改了 Skill，用 `/reload-skills` 拾取，不用重启会话；插件来源的改动用 `/reload-plugins`。
- 想让某个 Skill **只能你手动调**，在 frontmatter 里写 `disable-model-invocation: true`；反过来只想让模型调，写 `user-invocable: false`。

---

## 6. 验证与排查

**先跑这几条：**

```text
/skills                 列出可用 Skill（用量 / 上下文成本在插件管理器 Stats 标签页）
/context                看当前上下文总量，判断是不是 Skill 正文吃太多
/plugin validate .claude/skills    校验目录里的 Skill 结构（也接受 .claude、插件目录、manifest 文件）
claude --debug          看加载过程的日志
```

`/plugin validate` 的输出分两级：**error**（例：frontmatter 不是键值映射）和 **warning**（例：缺 `description`）。它只校验结构，不执行行为——行为验证用 `claude plugin eval`。

Skill 的用量和上下文成本报告原来是 `/skill-doctor`，现在并进了插件管理器的 **Stats** 标签页。

**常见现象对照：**

| 现象 | 先查什么 |
|---|---|
| 列表里根本没有 | 路径是不是 `~/.claude/skills/<名字>/SKILL.md` 或 `<项目>/.claude/skills/<名字>/SKILL.md`；`SKILL.md` 文件名大小写、是否被写成了别的名字 |
| 列表里的描述变成了正文内容 | frontmatter 没被当成键值对解析，或压根没写 `description`。跑 `/plugin validate` 看是 error 还是 warning（4.2 节） |
| 在列表里但从不自动触发 | `description` 没写清判断依据（4.3 节）；先手动敲 `/名字` 排除掉描述的因素 |
| 敲 `/名字` 提示找不到 | Skill 名用的是**目录名**，不是 frontmatter 里的 `name` |
| 改完不生效 | `/reload-skills`（插件改动用 `/reload-plugins`） |
| 在 `/skills` 里显示为锁定、改不了 | 多半是 frontmatter 写了 `disable-model-invocation`，或 settings 里 `skillOverrides` 把它设成了 `off` |
| 我能敲 `/名字`，但模型不会自己用 | 有 `disable-model-invocation: true`，这是预期行为 |
| 反过来：模型会调，我敲不出来 | `user-invocable: false`，同样是预期行为 |
| 内置 Skill 全不见了 | `disableBundledSkills` 设置项或 `CLAUDE_CODE_DISABLE_BUNDLED_SKILLS` 环境变量 |
| 企业策略锁掉了自定义来源 | `strictPluginOnlyCustomization` 里含 `"skills"` |

**`skillOverrides`** 写在 settings 里，按 Skill 名字生效，用来临时控成本：

| 取值 | 效果 |
|---|---|
| `off` | 对你和模型都隐藏 |
| `name-only` | 只列名字、不带描述（省上下文，但也基本不会被自动触发） |
| `user-invocable-only` | 对模型隐藏，只保留你敲 `/名字` 这条路 |

---

## 7. 团队规范

- 先找已有 Skill 和仓库工具，再考虑新增实现。
- 一个 Skill 只解决一类清晰问题，不把多个无关工作流揉在一起。
- 不在 Skill 或文档中提交密码、Token、私钥及其他敏感信息。
- 不用扩大权限或跳过审批来掩盖范围不清的问题；先缩小任务和权限。给 Skill 写 `allowed-tools` 时，只开它真正需要的。
- 需要修改文件时，明确修改范围、不要触碰的内容和验证命令。
- Skill 里的脚本、模板和资源优先复用，不要在提示词里重复维护同一份逻辑。
- `.agents/` 下的 Agent 可读文件使用英文；面向团队的 `agent/` 文档可以使用中文。

推荐的任务提示词：

```text
/weekly-report 只处理我自己的提交。先说明归类和取舍依据，再输出；不要把 review 别人代码的时间算成产出。
```

**提交前检查：**

- 是否复用了已有 Skill？
- `description` 能不能让模型正确判断"该用 / 不该用"？
- 是否包含最小权限和敏感信息约束？
- 是否有一个最小可运行的验证方式（`/plugin validate` + 手动 `/名字` 跑一次）？
- 是否同步更新了相关 Skill 或说明？
- 是否只改动了完成任务所需的文件？

---

## 8. 团队共享与用插件分发

Skill 写完放哪，决定谁能用到。三种方式：

| 方式 | 生效范围 | 适合 | 代价 |
|---|---|---|---|
| `~/.claude/skills/<名字>/` | 只有你，所有项目 | 个人习惯、试验 | 换机器要重来，同事看不见 |
| `<仓库>/.claude/skills/<名字>/` | 跟着仓库走 | 团队成员共享的项目规则 | 每个仓库各存一份。注意本仓库 `.claude/`目前在 `.gitignore` 里，直接放进去**不会提交**，要共享得先决定怎么处理这条忽略规则 |
| 插件（marketplace） | 装了插件的所有人 | 跨项目复用、要版本和分发 | 多一层打包和发布流程 |

怎么选只看两件事：**要不要跨项目复用**、**要不要装到同事机器上**。都只在自己一个仓库里用，第一种就够。

> 别和本仓库的 `.agents/skills/` 搞混：那是 Codex / Pi 的位置，Claude Code 不认。同一份规则两边都要用，就在两个位置各放一份，或者用插件统一发。

插件这条路大致是：

```powershell
claude plugin init my-skill             # 在 ~/.claude/skills/my-skill/ 起骨架，下次启动按 my-skill@skills-dir 加载
claude plugin validate .claude/skills   # 校验目录里的 skill / agent / command 结构
claude plugin eval                      # 跑行为验证用例
```

会话里：

```text
/plugin marketplace add <owner/repo 或 URL>
/plugin                 # 插件主菜单：浏览、安装、启用/禁用（/plugins 是同一个命令）
/plugin help            # 看当前版本有哪些子命令
/reload-plugins         # 装完/改完让当前会话生效
```

插件带来的 Skill 在列表和调用里都带命名空间：`插件名:skill名`、`/插件名:skill名`。

**别为了一个任务装插件**：只要项目内的规则，用第 3 节的两个目录就够；只有确实要跨项目复用、要给同事装、要版本管理时，才值得走插件。

---

## 一页速查

```text
是什么      包含 SKILL.md 的目录；description 常驻，正文和附带文件按需加载
            正文上限 1 MB，且加载后整轮对话都占上下文

位置        ~/.claude/skills/<名字>/SKILL.md          用户级，所有项目
            <项目>/.claude/skills/<名字>/SKILL.md     项目级（项目根 + 当前目录）
            插件 skills/<名字>/SKILL.md               带 插件名: 前缀
            --plugin-dir <路径>                       临时调试插件

写           调用名 = 目录名（frontmatter 的 name 不一致不报错，靠自觉）
            description 写清"做什么 + 什么时候用"，把用户原话写进去
            正文命令式、讲清为什么；细节下沉 references/；确定性逻辑放 scripts/
            allowed-tools 只开需要的；要只手动调用就 disable-model-invocation: true

变量        ${CLAUDE_SKILL_DIR}   ${CLAUDE_PROJECT_DIR}   ${CLAUDE_PLUGIN_ROOT}
            $ARGUMENTS   $ARGUMENTS[0]   $1 / $2

调用        自动匹配 description  |  /名字 后面跟参数  |  插件用 /插件名:skill名

排查        /skills                        列表（用量 / 成本看插件管理器 Stats）
            /context                       上下文总量
            /plugin validate .claude/skills 结构校验
            /reload-skills                 拾取磁盘改动（插件用 /reload-plugins）
            claude --debug                 加载日志
            /skill-doctor 已并入插件管理器 Stats 标签页

控成本      skillOverrides: off | name-only | user-invocable-only
            disableBundledSkills 关掉内置 Skill

共享        个人 → ~/.claude/skills/    项目 → <仓库>/.claude/skills/    跨项目 → 插件
            注意：.agents/skills/ 是 Codex / Pi 的位置，Claude Code 不读
```
