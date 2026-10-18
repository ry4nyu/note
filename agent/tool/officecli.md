# OfficeCLI：让 Agent 直接写 Word / Excel / PPT

**仓库**：<https://github.com/iOfficeAI/OfficeCLI>
**类型**：CLI + Skill

让 agent 直接操作 Word / Excel / PowerPoint。**单二进制、不需要装 Office、无依赖**。

它有两个关键能力：一是生成 `.docx` / `.xlsx` / `.pptx`；二是内置 HTML 渲染引擎，把文档渲染成 HTML 或 PNG —— 这等于给了 agent "眼睛"，能自己看渲染结果再改。

## 没有它之前

| 做法 | 代价 |
|---|---|
| 让 agent 写 `python-docx` / `openpyxl` 脚本 | 先建 venv、装库、查 API，样式来回调；跑完还得你在 Office 里打开才发现表格错位 |
| 让 agent 生成 Markdown / CSV，你自己粘进 Office | 格式、页边距、字体全重做一遍；一旦要改数据就再来一轮 |
| 用 `pandoc` 转 | 转换模板不可控，样式全丢 |
| 人肉在 Office 里做 | 数据在代码/数据库里，你要先把它导出来、再贴进去 —— 真正花时间的是这一步 |

核心差别不是"能不能生成文件"，而是**生成后 agent 能不能自己看结果**。以前只能靠你截图反馈，一轮轮改。

## 优点 / 缺点

| 优点 | 缺点 |
|---|---|
| 单二进制、无 Office 依赖，WSL 里直接跑 | 排版精细度不如真 Office + 手工调，交领导的正式物料最后仍要人过一眼 |
| 能把文档渲染成 PNG/HTML 自检，改了再改 | 复杂图表、母版、门页这类高级特性别指望它 |
| docx / xlsx / pptx 一套命令，不用学三个库 | 安装脚本会往多个 agent 的 Skill 目录里写东西，装之前看一眼它动了哪些文件 |
| 天然适合"数据在库里、要交个表格文档"这种活 | 生成的文件仍是二进制，`git diff` 看不了，评审只能靠人 |

## 安装

装法很省事，把这条丢给 agent 就行：

```text
curl -fsSL https://officecli.ai/SKILL.md
```

或者自己装二进制后跑：

```bash
officecli install   # 拷贝二进制到 PATH，并把 officecli skill 装进检测到的所有 agent
```

README 声称检测 Claude Code、Cursor、Windsurf、GitHub Copilot 等。

## 三个 agent 怎么接

| Agent | 方式 | 调用 |
|---|---|---|
| Pi | 官方检测列表里没写 Pi；把 skill 目录放到 `~/.pi/agent/skills/officecli/`，或干脆不装 skill | `/skill:officecli` |
| Claude Code | `officecli install` 自动装到 `~/.claude/skills/officecli/` | `/officecli` |
| Codex | 自动装到 `~/.codex/skills/officecli/`，或手动放项目 `.agents/skills/officecli/`（Pi 也读这里） | `$officecli` |

**没被自动检测到时也不用较劲**：OfficeCLI 就是个命令行工具，agent 有 shell 就能用。在 `AGENTS.md` 里写一句就够了，不需要 skill：

```markdown
需要生成 Word / Excel / PPT 时用 officecli（先跑 `officecli --help` 看当前版本的命令），
生成后渲染成 PNG 自己看一遍再交付。
```

这比"把 SKILL.md 装进四个目录"省事，也不会因为 agent 升级导致 skill 失效。

## 用起来

- "把 `reports/q3.csv` 做成带图表的 xlsx"
- "根据这份需求写一份 Word 设计文档，带目录和表格"
- "生成 8 页汇报 PPT，每页一个要点"
- "把这份 docx 渲染成 PNG 我看看版式"

**典型场景**：周报、需求文档、数据统计表、汇报 PPT —— 尤其是"数据在代码/数据库里，要交个表格文档"这种活。

## 踩坑清单

- **agent 说没这个命令**：二进制不在 PATH（新 shell 才生效），或者它跑在另一个环境里（Windows 侧装了、WSL 里没有）。
- **生成的文件打不开 / 样式乱**：先让它自己渲染看一遍，别等到你打开才反馈。
- **装完 `git status` 多出一堆文件**：`officecli install` 改了 agent 配置和 skill 目录，确认后再提交。
- **别把敏感数据丢进示例文档**：生成的 docx/xlsx 会落盘，存在 `tmp/` 之类没被 `.gitignore` 的目录就有外泄风险。

**适合谁**：经常要交 Word / Excel / PPT，而且数据本身在代码库里的人。纯手工做两份比写提示词快的时候，就不用它。
