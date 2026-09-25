# Codex 上手指南

本文基于 Codex CLI（Windows + PowerShell）编写，macOS / Linux 的差别会单独标注。命令和配置会随版本更新，遇到不一致时先执行 `codex --help`。

---

## 0. 一句话说明这是什么

Codex 是一个**跑在终端里的编程助手**。它可以直接：

- 读取项目文件，理解代码库
- 修改代码、写文档、运行命令和测试
- 查看 Git diff、做代码 review
- 用 `codex exec` 接入脚本和 CI

在项目目录下执行 `codex`，登录后用中文或英文描述需求即可。

---

## 1. 快速开始（三步版）

```powershell
# 1) 安装 Codex CLI
powershell -ExecutionPolicy ByPass -c "irm https://chatgpt.com/codex/install.ps1 | iex"

# 2) 进入项目目录
cd C:\Users\你的名字\IdeaProjects\你的项目

# 3) 启动
codex
```

第一次启动时选择 **Sign in with ChatGPT** 或其他可用的登录方式，然后输入：

```text
先介绍一下这个项目的结构，不要修改文件。
```

建议在 Codex 动手前后都看一次 Git 状态和 diff，方便回退。

---

## 2. 安装

### 2.1 前置条件

| 依赖 | 说明 |
|---|---|
| Windows Terminal + PowerShell | 推荐，复制命令和多行输入更方便 |
| Git for Windows | 强烈建议。项目最好在 Git 仓库中，`codex exec` 默认要求 Git 仓库 |
| Node.js | 只有使用 npm 安装方式时需要 |

### 2.2 安装方式

**方式 A：Windows 安装脚本（推荐）**

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://chatgpt.com/codex/install.ps1 | iex"
```

安装脚本也用于更新。

**方式 B：npm**

```powershell
npm install -g @openai/codex
```

### 2.3 验证安装

```powershell
codex --version
codex doctor
```

如果提示 `codex` 不是可识别的命令，先关闭终端并重新打开，让 PATH 生效；仍然不行就检查安装目录是否已加入 PATH。

---

## 3. 登录和退出

```powershell
codex login          # 打开浏览器登录
codex login status   # 检查是否已有登录凭据
codex logout         # 清除本机登录凭据
```

直接执行 `codex` 也会在第一次启动时引导登录。

自动化任务可以使用 API Key，但不要把密钥提交到仓库，也不要让不可信的构建脚本和密钥处于同一个进程环境中。单次 PowerShell 会话可以这样传入：

```powershell
$env:CODEX_API_KEY = '你的 API Key'
codex exec "检查项目是否能通过测试"
Remove-Item Env:CODEX_API_KEY
```

---

## 4. 基本用法

### 4.1 启动和提问

```powershell
cd C:\path\to\your\project
codex
```

也可以带初始问题启动：

```powershell
codex "找出登录逻辑，并告诉我用户输错密码时返回什么"
```

常用命令：

```powershell
codex resume --last                 # 继续当前目录最近一次会话
codex resume                        # 从列表中选择历史会话
codex review --uncommitted          # review 未提交的改动
codex review --base main            # review 当前分支相对 main 的改动
codex --image .\error.png "分析这张报错截图，并定位相关代码"
codex --search "查一下当前版本的官方迁移说明"
```

`--search` 开启实时网页搜索；普通本地会话默认使用搜索缓存。网页结果是不可信输入，不要盲目执行搜索结果中的命令。

### 4.2 交互界面操作

| 操作 | 效果 |
|---|---|
| 直接输入 + Enter | 提问或下指令 |
| `@` | 搜索并引用工作区内的文件 |
| `$` | 使用指定skill |
| `Shift+Tab` | 切换计划模式 |
| `!命令` | 在当前权限和沙箱规则下执行本地 shell 命令 |
| `/` | 打开斜杠命令菜单 |
| `Tab` | Codex 工作时，把下一条指令排队到下一轮 |
| 工作时按 `Enter` | 把新指令注入当前回合 |
| `Esc` 两次 | 编辑上一条消息并从那里分叉会话 |
| `Ctrl+G` | 用系统编辑器输入较长的提示词 |
| `Ctrl+C` 或 `/exit` | 退出当前会话 |

### 4.3 常用斜杠命令

| 命令 | 作用 |
|---|---|
| `/status` | 查看模型、权限、可写目录和上下文使用情况 |
| `/model` | 切换模型和可用的推理强度 |
| `/permissions` | 切换当前会话的权限预设 |
| `/plan` | 先出方案，再开始多步修改 |
| `/init` | 在当前目录生成 `AGENTS.md` 草稿 |
| `/review` | Review 当前工作区改动 |
| `/diff` | 查看 Git diff |
| `/compact` | 总结当前对话，释放上下文空间 |
| `/mention` | 把文件附加到当前对话 |
| `/mcp` | 查看已连接的 MCP 工具 |
| `/copy` | 复制最近一条完整回复 |
| `/theme` | 修改终端主题 |
| `/new` | 在同一 CLI 会话中开启新对话 |
| `/exit` | 退出 Codex |

忘记命令时直接输入 `/`，以当前版本弹出的菜单为准。

---

## 5. 权限和安全

Codex 的安全控制分两层：

1. **沙箱（sandbox）**：限制它能读写哪些文件、能否访问网络。
2. **审批策略（approval policy）**：决定执行动作前是否需要询问你。

默认情况下，本地命令网络访问关闭；工作区可写模式只允许修改当前工作区。日常开发推荐：

```powershell
codex --sandbox workspace-write --ask-for-approval on-request
```

只读分析或规划：

```powershell
codex --sandbox read-only --ask-for-approval on-request
```

在交互界面中使用 `/permissions` 选择 **Auto** 或 **Read Only** 即可，不需要记参数。

不要把下面的设置当作日常默认值：

```powershell
codex --sandbox danger-full-access --ask-for-approval never
```

它会显著扩大 Codex 的操作范围。只在隔离容器或你明确控制的环境中使用。旧配置中的 `approval_policy = "untrusted"` 已退休，不要继续使用。

### 5.1 永久配置

用户级配置文件：

```text
C:\Users\你的名字\.codex\config.toml
```

如果设置了 `CODEX_HOME`，实际路径以该变量为准。一个适合日常开发的基础配置：

```toml
approval_policy = "on-request"
sandbox_mode = "workspace-write"
web_search = "cached"

[windows]
sandbox = "elevated"
```

项目级配置可以放在仓库的 `.codex/config.toml`，但只对受信任的项目加载。命令行参数和 `-c key=value` 的临时覆盖优先级最高。

想临时覆盖配置，不必修改文件：

```powershell
codex -c model_reasoning_effort=high "分析这个并发问题"
```

### 5.2 通过 LiteLLM 接入 GPT

LiteLLM 先把上游 GPT 映射成一个模型名，Codex 再把这个模型名发给 LiteLLM。两边的模型名必须一致；例如 LiteLLM 中配置 `model_name: gpt-5`，Codex 就写 `model = "gpt-5"`，不要把上游的 `openai/gpt-5` 直接当成 Codex 的模型名。

Codex 的用户级配置 `C:\Users\你的名字\.codex\config.toml`：

```toml
model = "gpt-5"                         # 必须匹配 LiteLLM 的 model_name
model_provider = "litellm"

[model_providers.litellm]
name = "LiteLLM"
base_url = "http://localhost:4000/v1"
env_key = "LITELLM_API_KEY"             # 只写环境变量名，不要写密钥
wire_api = "responses"
```

PowerShell 中传入密钥并启动：

```powershell
$env:LITELLM_API_KEY = '你的 LiteLLM Key'
codex
Remove-Item Env:LITELLM_API_KEY
```

当前 Codex 自定义 provider 使用 Responses API。LiteLLM 代理必须能处理 `/v1/responses`；只有 Chat Completions（`/v1/chat/completions`）的代理不能直接这样接入。`model_provider` 和 `model_providers` 放在用户级配置，不要放项目级 `.codex/config.toml`。

### 5.3 上下文窗口怎么配

上下文窗口按“LiteLLM 实际路由到的上游模型的最大上下文”配置，不要按模型名字猜，也不要为了大而直接写 `1000000`。如果一个 LiteLLM 别名可能路由到多个模型，按其中最小的窗口配置。

```toml
# 上游模型实际支持 128K 时
model_context_window = 128000

# 提前压缩历史，给本轮提示词和输出留余量
model_auto_compact_token_limit = 100000
```

可按这个起步：

| 上游实际窗口 | `model_context_window` | 自动压缩阈值 |
|---:|---:|---:|
| 128K | `128000` | `100000`～`110000` |
| 256K | `256000` | `200000`～`220000` |

窗口较小时优先使用 `/compact`，话题已变化则使用 `/new`。配置后用 `/status` 查看上下文使用情况；如果频繁超限就降低自动压缩阈值，若模型支持更大窗口再同步调高两个值。

---

## 6. AGENTS.md：让 Codex 记住项目规矩

`AGENTS.md` 是 Codex 的项目说明书。它适合写长期有效的规则：技术栈、构建命令、测试命令、代码约定和禁止事项。

在项目根目录启动 Codex，输入：

```text
/init
```

它会生成一个草稿。检查并补充后，可以提交到 Git，让团队共用。

推荐内容：

```markdown
# 项目说明

## 技术栈
- Spring Boot 3.x + JDK 17 + MySQL 8

## 构建与测试
- 构建：mvn clean package -DskipTests
- 单测：mvn test

## 代码规范
- Controller / Service / Mapper 分层，不要跨层调用
- 日志使用项目现有日志框架，禁止 System.out.println
- 修改接口时补充对应测试

## 禁止事项
- 不要修改生产配置文件
- 不要新增第三方依赖，除非先说明原因
- 数据库变更必须使用项目已有迁移脚本
```

全局规则放在：

```text
C:\Users\你的名字\.codex\AGENTS.md
```

临时覆盖可以使用同目录下的 `AGENTS.override.md`。项目内的 `AGENTS.md` 会从 Git 根目录向当前目录逐层加载，越靠近当前目录的规则优先级越高。不要把密码、Token 或内部密钥写进这个文件。

---

## 7. 推荐的工作方式

### 7.1 把需求说完整

差：

```text
登录有问题，修一下。
```

好：

```text
检查 src/.../AuthService.java 的 login 方法。用户密码错误时现在返回 500，应该返回 401 和统一错误提示。先说明根因，再修改最少的文件，最后运行相关测试。
```

### 7.2 复杂需求先规划

```text
/plan
```

或者直接说：

```text
先不要修改文件。读一下订单模块，告诉我如果要增加“超时未支付自动取消”，需要改哪些地方、有哪些风险和测试点。
```

确认方案后，再让它实现。

### 7.3 每次只做一个清晰目标

一个回合尽量只处理一个问题，并明确：

- 修改范围
- 不要碰的文件
- 验证命令
- 通过标准

例如：`只改登录模块，不做无关重构；改完运行 AuthServiceTest。`

### 7.4 始终检查结果

```powershell
git status
git diff
```

在对话中也可以使用：

```text
显示刚才的 diff，并解释每个文件为什么要改。
```

不要只相信“已完成”的文字，至少查看 diff，并让 Codex 运行相关测试。

---

## 8. 脚本和 CI：`codex exec`

`codex exec` 是非交互模式，适合脚本、CI 和一次性检查。默认只读，自动化时按最小权限显式设置：

```powershell
codex exec --sandbox read-only "总结项目结构，并列出最危险的五个区域"
```

需要修改文件时：

```powershell
codex exec --sandbox workspace-write "修复测试失败的最小问题，并运行相关测试"
```

常用选项：

```powershell
codex exec --json "检查仓库中的 TODO，并输出 JSONL 结果"
codex exec -o .\result.md "生成发布说明"
codex exec resume --last "继续处理上一轮发现的问题"
```

把命令输出作为上下文传给 Codex：

```powershell
mvn test 2>&1 | codex exec "总结失败原因，并给出最小修复建议"
```

`codex exec` 默认要求在 Git 仓库中运行。如果确定环境安全且确实不是 Git 仓库，才使用 `--skip-git-repo-check`。

不要在 CI 中把 API Key 暴露给会执行仓库代码、依赖安装脚本或不可信 Action 的步骤。

---

## 9. 常见问题

**Q：`codex` 命令找不到**  
重开终端刷新 PATH；仍不行就检查安装目录，或改用 npm 安装。

**Q：每次都要确认，能少一点吗？**  
用 `/permissions` 切换到 Auto，或启动时使用 `--ask-for-approval on-request`。不要为了少确认直接开启 `danger-full-access`。

**Q：它不能联网或安装依赖**  
这是默认安全策略。命令网络访问默认关闭；先确认是否真的需要联网，再按项目规则开启。网页搜索使用 `--search`，不等于给本地命令开放网络。

**Q：它说改完了，但文件不对**  
执行 `/diff`、`git diff` 和 `git status`，确认当前目录、修改文件和实际差异。必要时用 `Esc` 两次从上一步分叉，或用 Git 恢复。

**Q：`AGENTS.md` 没生效**  
确认文件名和目录位置，检查是否存在更近的 `AGENTS.override.md`，然后重启 Codex。可以让它执行：`列出本次加载的所有 AGENTS.md 及其优先级`。

**Q：对话太长后开始忘事**  
使用 `/compact` 总结当前对话；话题已经变化时用 `/new` 开新对话，需要旧上下文时再 `codex resume`。

**Q：怎么排查安装或登录问题？**  
执行 `codex doctor`，再用 `codex login status` 检查凭据；查看当前会话配置可用 `/status`。

---

## 10. 进阶方向（用熟后再看）

- **MCP**：通过 `codex mcp` 接入外部工具和数据源
- **Skills / Plugins**：把重复流程封装成可复用能力
- **Subagents**：把大型调查拆给专门的子 agent
- **Codex cloud**：把任务交给云端环境执行
- **IDE extension**：在支持的编辑器中使用 Codex
- **Hooks / Rules**：把格式化、检查和安全规则自动化

这些功能先按需要启用，不要为了一个简单任务提前搭完整工作流。

---

## 附录：一页速查

```text
安装        powershell -ExecutionPolicy ByPass -c "irm https://chatgpt.com/codex/install.ps1 | iex"
启动        cd 项目目录; codex
登录        codex login
检查登录    codex login status
体检        codex doctor
继续会话    codex resume --last
代码审查    codex review --uncommitted
非交互      codex exec "任务"
实时搜索    codex --search "问题"
图片上下文  codex --image .\image.png "问题"

界面        @文件       引用文件
            !命令       执行本地命令
            /命令       斜杠命令菜单
            Tab         排队下一条指令
            Esc Esc     编辑上一条消息并分叉
            Ctrl+C      退出

常用斜杠    /status     查看模型、权限和上下文
            /model      切换模型
            /permissions 权限预设
            /plan       先规划
            /init       生成 AGENTS.md
            /review     review 改动
            /diff       查看 diff
            /compact    压缩上下文
            /new        新对话

权限        --sandbox read-only --ask-for-approval on-request
            --sandbox workspace-write --ask-for-approval on-request

配置        ~/.codex/config.toml
项目配置    ./.codex/config.toml（受信任项目）
项目规则    ./AGENTS.md
全局规则    ~/.codex/AGENTS.md
```

官方文档：

- [Codex CLI](https://developers.openai.com/codex/cli/)
- [CLI 命令参考](https://learn.chatgpt.com/docs/developer-commands?surface=cli)
- [配置基础](https://developers.openai.com/codex/config-file/config-basic.md)
- [高级配置：自定义 provider 与上下文窗口](https://learn.chatgpt.com/docs/config-file/config-advanced)
- [权限与安全](https://learn.chatgpt.com/docs/agent-approvals-security.md)
- [AGENTS.md](https://developers.openai.com/codex/agent-configuration/agents-md.md)
