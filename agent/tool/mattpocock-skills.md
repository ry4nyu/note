# mattpocock-skills：把真实工程流程写成 Skill

**仓库**：<https://github.com/mattpocock/skills>
**类型**：Skill 合集（27 个，MIT，作者 Matt Pocock / Total TypeScript）

一句话：**别人卖"全自动流程"，他卖的是一组小、可改、能拼的工程习惯**。README 里直接怼了 GSD / BMAD / Spec-Kit 那类框架——"它们帮你接管流程，代价是你失去控制权，流程本身出 bug 时你没法修"。这里每个 skill 都只有一个 `SKILL.md`，几十行到几百行，嫌不好直接改。

## 它解决的是哪几个问题

作者自己列的四个失败模式，正好对上四个 skill 组：

| 你遇到的现象 | 对应的 skill |
|---|---|
| 你说完需求，agent 做出来的东西根本不是你要的 | `/grill-me`、`/grill-with-docs` |
| agent 啰嗦，一个概念讲二十个词，代码里命名一天一个样 | `glossary` 机制（`/grill-with-docs` 顺手建 `GLOSSARY.md`） |
| 代码能跑但错，改完又错 | `/tdd`、`/diagnosing-bugs` |
| 越写越烂，改一处崩三处 | `/improve-codebase-architecture`、`/to-spec` 里的模块问答 |

### 它和别的 skill 合集不一样的地方

**分两层：user-invoked 管编排，model-invoked 管纪律。**

- **user-invoked**：只能你敲 `/xxx` 触发（frontmatter 里 `disable-model-invocation: true`），职责是串流程，比如 `grill-me` 的正文就一句话：调用 `grilling`。
- **model-invoked**：你和模型都能用，模型觉得任务对得上就自己加载。

规则是 user-invoked 可以调 model-invoked，反过来不行。**这条设计很实用**：编排层不会在你不想要的时候乱启动，纪律层又能被"流程"和"随手"两种方式复用。

## 没有它之前

| 现象 | 以前怎么对付 |
|---|---|
| 需求没对齐就开始写，写完发现理解错了 | 在 prompt 里多写几句，靠运气；或者自己先想清楚再让它动手 |
| 想让它"先问我几个问题" | 每次手写"先别写代码，先向我提问"，问两轮它就开始动手 |
| 代码库里的黑话（"materialization cascade"）agent 不懂，只能用一长串解释 | 每次会话都重新解释一遍；或者写进 `AGENTS.md`，越写越长 |
| 让它写测试，它先写实现再补测试（等于没写） | 手动要求"先写失败的测试"，盯几轮 |
| bug 复现靠猜，改完不知道有没有真修好 | 自己跑一遍、贴日志给它，来回几轮 |
| 一个需求拆成 ticket 全靠自己想 | 手写 issue，粒度全凭感觉 |
| 上下文快满了，新会话要从头讲 | 自己写交接笔记 |

关键区别是**这些规则是可执行的分支，不是口号**。比如 `diagnosing-bugs` 是个分阶段 gate 的循环（先建一个能变红的反馈回路 → 最小化 → 提假设 → 加观测 → 修 → 回归测试），而不是"请仔细调试"。

## 都装了什么

27 个，分 engineering（20 个）和 productivity（7 个）。挑主要的：

| Skill | 干什么 | 触发 |
|---|---|---|
| `grill-me` | 一直追问你，直到设计树的每个分支都有答案 | 你敲 |
| `grilling` | 上面那个追问行为的本体（`grill-me`、`/triage`、`/wayfinder` 都复用它） | 模型或你 |
| `grill-with-docs` | 边追问边建领域模型：磨术语、写 `GLOSSARY.md`、顺手记 ADR | 你敲 |
| `domain-modeling` | 单独跑领域模型那部分（挑战术语、压边界案例） | 模型或你 |
| `tdd` | 红-绿-重构，一次一个垂直切片 | 模型或你 |
| `diagnosing-bugs` | 难 bug / 性能回退的分阶段诊断循环 | 模型或你 |
| `code-review` | 两轴并行评审：Standards（团队规范 + Fowler 坏味道基线）、Spec（是否忠实实现原 issue） | 模型或你 |
| `to-spec` | 把当前对话压成 spec，发到 issue tracker | 你敲 |
| `to-tickets` | 把计划拆成 tracer-bullet ticket，标出阻塞关系 | 你敲 |
| `implement` | 按 spec / ticket 干活，在约定接缝处驱动 `/tdd`，收尾跑 `/code-review` | 你敲 |
| `implement-spec` | 整份 spec 在一个集成分支上做，多个 implementer 子 agent 并行跑 | 你敲 |
| `wayfinder` | 大到一次会话装不下的活儿，拆成一张决策 ticket 地图慢慢解 | 你敲 |
| `triage` | 用标签把 issue 在几个角色状态间推进 | 你敲 |
| `improve-codebase-architecture` | 扫一遍仓库找"加深模块"的机会，输出 HTML 报告，选一个再追问 | 你敲 |
| `codebase-design` | 深模块设计的共用词汇和纪律（小接口、干净接缝、从接口测） | 模型或你 |
| `prototype` | 建一次性原型回答设计问题：单文件 HTML，或几个差别很大的 UI 变体 | 模型或你 |
| `research` | 拿高可信一手资料查一个问题，落成带引用的 Markdown | 模型或你 |
| `pr` | PR 正文该长什么样：一张最小可视化 + before/after 证据 + 合并风险判断 | 模型或你 |
| `wizard` | 生成一个交互式 bash 向导，带你走只有人能做的步骤（开凭据、配 CI secret） | 模型或你 |
| `retro` | 会话结束后提改进建议（导航、自动检查、规范、steering 文件、工具），按严重程度排 | 你敲 |
| `ask-matt` | 不知道该用哪个 skill，让它给你路由 | 你敲 |
| `handoff` | 把当前对话压成交接文档给下一个 agent | 你敲 |
| `teach` | 用当前目录当教学工作区，多会话教你一个概念 | 你敲 |
| `wait-what` | 哪句话没听懂，让它用 `GLOSSARY.md` 里的词重新讲一遍 | 你敲 |
| `to-questionnaire` | 你答不了的决定，做成 Markdown 问卷发给唯一能答的人 | 你敲 |
| `writing-for-agents` | 写 skill、`AGENTS.md`/`CLAUDE.md` 这类"给 agent 读的文档"的方法 | 模型或你 |
| `setup-matt-pocock-skills` | 每个仓库跑一次：问用哪个 issue tracker、triage 用什么标签、文档放哪 | 你敲 |

仓库里还有 `skills/misc/`（4 个，如 `setup-pre-commit`）、`skills/in-progress/`（7 个）、`skills/deprecated/`（空）。**装的时候只拿 `engineering` 和 `productivity`**。

## 优点 / 缺点

| 优点 | 缺点 |
|---|---|
| 每个 skill 都小、纯 Markdown，改起来没有心理负担 | **重度依赖 issue tracker**：`triage`、`to-spec`、`to-tickets`、`implement` 都往 tracker 写；Jira 不在官方列表里，只能走"其它（你自己描述）" |
| 编排层和纪律层分开，`disable-model-invocation` 让流程不会被模型自己启动 | **27 个全装很占菜单和上下文**（每个 skill 都有一条 description 进系统提示） |
| `/grill-me` 这类"先问再做"确实解决最常见的失败模式 | 追问会很想跳过：一轮十几个问题，赶时间的时候很烦 |
| 会往你仓库里写文件：`GLOSSARY.md`、ADR、spec、文档 | 也是往仓库里**写文件**：目录结构不由你定，每个 repo 都要跑一次 setup |
| 不绑定模型，Claude / Codex / Copilot / Gemini / Pi 都有安装路径 | 全部英文提示词，输出语言跟模型走；要中文得自己在 `AGENTS.md` 里要求 |
| MIT，随时可以 fork 成自己团队的一套 | 迭代快（changeset 管理版本），命令和 skill 名可能变 |

## 安装

**每种 agent 只选一种装法。** README 明确写：插件和 `skills` CLI 都装 = 每个 skill 出现两次。

### Claude Code（推荐，自动更新）

```bash
claude plugin install mattpocock-skills@claude-plugins-official
```

也可以走作者自己的 marketplace：

```bash
/plugin marketplace add mattpocock/skills
/plugin install mattpocock-skills@mattpocock
```

### Codex

```bash
codex plugin marketplace add mattpocock/skills
codex plugin add mattpocock-skills@mattpocock
```

启动时自动更新。

### Pi（本仓库的写法）

官方列了 `pi` 作为 `skills` CLI 支持的 agent：

```bash
npx skills@latest add mattpocock/skills -a pi    # 安装时勾上 setup-matt-pocock-skills
npx skills@latest update                         # 手动更新；有新 skill 再跑一次 add
```

包方式（Pi 会从仓库的 `skills/` 目录发现 Skill）：

```bash
pi install git:github.com/mattpocock/skills
pi list
```

> 包方式我没实测：这个仓库**没有 `pi` 清单**，全靠约定目录发现，有概率连带扫到 `skills/misc/`、`skills/in-progress/`。装完先看 `/` 菜单里多了哪些，多出来的在 `settings.json` 里用 `skills` 数组收窄。

### 其它 agent

```bash
npx skills@latest add mattpocock/skills -a cursor   # 也支持 opencode / windsurf / amp / devin 等，省略 -a 会问
gemini skills install https://github.com/mattpocock/skills.git --path skills/engineering
gemini skills install https://github.com/mattpocock/skills.git --path skills/productivity
```

### 装完必须跑一次 setup

```text
/setup-matt-pocock-skills
```

**每个仓库跑一次**，它会问你三件事：

1. 用哪个 issue tracker（GitHub / GitLab / 本地文件 / 你描述的其它）
2. triage 时你会打什么标签（`/triage` 靠这个）
3. 生成的文档放哪

跳过这步，`/triage`、`/to-spec`、`/to-tickets` 都会停下来问或者行为不对。

## 三个 agent 怎么接

| Agent | 装法 | 调用 |
|---|---|---|
| Pi | `npx skills@latest add mattpocock/skills -a pi`，或 `pi install git:github.com/mattpocock/skills` | `/skill:grill-me`；model-invoked 的（`tdd`、`code-review`）也可以直接说"按 tdd 来" |
| Claude Code | `claude plugin install mattpocock-skills@claude-plugins-official` | `/grill-me`、`/tdd`、`/code-review` |
| Codex | `codex plugin marketplace add mattpocock/skills` → `codex plugin add mattpocock-skills@mattpocock` | 插件自带命令 |

三家共用的两点：

- **`disable-model-invocation: true` 三家都认**（Pi 在 `skill使用指导.md` 第 3 节也写明了），所以"只能你敲"的那批不会自己跑起来。
- **软链 vs 拷贝**：`skills` CLI 支持软链和拷贝，想跟着上游更新就软链；想在团队里改出一套自己的（作者自己说 "make them your own"），就拷进仓库 `<仓库>/.claude/skills/` 或 `.agents/skills/` 再改。

## 用起来

几个真正常用的入口：

```text
/grill-me 我想给 CLI 加个 --watch 模式，参数设计上帮我挑毛病
```

```text
/grill-with-docs 重做一下上传流程，边问边把 GLOSSARY.md 和 ADR 更新了
```

它会一边追问一边把术语固化下来：之后你和 agent 说的都是同一套词，变量名、文件名也跟着统一——这点比"少说几个字"值钱。

```text
/improve-codebase-architecture
```

扫完给一份 HTML 报告列出候选，你挑一个它再追问细节。作者的建议是**每隔几天跑一次**，当体检不当抢救。

```text
/to-spec                     # 把刚才聊的压成 spec
/to-tickets                  # 拆成带阻塞关系的 ticket
/implement                   # 开工，收尾自动 /code-review
/handoff                     # 上下文要满了，生成交接文档
```

## 踩坑清单

- **两个都装 skill 出现两份**：插件和 `skills` CLI 二选一，换装法先把旧的删掉。
- **`/triage`、`/to-spec` 行为怪**：先确认 `/setup-matt-pocock-skills` 在这个仓库跑过。
- **它一直在问问题**：`/grill-me` 正常就是一问一大轮（它一次问完"当前能问的全部"）。不想深入就在开头说清边界，或者干脆跳过这步用 `/to-spec`。
- **文档位置被它接管**：`GLOSSARY.md`、ADR、spec 的目录是 setup 时定的，不满意就重跑 setup 或手动改生成的配置。
- **团队用 Jira / 内部 tracker**：setup 里选"其它"自己描述，能不能跑通要看模型理解得多准，别指望和 GitHub 一样顺。
- **上下文吃紧**：27 个 skill 全装，光 description 就是一笔固定开销。只留你真正会敲的那几个，删目录即可。
- 上面的命令是抄各自 README 的，我没在三个 agent 上逐个实测；版本以你安装时的输出为准。

**不适合**：想找一个"点一下就全自动"的框架（这套的立场正好相反）；小脚本、一次性改动（追问和拆票比干活还贵）。

**适合谁**：在做长期维护的项目、并且愿意在动手前花十分钟把需求问清楚的人。跟 [`ponytail.md`](./ponytail.md) 不冲突：ponytail 管"少写"，这套管"先想清楚再写"。

**我的看法**：这套里最值钱的是 `grill-me` / `grill-with-docs` 和 `diagnosing-bugs` 三个，`GLOSSARY.md` 那个思路值得单独抄进项目 `AGENTS.md`。其余的先别全装——按周往身上加，比一次装 27 个稳。
