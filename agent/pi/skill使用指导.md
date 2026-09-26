# Pi Skill 使用指导

本文面向本仓库团队成员，说明如何在 Pi 里选择、使用和维护 Skill。Pi 相关机制基于 Pi 0.87.x，命令和默认行为随版本变化，遇到不一致先执行 `pi --help`。

前置阅读：还没装 Pi 看 [`README.md`](./README.md)；Skill 在 Pi 整体扩展体系里的位置见该文第 9 节。

## 1. Skill 是什么

Skill 是一组可复用的 Agent 工作规则，通常用于固定某类任务的触发条件、执行顺序、安全边界和验证方式。Pi 实现 [Agent Skills 规范](https://agentskills.io/specification)，一个 Skill 就是一个包含 `SKILL.md` 的目录，可以顺带打包脚本、参考资料和模板：

```text
pdf-tools/
├─ SKILL.md          # 必需：frontmatter + 正文指令
├─ scripts/          # 可选：可执行脚本
├─ references/       # 可选：按需读取的参考资料
└─ assets/           # 可选：模板等资源
```

关键点是**按需加载**：Pi 启动时只把每个 Skill 的 `name`、`description` 和路径放进系统提示，**不会把 `SKILL.md` 全文塞进上下文**。只有任务命中时，模型才去读 `SKILL.md` 并照做。所以 Skill 适合重复出现、需要稳定流程的工作；一次性的小任务不需要为了"走流程"增加额外步骤。

## 2. 什么时候使用 Skill

- 用户明确写出 Skill 名称、`$skill-name` 或 `/skill:name` 时，必须使用该 Skill。
- 当前任务明显符合某个 Skill 的描述时，使用它。
- 多个 Skill 同时适用时，只选择覆盖任务所需的最小集合，并说明使用顺序。
- 使用前完整阅读对应的 `SKILL.md`；其中引用了其他规则或脚本时，只继续读取任务需要的直接引用。
- Skill 不适用时，不要为了"走流程"强行调用；说明原因并采用最简单的可行方案。

示例：

```text
/skill:github-cli 查看当前仓库未关闭的 issue，并按优先级汇总
/skill:kit-manage 把这段重复的路径解析逻辑抽成 kit
```

> 模型可能漏加载相关 Skill。需要强制触发时，用 `/skill:name` 显式调用，后面跟的参数会作为用户请求追加到指令之后。

## 3. 如何安装和调用

### Skill 从哪些位置加载

Pi 会扫描下列位置，包含 `SKILL.md` 的目录会被**递归发现**：

| 位置 | 作用范围 | 说明 |
|---|---|---|
| `~/.pi/agent/skills/` | 用户级 | 对所有项目生效，和 Pi 的设置目录同级 |
| 项目 `.pi/skills/` | 项目级 | 受**项目信任**保护（见下） |
| `~/.agents/skills/` | 用户级 | Agent Skills 规范位置 |
| `.agents/skills/` | 项目级 | 本仓库用的就是这个；从工作目录向上、到仓库根为止逐级查找 |
| Pi 包（`packages`） | 按包 | 用包分发一个或多个 Skill |

本仓库的项目级 Skill 放在：

```text
.agents/skills/<skill-name>/SKILL.md
```

当前仓库已装：`agent-perf`、`github-cli`、`kit-manage`。`settings.json` 的 `skills` 数组还可以额外指定 Skill 文件或目录。

### 项目信任是前提

项目级 Skill 里的文件**可能会让模型执行脚本或改文件**，所以 Pi 在加载项目本地资源前必须先拿到信任决定（见 [`README.md`](./README.md) 第 5.2 节）。没信任时项目 Skill 不会生效：

```powershell
pi --approve        # 本次直接信任项目资源
pi --no-approve     # 本次完全不加载项目本地资源
```

交互界面里用 `/trust` 保存决定，之后不再问。**装载陌生人仓库的 Skill 前一定先看内容**。

### 调用方式

自动匹配，或强制调用：

```text
/skill:pdf-tools extract report.pdf     # 参数会追加到已加载指令之后
```

其它与调用相关的开关：

| 方式 | 作用 |
|---|---|
| `pi --skill <path>` | 临时加载一个 Skill，不写进配置 |
| `enableSkillCommands`（settings） | 控制 Skill 是否出现在 `/` 命令菜单；设为 `false` 时手动输入 `/skill:name` 仍可用 |
| `SKILL.md` 里 `disable-model-invocation: true` | 只允许显式命令调用，禁止模型自动选中 |
| `/reload` | 编辑 Skill 后在当前会话里重新加载 |

### 第三方 Skill 与分发

按 Skill 项目的说明安装。例如用 Skills CLI 全局安装：

```powershell
npx skills add <owner>/<repository> -g
```

全局安装适合个人长期使用；**团队共享的规则应优先放进仓库并纳入版本控制**，而不是各自全局装一份。

要在团队或跨项目分发一组 Skill，用 [Pi 包](https://github.com/earendil-works/pi-coding-agent)（`pi install <source>`）通过 npm 或 git 发布，并保持环境初始化、运行时依赖声明在包内。

## 4. 目录与 frontmatter

`SKILL.md` 以 frontmatter 开头，正文直接写指令：

```markdown
---
name: pdf-tools
description: Extract text and tables from PDF files. Use when reading, converting, or inspecting PDFs.
---

# PDF tools

读取要转换的文档之前先看 `references/formats.md`。脚本路径相对于本 Skill 目录执行。
```

Agent Skills 规范定义的字段：

| 字段 | 用途 |
|---|---|
| `name` | 命令名和显示名 |
| `description` | 给模型用的路由描述，决定何时加载 |
| `license` | 许可证名称或随附的许可证文件 |
| `compatibility` | 环境要求 |
| `metadata` | 附加键值元数据 |
| `allowed-tools` | 实验性的预授权工具列表 |
| `disable-model-invocation` | 对模型隐藏该 Skill，只能显式调用 |

写 frontmatter 的几条硬规则：

- **`name`** 用小写字母、数字和连字符，首尾和中间都不能出现连续连字符，最长 64 字符。
- **`description`** 最长 1024 字符，必须写清"做什么"和"什么时候适用"。像 `Helps with PDFs` 这种给不出路由信息的描述等于没写。
- **必须有 `description`**，格式错误或缺 description 的 Skill 不会被加载。
- 同名冲突时保留**先发现**的那个，并给出告警。
- Pi 不强制 `name` 和父目录同名，但其它 Agent Skills 实现可能强制，**保持同名更可移植**。
- frontmatter 里的引用文件用**相对 Skill 目录的路径**；Pi 会告诉模型 Skill 的位置，让它自己解析。

## 5. Skill 和 Kit 怎么分工

先检查仓库中是否已有对应 Skill 或 Kit，优先复用。

Skill 负责：

- 理解用户意图和任务边界；
- 编排步骤、权限和安全要求；
- 决定何时调用工具；
- 解释和验证工具结果。

Kit 负责可重复且需要稳定实现的操作，例如：

- 解析和校验输入；
- 路径处理、过滤和分页；
- API 参数构造；
- 稳定的 JSON 或其他结构化输出。

如果一个流程需要上述能力，不要把大量命令解析和业务逻辑塞进 Skill。先阅读 [`.agents/spec/agent/skill.md`](../../.agents/spec/agent/skill.md) 和 [`.agents/spec/agent/kit.md`](../../.agents/spec/agent/kit.md)，再按 `/skill:kit-manage` 的流程创建或更新 Kit，并同步对应 Skill。

不要为一次性命令、薄别名或几行简单操作创建 Kit。

## 6. 团队使用规范

- 先找已有 Skill、Kit 和仓库工具，再考虑新增实现。
- 一个 Skill 只解决一类清晰问题，不把多个无关工作流揉在一起。
- 不在 Skill 或文档中提交密码、Token、私钥及其他敏感信息。
- 不用扩大沙箱或跳过审批来掩盖范围不清的问题；先缩小任务和权限。
- 需要修改文件时，明确修改范围、不要触碰的内容和验证命令。
- Skill 中的脚本、模板和资源应优先复用，不要在提示词中重复维护同一份逻辑。
- `.agents/` 下的 Agent 可读文件使用英文；面向团队的 `agent/` 文档可以使用中文。

推荐的任务提示词：

```text
/skill:github-cli 只处理登录模块。先说明根因，再做最小修改，最后运行相关测试；不要做无关重构。
```

## 7. 常见问题排查

### Skill 没有被调用

1. 确认 Skill 名称和目录名一致。
2. 确认文件路径为 `.agents/skills/<name>/SKILL.md`（或第 3 节里的其它加载位置）。
3. 检查 frontmatter 是否完整，尤其是 `name` 和 `description`。
4. **确认当前项目已被信任**——没信任时项目级 Skill 不加载（用 `/trust` 或 `pi --approve`）。
5. 在对话中显式使用 `/skill:name`，排除自动匹配描述不准确的影响。
6. 编辑后 `/reload`；名称冲突时确认保留的是哪一个（启动诊断会告警）。

### Skill 做了不该做的事

检查 `description` 和正文是否写得过宽。补充"不适用场景"、输入边界、权限限制和完成后的验证要求，而不是在每个调用方增加临时补丁。

### Skill 越写越长

删除重复的命令说明和通用常识。稳定的解析、校验和输出逻辑移到 Kit；Skill 只保留路由、编排和解释结果所需的规则。

### 启动时提示 Skill 无效 / 被跳过

按第 4 节的规则逐条核对：`name` 的字符与长度、是否缺 `description`、frontmatter 是否是合法 YAML。Pi 对大多数非法字段只告警不停止启动，所以"没报错"不代表"加载了"——以 `/` 命令菜单和启动头部列出的资源为准。

## 8. 提交前检查

- 是否复用了已有 Skill 或 Kit？
- 触发条件和不适用场景是否明确？
- 是否包含最小权限和敏感信息约束？
- 是否有一个最小可运行的验证方式？
- 是否同步更新了相关 Skill、Kit 或说明？
- 是否只改动了完成任务所需的文件？
