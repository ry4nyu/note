# ponytail：让 Agent 别过度设计

**仓库**：<https://github.com/DietrichGebert/ponytail>
**类型**：Skill

名字来自"公司里那位扎马尾的老程序员"——你甩给他五十行代码，他啥也不说，给你换成三行，还能跑。

这个 skill 专治 agent 的**过度设计**：你要个日期选择器，它给你装个 flatpickr、写个包装组件、加个样式文件，再顺便跟你讨论时区问题。ponytail 的规则是"只写任务需要的"，同时明确**不许砍校验、错误处理、安全和无障碍**。

官方在真实 Claude Code 会话上测的对比（vs 不装 skill）：

| | 代码量 | token | 成本 | 耗时 |
|---|--:|--:|--:|--:|
| ponytail | **-54%** | -22% | -20% | -27% |

> 百分比为项目 README 自述，未本地复现。

## 没有它之前

| 现象 | 以前怎么对付 |
|---|---|
| 一句话需求换来 5 个新文件、一个抽象层、两个没人用的配置项 | 评审时一个个删，或者干脆自己重写 |
| "要不要加点扩展性"——加完发现永远只有一个调用方 | 事后清理，下次还这样 |
| 为了一个 `date` 字段引入第三方库 | 提要求时反复强调"别加依赖" |
| 想给 agent 立规矩 | 在 `AGENTS.md` 里手写"最小改动、不要过度设计"，写得比代码还长 |

规则型 skill 的价值就在这：**把"别多余"变成可执行的分级规则**，而不是每次靠你在 prompt 里喊。

## 优点 / 缺点

| 优点 | 缺点 |
|---|---|
| 分 lite / full / ultra 三档，能从"少写点"一路收紧到"最懒方案" | 收得太紧时，后续确实要加的功能会被砍，得提前说清"这一步之外的需求" |
| 明确不许砍校验、错误处理、安全、无障碍 —— 不是"少写就行" | 探索型任务（"给我看看有几种方案"）会显得不耐烦 |
| 规则在提示词里，三家 agent 都能装 | 提示词约束，模型仍可能越界；配合项目 `AGENTS.md` 才稳 |
| 每条捷径要求写 `shortcut:` 注释，欠的债有据可查（还能用 `/ponytail-debt` 列账） | 团队要接受"先偷懒后升级"的节奏，不然 reviewer 会一直问"为什么不抽象" |

## 安装

```bash
npx skills add DietrichGebert/ponytail -g
```

Pi 也可以当包装（本机就是这么装的，见 [`../pi/README.md`](../pi/README.md)）：

```bash
pi install git:github.com/DietrichGebert/ponytail
pi list
```

## 三个 agent 怎么接

| Agent | 装到哪 | 调用 |
|---|---|---|
| Pi | `pi install git:...`（包）或 `~/.pi/agent/skills/ponytail/` | `/skill:ponytail`、`/ponytail ultra` 换档 |
| Claude Code | `~/.claude/skills/ponytail/` 或项目 `<仓库>/.claude/skills/ponytail/` | `/ponytail`、`/ponytail lite` |
| Codex | `~/.codex/skills/ponytail/`，或项目 `.agents/skills/ponytail/` | `$ponytail 简化这个实现` |

三家共用一个兜底办法：在项目 `AGENTS.md`（三家都读）里写一句硬规则，skill 只作为加档：

```markdown
改代码前先找现有实现，能用一行就不用三行；不新增依赖、不新增配置项；不许砍校验、错误处理和权限检查。
```

验证：装完让 agent 干一件小事（"给这个函数加个日志"），看它有没有顺手新建文件。差别很明显。

**我的看法**：如果你经常吐槽"这活儿我自己写十分钟就完了"，这个最值得先装。它和 [`caveman.md`](./caveman.md) 不冲突：caveman 管嘴上少说，ponytail 管手上少写。
