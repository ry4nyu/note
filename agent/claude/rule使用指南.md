# Claude Code Rule 使用指南

本文面向本仓库团队成员，说明怎么在 Claude Code 里组织**项目规则与记忆**：`CLAUDE.md`、`.claude/rules/`、`paths` 作用域，以及自动记忆。文中机制基于 Claude Code 2.1.x（Windows + PowerShell 环境），跟随版本变化，遇到不一致先敲 `/memory` 看本次实际加载了什么。

前置阅读：[`README.md`](./README.md)（安装、配置、权限模式；第 6 节是 `CLAUDE.md` 入门）。「什么时候该用 Rule、什么时候该用别的」见 [`skill使用指南.md`](./skill使用指南.md) 第 2 节和 [`Hook使用指南.md`](./Hook使用指南.md) 第 1 节。Codex / Pi 的对应物是本仓库的 `.agents/AGENTS.md` 和 `.agents/spec/`，**Claude Code 不读那两个位置**（第 8 节）。

> **关于本文的事实来源**
>
> - 第 2、3、4 节的加载路径、"递归且只认 `.md`"、`paths` 的过滤逻辑、`claudeMdExcludes` 语义，是逐条核对本机 `claude.exe` 2.1.268 的反编译字符串得到的（第 7 节给了你自己复核的办法）。这部分当实测看。
> - 大括号展开、pattern 预算、4 MiB 上限、自动记忆的 200 行限额、`paths` 的 YAML 示例写法来自官方 memory 文档，**没有在本机复现**，按参考值看待。
> - 社区有反馈说 `paths` 的**数组**写法在部分版本被静默忽略（issue #13905）。**这条我没有核实过**，所以本文不把它当结论 —— 第 7 节给了确认规则到底加载没加载的办法，以你的实测为准。
> - 一个常见混淆先说明：`.cursor/rules` 里的 `alwaysApply` / `globs` 是 **Cursor 的字段，Claude Code 不认**，别照搬。

---

## 1. Rule 是什么

Rule 是 Claude Code 的**项目约定载体**：放在固定位置、会被自动放进上下文的 Markdown 文件。它不提供新工具、不执行命令，只影响模型"默认该怎么做"。

Claude Code 里能固化规则的地方不止一处，先分清再动手：

| 你想要的效果 | 该用哪个 | 为什么 |
|---|---|---|
| 每次都必须发生，不依赖模型判断 | **Hook**（[`Hook使用指南.md`](./Hook使用指南.md)） | 由 harness 执行，不经过模型，写了就一定跑 |
| 恒真、且很短的项目事实 | **CLAUDE.md**（[`README.md`](./README.md) 第 6 节） | 每轮都在上下文里，代价是每轮都付 token |
| 成体系、要分文件、要按路径生效的约定 | **Rule（本文）** | 自动加载；`paths` 让它只在相关文件上生效 |
| 一类任务的流程、检查单、边界 | **Skill**（[`skill使用指南.md`](./skill使用指南.md)） | 平时只占一行描述，需要时才展开 |
| 要接入新工具或数据源 | **MCP server**（[`mcp使用指南.md`](./mcp使用指南.md)） | 提供的是工具本身 |

一句话版：**恒真且短的放 CLAUDE.md，成体系的放 rules，只想在特定文件上生效的加 `paths`，必须每次发生的放 Hook。**

> **Rule 是上下文，不是强制。** 模型读到规则不等于一定照做，尤其在你和它上下文都很满的时候。真正需要卡住的用 Hook、权限系统、测试或 CI 门禁 —— 见第 7 节的反模式。

适合写进 Rule 的：分层与命名规范、日志/异常约定、某个目录的特殊规则、测试要求、安全边界说明。
不适合的：必须阻断的操作（用 Hook）、一次性的临时提醒（直接说）、密码和密钥（哪都别写）。

---

## 2. 四层记忆

Claude Code 按四个层级收集指令文件，从宽到窄：

| 层级 | 位置（Windows） | 作用范围 | 能提交给同事吗 |
|---|---|---|---|
| **Managed** | 托管策略目录 / `C:\Program Files\ClaudeCode\CLAUDE.md` | 全组织 | 由管理员下发 |
| **User** | `C:\Users\你的名字\.claude\CLAUDE.md` 和 `.claude\rules\` | 你所有项目 | 否，只在本机 |
| **Project** | `<项目>\CLAUDE.md`、`<项目>\.claude\CLAUDE.md`、`<项目>\.claude\rules\` | 单个项目 | **能**，跟着仓库走 |
| **Local** | `<项目>\CLAUDE.local.md` | 单个项目 | 否，个人用 |

**优先级从低到高：Managed → User → Project → Local。** 后面的覆盖前面的。托管层永远生效，而且**排除不掉**。

加载方式：从当前工作目录往上层目录走，把各层的文件从根往下拼起来。另外，用 `--add-dir`（或 `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD`）追加的目录，也会带上它自己的 `CLAUDE.md` 和 `.claude/rules/`。

几条容易踩的：

- **Project 覆盖 User** 是设计如此，不是 bug。想让某条规范在所有项目都生效，写 User 层；想让它跟着某个仓库走，写 Project 层。
- **`CLAUDE.local.md` 是给"我不想提交"的个人覆盖用的**，别把团队规范写这里。
- **本仓库的 `.claude/` 在 `.gitignore` 第 1 行** —— 项目级 rules 直接放进去**不会被提交**，见第 8 节。

---

## 3. `.claude/rules/` 目录机制

### 3.1 放在哪

| 位置 | 作用范围 | 说明 |
|---|---|---|
| `~/.claude/rules/*.md` | 用户级 | 对你所有项目生效；想跨项目复用又不想走审批，放这里 |
| `<项目>/.claude/rules/*.md` | 项目级 | 跟仓库走，团队共享的主力位置 |
| 托管策略的 rules 目录 | 全组织 | 管理员下发，**排除不掉** |
| `--add-dir` 追加目录的 `.claude/rules/` | 该目录 | 随追加目录一起被收集 |

### 3.2 只认 `.md`，而且是递归的

- 目录**递归扫描**，但**只加载 `.md` 文件**，其他后缀一律不进上下文。
- 所以索引文件要叫 **`INDEX.txt` 而不是 `README.md`**。用 `.md` 写索引，它会被当成一条规则一起加载，白占上下文；用 `.txt` 人能读到，Claude 不看。
- 子目录随便分（`database/`、`api/`、`testing/`…），**目录结构本身不影响加载行为**，只影响你自己找文件。

```text
.claude/rules/
├─ INDEX.txt              # 给人看的目录/变更记录（.txt 不会被加载）
├─ code-style.md          # 不带 paths：常驻
├─ testing.md             # 不带 paths：常驻
├─ security.md            # 不带 paths：常驻
└─ backend/
   ├─ mapper.md           # 带 paths：只在读 Mapper 时加载
   └─ scheduler.md        # 带 paths：只在读定时任务时加载
```

### 3.3 一个域一个文件

别写 `everything.md`。文件越聚焦，越可能只在真需要的时候被加载，模型对它的遵循度也越高。推荐按 `code-style.md` / `testing.md` / `security.md` 这样切，一个文件回答一类问题。

### 3.4 符号链接与"外部引入"

`.claude/rules/` 支持软链，所以你可以把一份共享规则链进多个仓库。两条边界：

- 指向**工作目录之外**的软链，按"外部引入"对待，**需要你先批准**，而且批准后也**只有不带 `paths` 的规则会被加载**。
- 想跨项目共享又不想走审批流程，直接放 `~/.claude/rules/`，它对这台机器上所有项目生效。
- 循环软链会被检测并跳过。

---

## 4. 写一条 Rule

Rule 就是普通 Markdown，靠 `paths` frontmatter 控制**什么时候加载**：

| 写法 | 加载时机 |
|---|---|
| 不写 frontmatter，或不写 `paths` | **会话启动就加载**，与同层 `.claude/CLAUDE.md` 同优先级 |
| 写了 `paths` | **只在模型碰到匹配的文件时加载** |

```markdown
---
paths:
  - "src/main/java/**/mapper/**/*.java"
---

# Mapper 层约定

- 只写 SQL 映射，不放业务判断，业务逻辑上移到 Service。
- 复杂查询写在 XML 里，不要用注解拼字符串。
- 新增查询必须同时在 `src/main/resources/mapper/` 里加对应 XML。
- 返回集合的查询显式写 `resultMap`，不要靠下划线转驼峰蒙。
```

不带 `paths` 的常驻规则长这样：

```markdown
# 代码规范

- 分层：Controller / Service / Mapper，不要跨层调用。
- 统一返回体 `Result<T>`，不要直接返回实体。
- 日志统一用 `@Slf4j`，禁止 `System.out.println`。
- 新增接口必须补 Swagger 注解。
```

怎么选：

- **天天都会用到的**（分层、命名、日志、提交规范）→ 不带 `paths`，启动就加载。
- **只有动某类文件才需要的**（某个目录的特殊约定、某种框架的写法）→ 带 `paths`，别让它常驻。
- `paths` 匹配的是**模型读到的文件路径**，用 glob：`src/api/**/*.ts`、`src/**/*.{ts,tsx}`。大括号会展开成多条模式。

> **别用 `@` 引用来省上下文。** `@path/to/file` 只是把内容引进来，**启动时照样全量加载**，嵌套最多 5 跳。真正能省上下文的手段是 `paths`。

> **写规则用命令式，并且解释为什么。** 堆 `ALWAYS` / `MUST` 的效果通常不如把道理说清楚 —— 这一点和 [`skill使用指南.md`](./skill使用指南.md) 4.4 节是同一个道理。

---

## 5. 加载顺序与优先级

- **层与层**：Managed → User → Project → Local，后面的覆盖前面的。
- **同层内**：`CLAUDE.md` 先、`CLAUDE.local.md` 后；不带 `paths` 的 rule 与同层的 `.claude/CLAUDE.md` 同优先级。
- **子目录里的 `CLAUDE.md`**：不在启动时加载，等你进到那个目录干活时才加载。这也是"按需加载"的另一种做法，代价是规则散在代码树里。
- **`/compact` 之后**：项目根的 `CLAUDE.md` 会被重新读取注入；`paths` 作用域的规则会随着你后续再读到匹配文件而重新生效。

> **真出现冲突时的排查顺序**：先 `/memory` 看加载了哪些文件、顺序如何，再回头改 —— 多层规则互相打架是"模型表现不稳"最常见的根因之一。

---

## 6. 自动记忆（auto memory）

这跟手写规则是两码事，别混：

| | 手写 Rule / CLAUDE.md | 自动记忆 |
|---|---|---|
| 谁写的 | 你 | Claude 自己 |
| 放哪 | 项目里 / `~/.claude/rules/` | `~/.claude/projects/<项目>/memory/` |
| 进 git 吗 | 项目级可以 | 不进 |
| 怎么写入 | 直接编辑文件 | 让模型自己记；也可用 `#内容` 快速记一条 |
| 怎么看 | 直接打开文件 | `/memory` |

- 默认开启。`MEMORY.md` 是索引，每条记忆一个文件，带 `name` / `description` / `type` 三类 frontmatter。
- 每次会话只加载 `MEMORY.md` 的**前 200 行（或 25 KB，取小的那个）**，其余按需读取。
- 关掉：`/memory` 里切换，或 settings 里的 `autoMemoryEnabled`，或环境变量 `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1`。

> **`#内容` 到底写到哪个文件，我没有逐版本核实过。** 对话里用它快速记一条是没问题的，但**落到 `CLAUDE.md` 还是自动记忆目录，以 `/memory` 里实际显示的为准**，别假定。

> 自动记忆是"模型自己攒的笔记"，**不能替代你写进 rules 的团队约定**：它不进仓库、同事看不到、内容也不经过你审。团队规范一律落到 rules 或 `CLAUDE.md`。

---

## 7. 验证与排查

**先跑这几条：**

```text
/memory        本次会话实际加载了哪些指令文件，可查看/编辑；也在这里开关自动记忆
/context       看当前上下文总量，判断是不是规则吃太多
/doctor        体检，会专门指出重复或臃肿的记忆文件
claude --debug 看加载过程的日志
```

**要确认某条规则到底有没有加载**，用 `InstructionsLoaded` hook 把它打出来。它每加载一个指令文件就触发一次，stdin 收到的 JSON 关键字段：

| 字段 | 含义 |
|---|---|
| `file_path` | 被加载的文件 |
| `memory_type` | `User` / `Project` / `Local` / `Managed` |
| `load_reason` | `session_start` / `nested_traversal` / `path_glob_match` / `include` / `compact` |
| `globs` | 命中的 `paths` 模式（没写 `paths` 时为空） |
| `trigger_file_path` | 触发这次加载的文件（`path_glob_match` 时才有） |

预期结果：**不带 `paths` 的规则**在会话一开始就出现，`load_reason` 是 `session_start`；**带 `paths` 的规则**一开始不出现，等你让它读一个匹配的文件后才出现，`load_reason` 是 `path_glob_match`。如果一条规则从来没出现过，就是没被加载 —— 回去查后缀是不是 `.md`、路径对不对、`paths` 写法有没有被解析成 pattern。

Hook 的配置格式和脚本写法见 [`Hook使用指南.md`](./Hook使用指南.md) 第 3、8 节。

**排除掉不想加载的文件** —— 写在 `C:\Users\你的名字\.claude\settings.json`：

```json
{
  "claudeMdExcludes": [
    "**/monorepo/other-team/CLAUDE.md",
    "/home/user/monorepo/other-team/.claude/rules/**"
  ]
}
```

- 按**绝对路径**匹配（picomatch），不是相对路径。
- **只对 User / Project / Local 生效**，托管层的文件排除不掉。
- 多层都写了会合并生效。

**常见现象对照：**

| 现象 | 先查什么 |
|---|---|
| 写了规则但模型不照做 | Rule 只是上下文，不保证执行。要卡死的改用 Hook、权限规则或测试门禁 |
| 规则像是压根没加载 | `/memory` 看有没有列出来；后缀是不是 `.md`（`.txt` 不加载）；`paths` 写对没；用 `InstructionsLoaded` 看 `load_reason` |
| 上下文很大但没几条规则 | rules 目录里放了 `README.md` 之类给人看的文件 → 改成 `.txt` |
| 规则被别的文件盖掉 | `/memory` 看加载顺序；多层冲突时 Project 覆盖 User |
| 项目级 rules 提交不上去 | 本仓库 `.claude/` 在 `.gitignore` 里（第 8 节） |
| 不确定 `paths` 有没有被解析 | 看 `InstructionsLoaded` 的 `globs` 字段；为空说明没解析成模式，等于这条规则既没常驻也没生效 |
| 改动不生效 | 直接改文件通常会被拾取；没生效就重启一次会话强制重新加载 |

---

## 8. 团队规范

- **一个规则文件只讲一个域**，不把数据库、接口、测试揉进一个 `everything.md`。
- **引用而不是复制**：指向权威文档的位置，别把整篇内容拷进来 —— 复制出来的副本迟早和上游不一致，还白占上下文。
- **不写密码、Token、私钥**；示例路径用 `C:\Users\你的名字\...` 这类占位符。
- **给人看的索引和变更记录用 `.txt`**，别用 `.md`。
- **写完跑一次 `/memory`**，确认加载范围和你以为的一致。

**怎么共享给同事：**

| 方式 | 生效范围 | 代价 |
|---|---|---|
| 直接放 `~/.claude/` | 只有你，所有项目 | 换机器要重来，同事看不见 |
| 提交到仓库 `.claude/` | 跟着仓库走 | **本仓库 `.claude/` 在 `.gitignore` 里，直接放进去不会提交**，要共享得先决定怎么处理这条忽略规则 |
| 托管策略 | 全组织 | 要管理员下发 |

> 别和 `.agents/` 搞混：`.agents/AGENTS.md` 和 `.agents/spec/agent/*.md` 是 **Codex / Pi** 的规则位置，**Claude Code 不读**。同一份约定两边都要用，就在两个位置各放一份。

推荐的做法是：项目级 rules 提交进仓库（先解决 gitignore），个人习惯放 `~/.claude/rules/`，别用自动记忆承载团队约定。

**提交前检查：**

- 是否一个文件只讲一个域？
- 是否真的需要常驻？能不能加 `paths` 把范围缩小？
- 有没有混进密钥、内网地址、真实账号？
- `/memory` 里确认加载符合预期了吗？
- 是否只改动了完成任务所需的文件？

---

## 一页速查

```text
是什么      会自动进上下文的 Markdown 规则；不强制、不执行，只影响模型默认怎么做

层级        Managed   托管策略，全组织，排不掉
            User      ~/.claude/CLAUDE.md + ~/.claude/rules/                 所有项目
            Project   <项目>/CLAUDE.md、.claude/CLAUDE.md、.claude/rules/    跟仓库走
            Local     <项目>/CLAUDE.local.md                               个人
            优先级（低→高）：Managed → User → Project → Local

目录        递归扫描，只加载 .md；给人看的索引用 INDEX.txt 不要用 README.md
            ~/.claude/rules/*.md             用户级
            <项目>/.claude/rules/*.md         项目级（本仓库 .claude/ 被 gitignore）

paths       不写 → 会话启动就加载（与同层 .claude/CLAUDE.md 同优先级）
            写了 → 只在碰到匹配文件时加载
            glob：src/api/**/*.ts、src/**/*.{ts,tsx}
            ⚠ @引用省不了上下文，启动仍全量加载（最多 5 跳）
            ⚠ alwaysApply / globs 是 Cursor 的字段，Claude Code 不认

自动记忆    ~/.claude/projects/<项目>/memory/ + MEMORY.md 索引
            只加载前 200 行 / 25KB；#内容 快速记一条；/memory 查看与开关
            不进 git，不能替代团队 rules

排查        /memory                     本次加载了哪些指令文件
            /context                    上下文总量
            /doctor                     体检，指出重复/臃肿的记忆文件
            InstructionsLoaded hook     看 file_path / memory_type / load_reason / globs
            claudeMdExcludes            按绝对路径排除（Managed 排不掉）
            claude --debug              加载日志

共享        个人 ~/.claude/   项目 <仓库>/.claude/   全组织 托管策略
            注意：.agents/ 是 Codex / Pi 的位置，Claude Code 不读
```
