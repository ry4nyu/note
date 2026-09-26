# Pi 上手指南

本文基于 Pi 0.87.x（Windows + PowerShell 环境实测），macOS / Linux 的差别会单独标注。命令和默认值随版本变化，遇到不一致先执行 `pi --help`。

Pi 是 Earendil Works 做的终端编程 Agent。和 Claude Code、Codex 相比，它的取舍是"最小内核 + 一切靠扩展"：内置工具只有 `read` / `bash` / `edit` / `write` / `grep` / `find` / `ls`，其余能力（规划模式、代码图谱、技能）都通过扩展、包和 Skill 挂上去。

---

## 0. 一句话说明这是什么

Pi 是一个**跑在终端里的编程助手**。它可以直接：

- 读项目文件、搜索代码、理解目录结构
- 改代码、写文档、跑命令和测试
- 管理会话（继续、分叉、回退到某个节点、压缩、导出）
- 用 `pi -p` / `--mode json` / RPC 接入脚本和 CI，或用 TypeScript SDK 内嵌

在项目目录下敲 `pi`，选好模型后就用中文/英文描述需求。

---

## 1. 快速开始（三步版）

```powershell
# 1) 安装（需要 Node.js 22.19+）
npm install -g --ignore-scripts @earendil-works/pi-coding-agent

# 2) 配好模型接入（见第 5 节，二选一：/login 官方账号 或 自建网关）

# 3) 进项目目录，启动
cd C:\Users\你的名字\IdeaProjects\你的项目
pi
```

启动时如果目录里有需要信任的资源（比如 `.pi/` 下的配置、扩展，或 `.agents/skills/`），Pi 会先问你一次是否**信任这个工作目录**（见 5.2），确认后就能直接打字提问了。

> 本机已经配好公司网关（`~/.pi/agent/models.json`），直接 `pi` 就能用，不用再 `/login`。

---

## 2. 安装

### 2.1 前置条件

| 依赖 | 说明 |
|---|---|
| Node.js | **必须 22.19+**，Pi 只有 npm 一种 Windows 安装方式 |
| Git for Windows | **强烈建议**。Pi 的 `bash` 工具和 `!` 命令默认走 Git Bash |
| Windows 版本 | Windows 10 / 11 均可，原生运行或跑在 WSL 里都行 |
| 终端 | 推荐 Windows Terminal + PowerShell 7 |

```powershell
node --version     # 必须 >= v22.19.0
```

### 2.2 安装 Pi

```powershell
npm install -g --ignore-scripts @earendil-works/pi-coding-agent
```

> `--ignore-scripts` 是官方推荐的写法，Pi 正常安装**不需要**依赖的生命周期脚本。公司如果限制 npm 脚本执行，这个参数正好符合要求。

**macOS / Linux / WSL** 可以用官方脚本：

```bash
curl -fsSL https://pi.dev/install.sh | sh
```

### 2.3 验证安装

```powershell
pi --version     # 打印版本号，例如 0.87.1
pi --help        # 看当前版本支持的完整参数
```

> 如果提示 `pi` 不是可识别的命令：关掉终端重开（PATH 需要刷新），或检查 npm 全局 bin 目录（`npm config get prefix`）是否在 PATH 里。

---

## 3. 基本用法

### 3.1 启动

```powershell
cd C:\path\to\your\project     # 必须先切目录，Pi 按它发现文件、AGENTS.md 和配置，并按它给会话分组
pi                             # 进入交互界面
```

常用启动方式：

```powershell
pi "先介绍一下这个项目结构，不要改文件"   # 带初始问题启动
pi --continue                             # 继续当前目录上一次会话（-c）
pi --resume                               # 从历史会话列表里挑一个（-r）
pi -p "解释一下 pom.xml"                  # 非交互模式，跑完就退出，适合脚本
pi -p --mode json "列出所有 TODO" > out.jsonl   # 结构化输出，适合喂给别的程序
pi --name "重构登录模块"                  # 给本次会话起个名字，方便以后 /resume 找到
pi --model sonnet:high "帮我分析这个并发问题"    # 指定模型 + 思考等级
pi --tools read,grep,find,ls -p "评审 src/ 下的代码"   # 只读模式，物理上改不了文件
pi update                                 # 升级 Pi 和模型目录
```

`pi --help` 里还有几组值得知道的开关：

| 参数 | 作用 |
|---|---|
| `-p, --print` | 非交互：处理完就退出 |
| `--mode text\|json\|rpc` | 输出模式，`rpc` 可以用程序双向驱动 |
| `-c` / `-r` / `--session <id>` / `--fork <id>` | 继续 / 选会话 / 指定会话 / 从某会话分叉 |
| `--no-session` | 本次不存会话（临时用） |
| `--tools` / `--exclude-tools` / `--no-tools` | 工具白名单 / 黑名单 / 全关 |
| `--thinking off\|minimal\|low\|medium\|high\|xhigh\|max` | 启动时的思考等级 |
| `--models "anthropic/*,deepseek-*"` | 限定 Ctrl+P 循环的模型范围 |
| `-a, --approve` / `-na, --no-approve` | 本次运行信任 / 不信任项目本地资源 |
| `-e <path>` / `--skill <path>` | 临时加载扩展 / Skill |
| `--tui-mode fullscreen` | 全屏模式（编辑器和状态栏固定） |
| `--offline` | 关掉启动时的联网动作 |

### 3.2 交互界面操作

界面分三块：上方是对话记录（transcript），中间是输入编辑器，底部状态栏显示当前目录、会话、模型、上下文用量和累计费用。**编辑器边框的颜色/样式表示当前思考等级**。

| 操作 | 效果 |
|---|---|
| 直接输入 + `Enter` | 提交提问或指令 |
| `Shift+Enter` 或 `Ctrl+J` | 输入内换行（不发送） |
| `Ctrl+G` | 用外部编辑器（Windows 默认记事本）写长提示词 |
| `@文件名` | 搜索并引用文件，`Tab` 补全路径 |
| 粘贴 / 拖入图片 | 直接进上下文（Windows 上是 `Alt+V`） |
| `!命令` | 执行 shell 命令，**输出进上下文**（如 `!git status`） |
| `!!命令` | 执行 shell 命令，**输出不进上下文**（只看结果，比如 `!!npm test`） |
| `/命令` | 斜杠命令，输入 `/` 会弹出可搜索的菜单 |
| `Esc` | 打断它正在做的事 |
| `Ctrl+O` | 展开 / 折叠工具调用输出 |
| `Ctrl+T` | 显示 / 隐藏 thinking 块 |
| `Ctrl+L` | 打开模型选择器 |
| `Ctrl+P` | 循环切换模型（Windows 上反向是 `Alt+P`） |
| `Shift+Tab` | 循环切换思考等级（**注意：不是权限模式**） |
| `Ctrl+X` | 复制最后一条回复 |
| `Ctrl+C` | 第一次清空输入框，第二次退出 |
| `Ctrl+D` | 退出（输入框为空时） |

**任务执行中还能再说话**，这点比"等它跑完"实用：

| 想要的效果 | 操作 |
|---|---|
| 调整当前任务方向 | 输入内容按 `Enter`（等当前工具调用跑完就生效） |
| 追加后续任务 | Windows 上按 `Ctrl+Q`（其它平台 `Alt+Enter`） |
| 把排队的消息收回输入框 | Windows 上按 `Alt+Q`（其它平台 `Alt+Up`） |
| 中止当前任务 | `Esc` |

> ⚠️ **Windows Terminal 会占用部分 Alt 快捷键**，所以 Pi 在 Windows 上给这几个动作换成了 `Ctrl+Q` / `Alt+Q` / `Alt+P`。想用 `Alt+Enter` 这类组合，需要按官方文档调整 Windows Terminal 的按键映射。

---

## 4. 常用斜杠命令

输入 `/` 打开可搜索菜单，**菜单里显示的就是当前会话实际加载的全部命令**（扩展、Skill、提示词模板都会往里加）。

| 命令 | 作用 |
|---|---|
| `/model [provider/model]` | 选择模型，`Ctrl+S` 存为默认 |
| `/thinking [级别]` | 设置思考等级，`Ctrl+S` 存为启动默认 |
| `/scoped-models` | 配置 Ctrl+P 循环包含哪些模型 |
| `/login` / `/logout` | 添加 / 移除 provider 凭据 |
| `/new` | 开新会话 |
| `/resume` | 切到别的历史会话 |
| `/name [名字]` | 给当前会话命名 |
| `/session` | 看当前会话的文件、ID、消息数、token 用量和费用 |
| `/tree` | 会话树导航（回看、跳到某个节点） |
| `/fork` | 从某条历史消息另起一个会话 |
| `/clone` | 把当前会话复制一份 |
| `/compact [额外要求]` | 压缩上下文，可带自定义总结要求 |
| `/export [路径]` | 导出成 HTML 或 JSONL |
| `/share` | 上传会话并返回查看链接（导出前先自己看一遍内容） |
| `/trust` | 保存对当前项目的信任决定，后续进程不再问 |
| `/reload` | 重新加载按键、扩展、Skill、模板、主题和上下文文件 |
| `/hotkeys` | 查看当前会话实际生效的快捷键 |
| `/settings` | 打开设置界面 |
| `/changelog` | 看更新日志 |
| `/debug` | 把终端渲染结果和会话消息写到 agent 目录下的 `pi-debug.log` |
| `/quit` | 退出 |

---

## 5. 配置

### 5.1 模型接入（二选一）

**方式 A：官方账号 / API Key**

在 Pi 里执行 `/login`，选 provider，按提示走订阅登录或存 API Key。凭据存在 `~/.pi/agent/auth.json`。

不想让 Pi 落盘凭据（CI 场景），就改用环境变量——各 provider 对应的变量名见官方 Provider Authentication 文档（比如 `ANTHROPIC_API_KEY`、`OPENAI_API_KEY`）。

**方式 B：公司自建网关 / 任何兼容端点**

Pi 不认识"公司网关"这个概念，但只要求端点说一种它支持的 API。写进 `C:\Users\你的名字\.pi\agent\models.json`：

```json
{
  "providers": {
    "litellm": {
      "baseUrl": "https://网关地址，找对接人拿/v1",
      "api": "openai-completions",
      "apiKey": "$HS_LLMLITE_API_KEY",
      "models": [
        {
          "id": "deepseek-flash",
          "name": "DeepSeek V4.1 Flash",
          "reasoning": true,
          "input": ["text", "image"],
          "contextWindow": 1000000,
          "maxTokens": 393216,
          "thinkingLevelMap": { "low": "low", "high": "high", "max": "max" },
          "compat": {
            "supportsStore": false,
            "supportsDeveloperRole": false,
            "maxTokensField": "max_tokens",
            "thinkingFormat": "deepseek"
          }
        }
      ]
    }
  }
}
```

几个关键点：

- **`apiKey` 只写变量名**：支持 `$NAME` / `${NAME}` 环境插值、字面值，或 `!命令`（请求时才执行）。**不要把令牌写死在文件里**。
- **`api` 决定按哪种协议发请求**：网关常见的是 `openai-completions`。写错了会报端点拒绝请求。
- **`models` 是模型清单**：`id` 要跟网关暴露的模型名完全一致，`contextWindow` / `maxTokens` 决定 Pi 认为的窗口大小（见 5.4）。
- **`input`** 声明模型能不能吃图片；**`reasoning`** 声明是不是推理模型；**`thinkingLevelMap`** 把 Pi 的思考等级映射到网关支持的档位。
- **`compat`** 用于描述端点真实存在的差异，不要因为对方"声称兼容 OpenAI"就乱加。
- **改完不用重启**：打开 `/model` 就会重新读这个文件。

本机的实际配置就是这一套，可以直接打开 `~/.pi/agent/models.json` 抄结构。

**凭据优先级**（从高到低）：运行时 `--api-key` → `auth.json` 里存的 → `models.json` 里的 `apiKey` → provider 的环境变量。排查用：

```powershell
pi auth check --provider litellm --json     # 看这个 provider 是否就绪
pi auth print-api-key --provider openai     # 给外部客户端取 key
```

> ⚠️ `auth.json`、`models.json` 都**不要提交到 git**。用环境变量插值就是为了让密钥留在文件之外。

### 5.2 权限与项目信任（和另外两个工具不一样）

**先说清楚：Pi 没有 Claude Code 那种 `auto` 权限模式，也没有 Codex 那种「逐次审批」。它默认不问你** —— 你看到的是每一次工具调用和它的结果（`Ctrl+O` 展开），而不是每个操作前的弹窗。

Pi 的立场是：审查记录**不构成安全边界**，真正的边界只有两个。

**闸门一：项目信任（project trust）**

它管的是"这个目录里的东西要不要**执行**"。当工作目录里存在下面这些资源时，Pi 必须先拿到信任决定：

- `.pi/settings.json`
- `.pi/extensions`、`.pi/skills`、`.pi/prompts`、`.pi/themes`
- `.pi/SYSTEM.md` 或 `.pi/APPEND_SYSTEM.md`
- 当前目录或祖先目录里的项目级 `.agents/skills`

决定按这个顺序产生：命令行 `--approve` / `--no-approve` → 用户级/命令行扩展处理 `project_trust` 事件 → 已保存的决定（`~/.pi/agent/trust.json`，就近生效）→ 全局设置 `defaultProjectTrust`（默认 `"ask"`）。

```powershell
pi --no-approve        # 本次完全不加载项目本地资源
pi --approve           # 本次直接信任
```

平时在交互界面里用 `/trust` 保存决定就行，一次之后不再问。

> ⚠️ **上下文文件不受这个闸门保护**：`AGENTS.md`、`AGENTS.override.md`、`CLAUDE.md` 不管项目可不可信都会被加载。也就是说，"只读看一下别人的仓库"时，仓库里的 `AGENTS.md` 已经进模型上下文了 —— 把它当**不可信输入**看待。真要干净，用 `--no-context-files`。

**闸门二：工具白名单**

这是 Pi 唯一"削弱能力"的硬开关，比审批模式实在得多：

```powershell
pi --tools read,grep,find,ls -p "评审这个模块"    # 只读，物理上改不了文件
pi --exclude-tools bash,write                     # 只禁掉危险的
pi -nt                                            # 工具全关（扩展工具也关）
pi -nbt                                           # 只关内置工具
```

要长期生效就写进 `~/.pi/agent/settings.json` 的 `defaultTools`：

```json
{
  "defaultTools": ["read", "powershell", "edit", "write", "grep", "find", "ls"]
}
```

**真正的隔离：容器或虚拟机**

Pi 会用它所属操作系统用户的权限读写和执行文件，**没有内置沙箱**。要跑"不可信或无人值守"的任务，唯一靠谱的办法是把整个 Pi 放进容器 / 虚拟机 / 沙箱，只暴露这次任务需要的目录、凭据和网络。

**几条硬要求：**

1. **不要在公司主力机器上直接跑未审阅的 `.pi/extensions/`** —— 扩展在 Pi 进程内执行，权限和 Pi 完全一样。
2. **能只给只读就给只读**：评审、调研类任务一律 `--tools read,grep,find,ls`。
3. **凭据给最小范围、最短有效期**，能不给就不给。
4. **改动前后都看 git**：`git status` / `git diff` 是你唯一的回退手段（Pi 的 `/tree`、`/fork` 管的是对话，不是文件）。

### 5.3 让 Pi 用 PowerShell（Windows）

Windows 上默认是这样分工的：

| 场景 | 默认走什么 |
|---|---|
| 模型调用的 `bash` 工具 | Git Bash |
| 你在输入框敲的 `!` / `!!` 命令 | Bash（**换不掉**） |
| 可选的 `powershell` 工具 | 只要在 `defaultTools` 里启用就会用 |

**启用 `powershell` 工具**，编辑 `~/.pi/agent/settings.json`：

```json
{
  "defaultTools": ["read", "powershell", "edit", "write", "grep", "find", "ls"]
}
```

本机已经这么配了。启用后 Pi 会优先用 `pwsh.exe`，找不到再回退到系统自带的 Windows PowerShell，启动参数是 `-NoProfile -NonInteractive -ExecutionPolicy Bypass`（公司如果强制了执行策略，管理员策略仍然优先）。

要**彻底换成** PowerShell（而不是两个并存），把数组里的 `bash` 去掉即可。这个工具只在原生 Windows 下可用；WSL 里没有。

验证一下：

```text
!printf 'Bash is working\n'          # 检查 Git Bash
!$PSVersionTable.PSVersion           # 在 PowerShell 工具里看版本（5.1 是系统自带，7.x 是 pwsh）
```

Bash 装在 Pi 找不到的地方（Cygwin / MSYS2）时，指定路径：

```json
{
  "shellPath": "C:\\cygwin64\\bin\\bash.exe"
}
```

> ⚠️ JSON 里 Windows 路径的每个反斜杠要写两个。`shellPath` 支持开头的 `~`。

### 5.4 上下文窗口与自动压缩

Pi 判断"这个模型窗口多大"，**依据的是模型元数据**，不是猜的：

- 内置目录里的模型 → 用目录里的值（`pi update --models` 可以刷新目录）
- `models.json` 里的自定义模型 → 用你写的 `contextWindow`
- **没写 `contextWindow`** → 退回一个偏保守的默认值

所以"聊几句就自动压缩"这类问题，根因通常是**自定义模型没声明窗口**：网关那边支持 1M，Pi 却按小窗口算。本机 `models.json` 里每个模型都写了 `contextWindow: 1000000`，就是为了避免这个问题。

自动压缩默认开启，相关设置：

```json
{
  "compaction": {
    "enabled": true,
    "reserveTokens": 16384,
    "keepRecentTokens": 20000
  }
}
```

| 设置 | 默认 | 作用 |
|---|---|---|
| `compaction.enabled` | `true` | 是否自动压缩 |
| `compaction.reserveTokens` | `16384` | 给模型本轮回复预留的 token |
| `compaction.keepRecentTokens` | `20000` | 最近多少 token 原样保留、不做摘要 |
| `compaction.modelOverrides` | 无 | 按 `provider/modelId` 单独配 |

手动操作：`/compact` 压缩、`/new` 换话题、`/session` 看用量和费用。窗口够大时不建议关自动压缩，但要排查问题可以用 `compaction.enabled: false`。

另外 `cacheWarming`（默认 `"streaming"`）会在模型声明了缓存寿命、且预估省下的缓存未命中费用超过 $0.05 时保持 prompt 缓存热；避免缓存失效的收益体现在费用上。`/session` 会显示下一次的决策。

---

## 6. AGENTS.md：让 Pi 记住项目规矩

Pi 会在**工作目录及其所有父目录**里找上下文文件，只要 Pi 运行在该目录或其子目录下就生效。可用文件名：`AGENTS.override.md`、`AGENTS.md`、`AGENTS.MD`、**`CLAUDE.md`**、`CLAUDE.MD`（所以从 Claude Code 迁过来可以直接复用 `CLAUDE.md`）。同目录下 `AGENTS.override.md` 会顶掉同名的 `AGENTS.md` / `CLAUDE.md`。

用户级规则放在 `~/.pi/agent/AGENTS.md`，对所有工作目录生效。

在项目根目录建一个 `AGENTS.md`，写清这些，它会少犯很多错：

```markdown
# 项目说明

## 技术栈
Spring Boot 3.x + MyBatis-Plus + MySQL 8 + Redis，JDK 17

## 构建与测试
- 构建：mvn clean package -DskipTests
- 单测：mvn test -Dtest=XxxTest
- 启动本地环境：见 README

## 代码规范
- 分层：Controller / Service / Mapper，不要跨层调用
- 统一返回体 Result<T>，不要直接返回实体
- 日志用 @Slf4j，禁止 System.out.println
- 新增接口必须补 Swagger 注解

## 禁止事项
- 不要动 src/main/resources/application-prod.yml
- 不要引入新的第三方依赖，先问我
- 数据库变更一律走 Flyway 脚本，不要手写 DDL
```

临时关掉上下文加载：`pi --no-context-files`（排查"到底是哪条规则在起作用"时很好用）。改完文件后 `/reload` 让它生效。

> 启动头部会列出本次加载了哪些指令和资源 —— 想知道到底读进去了什么，看那里最准。
>
> ⚠️ **不要把密码、Token 或内部密钥写进这个文件**，它会被完整发给模型。

---

## 7. 把它用好的几个习惯

1. **说清楚"在哪、做什么、期望什么"**
   - 差：`登录有问题，修一下`
   - 好：`src/main/java/.../AuthService.java 的 login 方法，用户输错密码时返回的是 500，应该是 401 加错误提示，帮我改`

2. **复杂需求先让它只读调研**

   ```powershell
   pi --tools read,grep,find,ls "读一下 order 模块，告诉我如果要加「超时未支付自动取消」该改哪些地方"
   ```

   也可以加载 plan-mode 扩展，用 `pi --plan` 直接进只读规划模式。

3. **一次只做一件事**，别一口气丢十个需求

4. **用 git 兜底**：让它动手前确认工作区干净，搞砸了 `git checkout .` 恢复。Pi 的 `/tree`、`/fork`、`/clone` 管的是对话分支，**不回滚文件**

5. **让它自己验证**：`改完跑一下相关单测` / `启动服务确认接口能通`

6. **聊长了 `/compact`，换话题 `/new`**；重要的会话先 `/name`，以后好找

7. **问它"你是什么模型"没有任何意义** —— 直接看环境变量才是准的。Shell 工具里注入了 `PI_PROVIDER` / `PI_MODEL` / `PI_REASONING_LEVEL` / `PI_SESSION_ID`：

   ```powershell
   !printf '%s/%s\n' "$PI_PROVIDER" "$PI_MODEL"
   ```

   （这些变量只在模型调用的 `bash` / `powershell` 工具里注入，你自己敲的 `!` 命令拿不到。）

8. **用完记得清会话**：一次性的小问题加 `--no-session`，别把会话目录堆满。

---

## 8. 常见问题

**Q：`pi` 命令找不到**
关掉终端重开刷新 PATH。还不行检查 `npm config get prefix` 对应的目录在不在 PATH 里。

**Q：安装/启动报 Node 版本错误**
Pi 要求 Node.js 22.19+，`node --version` 确认一下。升级 Node 后重开终端。

**Q：怎么接公司网关？**
见 5.1 方式 B。要点就三条：`baseUrl` 指网关、`api` 写对协议、`apiKey` 只写环境变量名。写完之后 `/model` 里能不能看到模型，取决于凭据能否解析——看不到多半是环境变量没在当前进程里。

**Q：它为什么不问我"是否允许"就直接改文件了？**
这是设计如此（见 5.2）。要限制能力就用 `--tools` / `defaultTools`，或者把 Pi 放进容器。别指望审批弹窗当安全边界。

**Q：它说改了但文件不对 / 改错文件了**
`git status`、`git diff` 确认实际差异，然后 `git checkout` 恢复。对话层面可以用 `/tree` 回看它当时读了什么。

**Q：`AGENTS.md` 没生效**
确认文件名和所在目录（父目录也生效），检查同目录有没有 `AGENTS.override.md` 把它顶掉了。改完 `/reload`。启动头部会列出实际加载的指令。

**Q：聊久了它开始忘事 / 没聊几句就压缩**
先看 `/session` 的上下文用量。如果窗口明显偏小，检查 `models.json` 里这个模型有没有写 `contextWindow`（见 5.4）——这是最常见的原因。手动救急用 `/compact`。

**Q：Windows 上 `Alt+Enter` 之类的快捷键没反应**
Windows Terminal 保留了这些组合，Pi 在 Windows 上换成了 `Ctrl+Q` / `Alt+Q` / `Alt+P`。用 `/hotkeys` 看当前实际生效的快捷键。

**Q：图片粘不上**
Windows 上粘贴图片的快捷键是 `Alt+V`（不是 `Ctrl+V`）。

**Q：会话文件在哪**
`~/.pi/agent/sessions/`。`/session` 会打印当前会话的完整路径。`--session-dir` 或 `PI_CODING_AGENT_SESSION_DIR` 可以改位置。

**Q：想换配置目录**
设 `PI_CODING_AGENT_DIR`，默认是 `~/.pi/agent`。

**Q：卸载后配置还在吗**
在。npm 卸载和官方脚本都不会动 `~/.pi/agent/` 里的配置、凭据、会话和已装的包，需要手动删。

**Q：出问题了怎么排查**
`/debug` 把渲染结果和会话消息写到 agent 目录的 `pi-debug.log`；`/session` 看会话状态；`/changelog` 看最近改了什么。**分享这些文件前先自己看一遍**，里面可能有代码和凭据。

---

## 9. 进阶方向（用熟了再看）

Pi 的扩展顺序是官方明确建议的 —— **从最轻的机制开始**：

| 想解决的问题 | 用什么 |
|---|---|
| 给某个目录加长期规则 | `AGENTS.md`（第 6 节） |
| 把常用提示词变成 `/命令` | 提示词模板（`prompts/`） |
| 加任务专属说明和配套文件 | Skill（`skills/`） |
| 加可执行工具、命令、事件钩子 | 扩展（`extensions/`，TypeScript） |
| 接一个协议不兼容的模型服务 | 自定义 provider 扩展 |
| 打包分发上面这些 | Pi 包（`packages`） |

其他值得知道的：

- **Pi 包管理**：`pi install <source>` / `pi list` / `pi config`（TUI 里逐个开关资源）。本机已经装了一个包：`git:github.com/DietrichGebert/ponytail`（见 [`../README.md`](../README.md) 里的 ponytail 一节）。
- **Skill 加载位置**：`~/.pi/agent/skills/`、项目 `.pi/skills/`、以及项目或祖先目录的 `.agents/skills/`（后者受项目信任保护）。启用后可用 `/skill:name` 调用。
- **自定义按键**：`~/.pi/agent/keybindings.json`，按官方 Keybindings 文档里的 action id 覆盖，改完 `/reload`。
- **本地模型**：直接对接 llama.cpp router（`/llama` 管理）；Ollama / LM Studio / vLLM / SGLang 走 5.1 的 `models.json` 兼容端点。
- **非交互与自动化**：`pi -p`、`--mode json`（JSONL 事件）、`--mode rpc`（双向）、TypeScript SDK。
- **隔离运行**：容器化运行 Pi（官方有专门的容器化文档），这是唯一的强边界。
- **代理**：`httpProxy`（只能写在用户级 settings.json）或 `HTTP_PROXY` / `HTTPS_PROXY`。

---

## 附录：一页速查

```text
安装        npm install -g --ignore-scripts @earendil-works/pi-coding-agent   (Node >= 22.19)
启动        cd 项目目录; pi
继续上次    pi -c
选会话      pi -r                       |  pi --session <id>   |  pi --fork <id>
一行问答    pi -p "问题"                |  pi -p --mode json "问题" > out.jsonl
命名会话    pi --name "重构登录模块"
只读模式    pi --tools read,grep,find,ls -p "评审这个模块"
换模型      pi --model sonnet:high "..."   |  pi --models "deepseek-*"
升级        pi update        |  刷新模型目录 pi update --models
装扩展/包   pi install <source>   |  pi list   |  pi config

界面        Enter            提交
            Shift+Enter / Ctrl+J  换行
            Ctrl+G           外部编辑器写长提示词
            @文件 / Tab      引用文件、补全路径
            Alt+V           粘贴图片（Windows）
            !命令            执行命令，输出进上下文
            !!命令           执行命令，输出不进上下文
            Esc              打断
            Ctrl+O           展开/折叠工具输出
            Ctrl+T           显示/隐藏 thinking
            Ctrl+L / Ctrl+P  选模型 / 循环切模型（反向 Alt+P）
            Shift+Tab        切思考等级（不是权限模式！）
            Ctrl+Q / Alt+Q   追加任务 / 收回排队消息（Windows）
            Ctrl+X           复制最后一条回复
            Ctrl+C 两次      退出

配置        ~/.pi/agent/settings.json     偏好、默认模型、defaultTools、compaction
            ~/.pi/agent/models.json       自建网关、自定义模型、contextWindow
            ~/.pi/agent/auth.json         /login 存下的凭据（勿提交）
            ~/.pi/agent/trust.json        项目信任决定
            ~/.pi/agent/keybindings.json  自定义按键
            ~/.pi/agent/AGENTS.md         全局规则
            ~/.pi/agent/sessions/         会话文件
            ./.pi/settings.json           项目级设置（需要项目信任）
            ./AGENTS.md 或 ./CLAUDE.md    项目规则（不信任也加载！）

关键设置    defaultTools        ["read","powershell","edit","write","grep","find","ls"]
            defaultProjectTrust "ask" | "always" | "never"
            compaction.*        enabled / reserveTokens / keepRecentTokens
            shellPath           自定义 Bash 路径（JSON 里反斜杠写两个）
            httpProxy           只能写在用户级 settings.json

上下文窗口  models.json 里每个模型的 contextWindow → 决定 Pi 认为的窗口上限
            （自定义模型不写这项 = 退回保守默认值 = 老是被压缩）
            /session 看实际用量   |   /compact 手动压缩   |   /new 换话题

权限        pi --tools read,grep,find,ls   只读（最实用的一招）
            pi --approve / --no-approve   本次信任 / 不信任项目资源
            /trust                        保存信任决定
            ⚠️ 默认不问每次工具调用；没有内置沙箱，隔离要靠容器

排查        /hotkeys    看当前生效快捷键
            /session    看会话、用量、费用
            /debug      写 pi-debug.log
            pi auth check --provider <name> --json

文档        pi.dev  |  github.com/earendil-works/pi
```
