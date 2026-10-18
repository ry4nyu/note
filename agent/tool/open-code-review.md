# open-code-review：让 Agent 评审代码

**仓库**：<https://github.com/alibaba/open-code-review>
**官网 / 文档**：<https://open-codereview.ai>
**类型**：CLI（另有各 agent 的插件 / skill 包、MCP Server）

前面几篇是"让 agent 看懂代码 / 少说废话"，这个是**让 agent 评审代码**。它原本是阿里内部的官方 AI 代码评审助手，跑了两年多、服务数万名开发者后开源。

工作方式：读 git diff → 挑出要审的文件 → 交给带 tool-use 的可配置 LLM（agent 可以自己翻完整文件、搜代码库、看其他改动文件）→ 输出**行级精确**的结构化评审意见。

## 没有它之前

三条路，都不太够用：

| 以前怎么做 | 问题 |
|---|---|
| 人肉 CR，两个人对着 diff 看 | 慢，而且看得动的时候往往已经合并了 |
| 甩给 agent 一句"review 这个 PR" | 见下：挑文件审、行号飘、prompt 改一个字结果就变 |
| 自己写一个 review skill / slash command | 上面那些毛病一个不少，只是换了个触发方式 |
| 接 SonarQube 这类静态扫描 | 抓得住 bug 模式，抓不住"这段逻辑和需求不符" |

README 点名了纯自然语言方案的三个毛病：

- **大改动偷懒**：changeset 一大，agent 就开始挑几个文件审完交差。
- **定位漂移**：报的问题对不上代码位置，行号乱飞。
- **质量随 prompt 波动**：自然语言规则难以调试，prompt 改一个字结果就变。

根因是纯语言驱动的架构对评审流程**没有硬约束**。

## 它的解法：确定性工程 × agent 混合

该由代码保证的，不交给模型：

- **精确选文件**：哪些文件该审、哪些该过滤，由工程逻辑决定，避免漏审。
- **智能打包**：相关文件合成一个评审单元（比如 `message_en.properties` 和 `message_zh.properties` 捆在一起），每个包跑一个独立上下文的 sub-agent。大改动下天然稳定，也支持并发。
- **细粒度规则匹配**：按文件特征用模板引擎匹配评审规则，比"在 prompt 里写一堆规则"更稳、更可预测。
- **独立的定位与反思模块**：专门治上面那三个毛病里的"行号飘"和"内容飘"。

交给模型的只有它擅长的部分：动态上下文检索，配专用的 prompt 模板和从生产调用链数据里蒸馏出来的专用工具集。

## 优点 / 缺点

| 优点 | 缺点 |
|---|---|
| 评审流程有硬约束：选文件、打包、规则匹配都是工程逻辑，不随 prompt 波动 | **Recall 刻意更低**：宁可漏报，不给你一堆要逐条 triage 的噪音。想"尽量别漏"这条就得认 |
| 行级精确意见，能直接对着改 | 多一层自己的模型配置（provider / model / key），和 agent 的模型配置互不相干 |
| Delegation 模式不用额外买模型额度，让正在用的 agent 跑 | 要 Git >= 2.41；再来一个全局 npm 包和一份独立 config |
| 能上 CI 做门禁，结果可落结构化 JSON | 评审质量仍取决于你给它配的那个模型；换模型要重新校准 |
| 大改动下不偷懒（这本来是最常见的失效模式） | 报出来的都对、但漏掉的也可能是真问题 —— 人工 CR 仍然不能省 |

## 两种跑法，先选一个

这是我第一遍看文档时最容易漏掉的地方 —— 评审用的模型**不一定是 `ocr` 自己的**：

| 模式 | 谁出模型 | 需要配 key 吗 |
|---|---|---|
| 默认（OCR 托管） | `ocr` 按你配的 provider / model 跑 | 要 |
| **Delegation 模式** | **你自己的 coding agent 用它的模型跑** | **不用** |

Delegation 模式下 `ocr` 只负责"选哪些文件、用哪些规则"，评审动作由你正在用的 agent（Claude Code / Codex / Cursor …）完成：

```bash
ocr delegate preview                 # 看这一轮会审哪些文件、命中哪些规则
ocr delegate rule src/main.go src/handler.go
```

不想为评审单独再买一份模型额度的话，直接走这条路。

## 安装

**前置：Git >= 2.41**（diff 生成、代码搜索、仓库操作都依赖它）。

```bash
npm install -g @alibaba-group/open-code-review
ocr --version
```

Windows 原生环境同样支持，PowerShell 里 `npm i -g` 即可；其他装法（install 脚本、Release 二进制、从源码编译）见[官方安装文档](https://open-codereview.ai/docs/installation)。

只走 Delegation 模式的话，可以跳过下面的模型配置。

```bash
ocr config provider                  # 选内置 provider 或自己加一个
ocr config model                     # 给当前 provider 挑模型
```

交互式引导会依次问 provider、API Key、模型，最后自动测一次连通性。注意这份配置是**独立的**，跟你 Claude Code / Pi 的模型配置互不影响。

配置存在 `~/.opencodereview/config.json`，所有交互式步骤都能用 `ocr config set <key> <value>` 一行搞定：

```bash
ocr config set provider <名字>
ocr config set providers.<名字>.model <模型名>        # 内置 provider
ocr config set custom_providers.<名字>.url <地址>     # 自建 provider
```

### 例：DeepSeek 走 LiteLLM 网关

`litellm` 是内置 provider（协议 OpenAI Chat Completions，默认地址 `http://localhost:4000/v1`，环境变量 `LITELLM_API_KEY`），不用自己拼：

```bash
ocr config set provider litellm
ocr config set providers.litellm.url http://localhost:4000/v1   # 网关不是默认地址才需要
ocr config set providers.litellm.api_key sk-xxxx                # 网关没开鉴权可跳过
ocr config set providers.litellm.model deepseek/deepseek-chat   # 填 LiteLLM 的 model_name
ocr llm test                                                    # 发一句探针验证连通
```

两个坑：

- **模型名填 LiteLLM 的 `model_name`，不是 DeepSeek 官方模型名。** 内置列表里的示例是 `deepseek/deepseek-chat`，但以你 `config.yaml` 里起的别名（比如 `deepseek-v3`）为准。`ocr config set` 不校验模型名，写错了要等到真跑评审才报错 —— 所以先 `ocr llm test`。
- **`ocr config set provider` 会清掉已选的 model**，所以顺序是 provider → url / api_key → model。`ocr config model` 那个交互菜单只列内置列表，别名不在里面，用自己的别名必须走 `ocr config set`；或者用 `ocr config provider` 里的 **Manual** 页手填 URL / protocol / model。

网关不是标准 OpenAI 协议、或者你想换个名字记，就自建 provider（`protocol` 必填，可选 `anthropic` / `openai` / `openai-responses` / `anthropic-bedrock`）：

```bash
ocr config set provider my-gw
ocr config set custom_providers.my-gw.protocol openai
ocr config set custom_providers.my-gw.url http://localhost:4000/v1
ocr config set custom_providers.my-gw.api_key sk-xxxx
ocr config set custom_providers.my-gw.model deepseek-chat
```

**不经过 LiteLLM 的话省事得多** —— DeepSeek 本身就是内置 provider：

```bash
ocr config set provider deepseek
ocr config set providers.deepseek.api_key sk-xxxx    # 或 export DEEPSEEK_API_KEY=...
ocr config set providers.deepseek.model deepseek-v4-pro
```

## 常用命令

```bash
# 工作区模式：暂存 + 未暂存 + 未跟踪的改动全审
ocr review

# 分支范围：从 main 分叉点（merge-base）开始算 feature-branch 的改动
ocr review --from main --to feature-branch

# 单个 commit
ocr review --commit abc123

# 全文件扫描：不看 diff，直接审整份文件（适合接手不熟悉的仓库，不需要 git 历史）
ocr scan
ocr scan --path internal/agent

# 中断了可以续跑
ocr session list
ocr review --from main --to feature-branch --resume <session-id>
ocr scan --resume <session-id>

# 结果落盘，给上层 agent 或 CI 用
ocr review --format json --output result.json
```

`ocr review` 不带参数就是"提交前自己跑一遍"，这是个人用最划算的场景。

## 接进现有工作流

### 三个 agent 怎么接

| Agent | 官方集成 | 调用 |
|---|---|---|
| Pi | 没有 Pi 专用插件；官方列了"其他支持 skill 的 agent 可以装 portable skill" | `/skill:<名字>`（portable skill），或直接让它跑 CLI |
| Claude Code | slash command 插件 | 官方文档里给的命令名 |
| Codex | 可调用 skill | `$<名字>` |
| Cursor / Kimi Code / OpenCode | 均有现成集成 | — |

插件名以 [官方文档](https://open-codereview.ai) 为准 —— 我没在 WSL 里逐个装过，不照抄命令名免得你踩空。

**零配置兜底（三家通用，最省事）**：不装任何插件，agent 只需要 shell 权限。在 `AGENTS.md` / `CLAUDE.md` 里写一句：

```markdown
评审当前改动时跑 `ocr review --format json --output /tmp/ocr.json`，按输出的文件和行号改；
只想看会审哪些文件时用 `ocr delegate preview`。
```

然后是这句人话：

```text
跑一遍 ocr review，把问题按严重程度排序，只改 blocker 和 major。
```

这样三家行为一致、能进 CI、也不依赖插件版本。

### 其它

- **CI 门禁**：GitHub Actions、GitLab CI、Gerrit、GitFlic CI 均有官方集成。
- **看结果**：Session Viewer 可以在浏览器里回放整轮评审，把意见标记成"已修 / 忽略"，边改边隐藏。
- **扩工具**：自带 MCP Server，可以给评审 agent 外挂更多工具。

## Benchmark 怎么看

官方放了 AACR-Bench：**50** 个热门开源仓库、**200** 个真实 PR、**10** 种语言，80+ 资深工程师标注出 **1,505** 个真实问题（数据集在 Hugging Face 上）。

同模型下对比通用 agent（Claude Code）：

- Precision / F1 明显更高，token 只花约 **1/9**，更快。
- **Recall 反而更低** —— 这是**刻意的取舍**：宁可漏报，少给你一堆噪音让你逐条 triage。

这两句话要一起读。如果你要的是"尽量别漏"，通用 agent 扫一遍可能更合适；要的是"报出来的都能直接改"，这个更合适。

> 以上数字均为项目 README 自述，我没有本地复现。

## 适合谁

- 团队评审想加一层机器兜底，或者要塞进 CI 做门禁 —— 这是它的主场。
- 个人：`ocr review` 提交前自检；不想额外配模型就走 Delegation。
- 不适合：只想让 agent 看完改动后随口说两句 —— 现有的 review skill 就够了，没必要上这套工程。

**我的看法**：它的价值在"确定性"那一半，不在模型。想要的是一个不会因为你 prompt 写歪就偷懒的评审流程 —— 这正是纯 skill 方案给不了的。
