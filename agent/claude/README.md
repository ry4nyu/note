# Claude Code 上手指南

本文基于 Claude Code 2.1.x（Windows + PowerShell 环境），macOS / Linux 差别会单独标注。

---

## 0. 一句话说明这是什么

Claude Code 是一个**跑在终端里的编程助手**。它不是网页版聊天窗口，而是能直接：

- 读你项目里的文件、理解整个代码库
- 帮你改代码、写代码、跑命令、跑测试
- 用 git 提交、发 PR、查 issue

你在项目目录下敲 `claude`，就进入一个对话界面，然后用中文/英文描述需求即可。

---

## 1. 快速开始（三步版）

```powershell
# 1) 安装（PowerShell 里执行）
irm https://claude.ai/install.ps1 | iex

# 2) 配好模型接入（见第 5 节，二选一：官方账号 或 公司网关）

# 3) 进项目目录，启动
cd C:\Users\你的名字\IdeaProjects\你的项目
claude
```

第一次启动会让你选主题、确认工作目录，然后就能直接打字提问了。

---

## 2. 安装

### 2.1 前置条件

| 依赖 | 说明 |
|---|---|
| Git for Windows | **建议装**。装了它才能用 bash 工具；不装也能跑，会自动降级用 PowerShell |
| Windows 版本 | Windows 10 1809+ / Windows 11 均可 |
| 终端 | 推荐 Windows Terminal + PowerShell 7，体验最好 |
| Node.js | 官方安装脚本**不需要**；只有走 npm 安装才需要 18+ |

Git 没装的话去 <https://git-scm.com/download/win> 下载，一路默认即可。

### 2.2 安装 Claude Code

**方式 A：官方安装脚本（推荐，不需要 Node.js）**

```powershell
irm https://claude.ai/install.ps1 | iex
```

装到 `%USERPROFILE%\.local\bin`，如需手动加 PATH 就加这个目录。这种方式支持后台自动更新。

**方式 B：npm（已有 Node.js 18+ 的话）**

```powershell
npm install -g @anthropic-ai/claude-code
```

> **不推荐用 cnpm / 淘宝镜像装**，容易漏文件。
> 如果安装时报 403、地区限制之类的错，先按第 5 节配好网关再试。

**macOS / Linux / WSL：**

```bash
curl -fsSL https://claude.ai/install.sh | bash
```

### 2.3 验证安装

```powershell
claude --version     # 能打印版本号就成了
claude doctor        # 体检，装完建议跑一次
```

> **如果提示 `claude` 不是可识别的命令**：关掉终端重新开一个（PATH 需要刷新）。还不行就手动把安装目录加进 PATH，或用 `claude doctor` 看提示。

---

## 3. 基本用法

### 3.1 启动

```powershell
cd C:\path\to\your\project     # 一定要先进项目目录，它按目录加载上下文
claude                         # 进入交互界面
```

常用启动参数：

```powershell
claude "帮我看看这个项目的登录逻辑在哪"   # 带一句初始问题直接启动
claude -c                                  # 继续上一次对话（continue）
claude -r                                  # 从历史对话列表里挑一个恢复（resume）
claude -p "解释一下 pom.xml"               # 非交互模式，打印结果就退出，适合写脚本
claude update                              # 升级到最新版
```

### 3.2 对话界面的几个关键操作

| 操作 | 效果 |
|---|---|
| 直接打字 + 回车 | 提问 / 下指令 |
| `alt+v` | 粘贴图片给agent |
| `@文件名` | 引用某个文件，输入 `@` 后有自动补全 |
| `!命令` | 直接执行 shell 命令（如 `!git status`、`!mvn test`） |
| `#内容` | 把这条内容记进记忆文件（CLAUDE.md），下次还生效 |
| `/命令` | 斜杠命令，输入 `/` 看全部 |
| `Esc` | 打断它正在干的事 |
| `Esc` `Esc` | 回退到之前某一步（rewind），改错了救命用 |
| `Shift+Tab` | 切换权限模式（见 5.2） |
| `Ctrl+J` 或 `\` + 回车 | 输入换行（不是发送） |
| `Ctrl+C` | 清空当前输入；连按两次退出 |
| `Ctrl+D` | 退出 |

---

## 4. 常用斜杠命令

| 命令 | 作用 |
|---|---|
| `/help` | 帮助，忘了啥命令就敲这个 |
| `/init` | 给当前项目自动生成 `CLAUDE.md`（项目说明书，见第 6 节） |
| `/clear` | 清空对话上下文，开新话题时用（省 token） |
| `/compact` | 压缩上下文，聊太久了用它续命（保留要点，丢掉细节） |
| `/context` | 看当前上下文用了多少 |
| `/status` | 看当前模型、账号、API 地址、工作目录 |
| `/model` | 切换模型 |
| `/config` | 图形化改配置（主题、权限等） |
| `/permissions` | 管理权限规则，减少重复的「是否允许」弹窗 |
| `/rewind` | 回退对话/代码改动 |
| `/resume` | 恢复历史对话 |
| `/cost` | 看这次会话花了多少（官方账号有参考意义） |
| `/doctor` | 体检，出问题先跑它 |
| `/review` | 让它 review 当前分支的改动 |
| `/mcp` | 管理 MCP 外部工具接入 |
| `/agents` | 管理子 agent |
| `/exit` | 退出（等同 Ctrl+D） |

---

## 5. 配置

### 5.1 模型接入（二选一）

这一步是**唯一容易卡住的地方**，按你拿到的账号类型选一种。

#### 方式 A：官方 Anthropic 账号

适用于自己有 Claude Pro / Max 订阅，或从 <https://console.anthropic.com> 申请了 API Key 的情况。

```powershell
claude          # 启动后输入 /login，按提示在浏览器里完成授权
```

或者用 API Key（在环境变量里配）：

```powershell
# 永久写入用户环境变量（配完要重开终端）
[Environment]::SetEnvironmentVariable("ANTHROPIC_API_KEY", "sk-ant-xxxxx", "User")
```

#### 方式 B：公司 / 自建网关

适用于走内部中转网关的情况。需要两个值，**找对接人要**：

- 网关地址（BASE_URL）
- 访问令牌（AUTH_TOKEN）

推荐直接写进配置文件，比设环境变量省事、跨终端生效：

编辑 `C:\Users\你的名字\.claude\settings.json`（没有就新建）：

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "https://网关地址，找对接人拿",
    "ANTHROPIC_AUTH_TOKEN": "令牌，找对接人拿",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "网关提供的模型名",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "网关提供的模型名",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "网关提供的模型名"
  }
}
```

> **注意**：
> - 三个 `ANTHROPIC_DEFAULT_*_MODEL` 都填上网关支持的同一个模型名，否则切换模型时可能报错。
> - 这个文件含密钥，**不要提交到 git**。
> - 改完重开终端或重启 `claude` 生效。
> - 可以一次配完 —— 顺手把默认权限模式也设了（见 5.3），在同一个 JSON 里再加一段：
>   `"permissions": { "allow": [], "defaultMode": "auto" }`，注意和 `env` 之间要加逗号。

#### 确认配好了

启动 `claude`，输入斜杠命令 `/status`，看当前模型和 API 地址对不对。或者在对话里直接问：

```
你是什么模型？当前 API 地址是什么？
```

### 5.2 权限模式（Shift+Tab 循环切换）

Claude 每次要改文件、跑命令，都会先问你。模式决定问得多勤：

1. **manual / 普通模式**：每步都问，最安全，新手上手用这个
2. **acceptEdits（自动接受编辑）**：改文件不问，跑命令还会问，适合你已经信任它的时候
3. **auto（自动判定）**：由一个分类器逐个操作判断，安全的直接放行、可疑的才问你。日常开发最舒服的模式，见 5.3
4. **plan（计划模式）**：只读不改，它先给你一份方案让你确认，**改老代码/复杂需求时强烈推荐先用这个**
5. **bypassPermissions**：啥都不问，危险，只在容器/沙箱里用

> 建议流程：复杂需求先 `Shift+Tab` 进 plan mode 让它出方案 → 方案没问题 → 切到 auto 让它开干。

### 5.3 把 auto 模式设为默认（少点弹窗）

**auto 模式**是日常最实用的选择：不用像 `manual` 那样每个操作都要你点"允许"，又不像 `bypassPermissions` 那样完全裸奔 —— 它用分类器判断每个动作，放行安全的、拦下有风险的。

**三种开启方式：**

```powershell
claude --permission-mode auto          # 仅当前会话
```

或在交互界面里按 `Shift+Tab` 循环切换到 auto（界面上会显示当前模式名）。

**设为永久默认（推荐）** —— 编辑 `C:\Users\你的名字\.claude\settings.json`：

```json
{
  "permissions": {
    "allow": [],
    "defaultMode": "auto"
  }
}
```

改完重开 `claude` 生效。**验证方法**：启动后敲 `/status`，或者直接问它"当前是什么权限模式"。

> 想搞清 auto 到底放行什么、拦什么，跑这几条命令（只读，安全）：
>
> ```powershell
> claude auto-mode config      # 看当前生效的规则（你的自定义 + 默认值）
> claude auto-mode defaults    # 只看出厂默认规则
> claude auto-mode critique    # 让 AI 点评你自己写的规则
> claude auto-mode reset       # 清掉自定义，恢复出厂默认
> ```
>
> 想深度定制的话，自定义规则写在 settings.json 顶层的 `autoMode` 段，分 `environment` / `allow` / `soft_deny` / `hard_deny` 四类。**新手先别动**，默认规则够用。

> ⚠️ **别把 auto 和 bypassPermissions 搞混**：auto 会拦危险操作，`bypassPermissions`（`--dangerously-skip-permissions`）是**什么都不拦**。后者只应该在一次性容器/沙箱里用，千万别在公司机器上开。

`--permission-mode` 支持的完整取值（用 `claude --help` 自己查最新的）：
`acceptEdits`、`auto`、`bypassPermissions`、`manual`、`dontAsk`、`plan`

> `` `default` `` 是 `manual` 的**旧名字**，两者是同一个模式（新版统一改叫 Manual），写哪个都行。

> **两个容易踩的坑**：
> 1. **auto 写在项目级 `.claude/settings.json` 里不生效**，会被静默忽略 —— 必须写在用户级 `~/.claude/settings.json`，或者用 `--permission-mode auto`。`bypassPermissions` 同理。
> 2. **auto 需要模型/套餐支持**。不满足条件时它**不会报错**，而是静默退回 Manual 模式 —— 所以"配置写了却不生效"多半不是配置写错，是模型不支持。用底栏徽标或 `/status` 确认当前实际生效的模式。

### 5.4 让 Claude 默认用 PowerShell（Windows）

**先分清楚这是两件独立的事**，很多人只配了一个然后纳闷为什么没变化：

| 想改什么 | 配哪个 | 作用范围 |
|---|---|---|
| Claude **自己执行命令**时用 PowerShell | `CLAUDE_CODE_USE_POWERSHELL_TOOL=1` | 模型调用的 shell 工具 |
| **你在输入框敲 `!命令`** 时用 PowerShell | `defaultShell: "powershell"` | 只影响 `!` 开头的命令 |

**1）启用 PowerShell 工具**（让 Claude 把 PowerShell 当主 shell，而不是 Bash）

在 `~/.claude/settings.json` 的 `env` 段里加：

```json
{
  "env": {
    "CLAUDE_CODE_USE_POWERSHELL_TOOL": "1"
  }
}
```

- Windows 上这个工具正在逐步默认开启。不想要就把值改成 `"0"` 显式关掉。
- 开启后，Windows 上 Claude 会**优先用 PowerShell**，不再默认走 Bash。
- 它会自动加 `-ExecutionPolicy Bypass`。如果公司安全策略要求遵守执行策略，设 `CLAUDE_CODE_POWERSHELL_RESPECT_EXECUTION_POLICY=1` 关掉这个行为。
- macOS / Linux 上也能开，但需要先装 `pwsh` 并加进 PATH。

**2）让 `!` 命令走 PowerShell**

在 `~/.claude/settings.json` **顶层**（不是 env 里）加：

```json
{
  "defaultShell": "powershell"
}
```

合法值只有两个：`"bash"` 和 `"powershell"`。

> ⚠️ **注意**：`defaultShell` 在**所有平台上默认都是 `bash`，Windows 上也不会自动变成 PowerShell**。所以想用 PowerShell 必须显式设置。

**合起来的完整配置**（网关 + 权限模式 + shell；上下文窗口见 5.5）：

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "https://网关地址",
    "ANTHROPIC_AUTH_TOKEN": "令牌",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "模型名",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "模型名",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "模型名",
    "CLAUDE_CODE_USE_POWERSHELL_TOOL": "1"
  },
  "permissions": {
    "allow": [],
    "defaultMode": "auto"
  },
  "defaultShell": "powershell"
}
```

**怎么验证生效了：**

```powershell
!$PSVersionTable.PSVersion      # 输入框里敲这个
```

- 输出形如 `Major Minor Patch ... 7.4.x` → 走的是 PowerShell ✅
- 输出空白或报错 → 还在走 bash
- 如果显示 `5.1`，说明用的是系统自带的 Windows PowerShell，不是 PowerShell 7，语法上有差别（Claude 会自动适配，但你手写命令时要注意）

也可以直接问它：`你现在执行命令默认用哪个 shell？`

> **小知识**：不要在教程/文档里假设 PowerShell 是 7 —— 先确认 `pwsh` 装没装。装的话推荐用 winget 装 PowerShell 7，体验比 5.1 好很多。

### 5.5 调整上下文窗口与自动压缩（走网关模型必看）

**先理解一个前提**：Claude Code 内置一张「模型 → 上下文窗口」对照表。用公司网关自带的模型（比如 `deepseek-flash`）时，这个模型**不在表里**，Claude Code 就按**未知模型**处理 —— 默认只给 **20 万 token** 窗口，哪怕网关那边声明支持 1M 也不会自动认。

所以「感觉上下文很小、老是压缩」通常不是错觉，是默认值就 20 万。

#### 两个配置项，必须一起设

在 `C:\Users\你的名字\.claude\settings.json` 里加这两项：

```json
{
  "env": {
    "CLAUDE_CODE_MAX_CONTEXT_TOKENS": "1000000"
  },
  "autoCompactWindow": 900000
}
```

| 配置 | 写在哪 | 作用 |
|---|---|---|
| `CLAUDE_CODE_MAX_CONTEXT_TOKENS` | `env` 段，**字符串** | 告诉 Claude Code「这个模型窗口有多大」，覆盖内置表 |
| `autoCompactWindow` | **顶层**，数字 | 聊到多少 token 触发自动压缩，合法区间 100000–1000000 |

**有效自动压缩窗口 = min(模型窗口, autoCompactWindow)** —— 两个都要设：

- 只设 `autoCompactWindow` → 被 20 万的模型窗口卡住，白设
- 只设 `CLAUDE_CODE_MAX_CONTEXT_TOKENS` → 窗口是大了，但压缩阈值还是它自己的默认值

建议 `autoCompactWindow` 比窗口小一截（1M 窗口配 900000），给「生成摘要」留出余量。

#### ⚠️ 关键坑：`modelPicker.behavesAs` 会让上面整条失效

`behavesAs` 的官方含义是「给一个本版本不认识的模型，指定一个它认识的模型 id」。**一旦映射成功，Claude Code 就不再把这个模型当"未知模型"了，于是 `CLAUDE_CODE_MAX_CONTEXT_TOKENS` 被直接跳过，窗口回落成被映射那个模型的值。**

```jsonc
// ✗ 这样写，上面的 1000000 不生效
"modelPicker": {
  "options": [{ "model": "deepseek-flash", "behavesAs": "claude-sonnet-4-6" }]
}

// ✓ 去掉 behavesAs，保住"未知模型"身份，覆盖才生效
"modelPicker": {
  "options": [{ "model": "deepseek-flash", "label": "deepseek-flash" }]
}
```

> **原理**（逆向 claude.exe 2.1.282 得到）：内部算窗口的函数结尾是
> `if (MAX_CONTEXT_TOKENS > 0 && 是未知模型) return MAX_CONTEXT_TOKENS; return 200000;`
> 而「是否未知模型」的判断是：**能把模型 id 解析成任何一个内置 Claude 模型，就返回否**。`claude-sonnet-4-6` 是本版本内置表里的真实 id，所以 `behavesAs` 指向它 = 主动放弃未知模型身份。

#### 验证（改完必须重启 claude）

```powershell
/context        # 看窗口总量是不是 1M 量级
/autocompact    # 看自动压缩窗口的实际数值 + 生效来源
```

`/autocompact` 会显示这个窗口是**从哪来的**（`env` / `settings` / `model default` / 实验值）。**如果来源显示是 `model default`，说明你的两项配置压根没被读到** —— 先回去查 JSON 语法和缩进。

#### 完整的 settings.json（含 1M 窗口）

和 5.4 那份合并起来就是全量配置：

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "https://网关地址",
    "ANTHROPIC_AUTH_TOKEN": "令牌",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "deepseek-flash",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "deepseek-flash",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "deepseek-flash",
    "CLAUDE_CODE_USE_POWERSHELL_TOOL": "1",
    "CLAUDE_CODE_MAX_CONTEXT_TOKENS": "1000000"
  },
  "modelPicker": {
    "options": [
      {
        "model": "deepseek-flash",
        "label": "deepseek-flash",
        "description": "Gateway custom model"
      }
    ],
    "replaceBuiltInOptions": true
  },
  "permissions": {
    "allow": [],
    "defaultMode": "auto"
  },
  "defaultShell": "powershell",
  "autoCompactWindow": 900000
}
```

#### 相关开关（用得上再说）

| 变量/配置 | 作用 |
|---|---|
| `CLAUDE_CODE_AUTO_COMPACT_WINDOW` | `autoCompactWindow` 的环境变量写法。优先级：`/autocompact` > 这个 env > `settings` > 实验值 > 模型默认 |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` | 用百分比下调压缩阈值，比如设 `70` = 用到 70% 就压 |
| `CLAUDE_CODE_MAX_OUTPUT_TOKENS` | 未知模型的单次输出上限默认只有 32000，用这个覆盖（输出上限和上下文窗口是两回事） |
| `DISABLE_AUTO_COMPACT` | 关掉自动压缩（手动 `/compact` 还能用） |
| `DISABLE_COMPACT` | 连手动 `/compact` 一起关掉 |
| `CLAUDE_CODE_DISABLE_1M_CONTEXT` | 关掉 1M 上下文能力（排查用） |

> **别乱加 `[1m]`**：给模型 id 加后缀 `[1m]`（如 `claude-sonnet-4-5[1m]`）是**官方 Claude 模型**申请 1M 窗口的写法，网关自建模型加这个不一定被识别，还可能直接用不了。走网关就用上面 `CLAUDE_CODE_MAX_CONTEXT_TOKENS` 这条路。

---

## 6. CLAUDE.md：让它记住项目规矩

在项目根目录建一个 `CLAUDE.md`，Claude 每次启动都会自动读。写清这些，它会少犯很多错：

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

快速补充的方式：在对话里直接输入 `#不要用 var，团队规范要求显式类型`，它会自动写进 CLAUDE.md。

**这个文件值得认真写**，一次投入长期受益，也是同事之间可以共享的团队资产（可以提交到 git）。

规矩一多，单靠一个 `CLAUDE.md` 会又长又难维护，还每轮都占上下文。这时把约定拆成多个文件放进 **`.claude/rules/`**：不带 `paths` 的启动就加载，带 `paths` 的只在模型碰到匹配文件时才加载。四层记忆（Managed / User / Project / Local）的优先级、`paths` 怎么写、怎么确认规则真的加载了，见 [`rule使用指南.md`](./rule使用指南.md)。

---

## 7. 把它用好的几个习惯

1. **说清楚"在哪、做什么、期望什么"**
   - 差：`登录有问题，修一下`
   - 好：`src/main/java/.../AuthService.java 的 login 方法，用户输错密码时返回的是 500，应该是 401 加错误提示，帮我改`

2. **复杂需求先让它出方案再动手**
   - `先别改代码，读一下 order 模块，告诉我如果要加"超时未支付自动取消"该改哪些地方`

3. **一次只做一件事**，别一口气丢十个需求

4. **用 git 兜底**：让它动手前确认工作区是干净的，搞砸了 `git checkout .` 就恢复。它自己也会经常建议你提交

5. **让它自己验证**：`改完跑一下相关单测` / `启动服务确认接口能通`，比你自己肉眼检查靠谱

6. **聊长了记得 `/compact` 或 `/clear`**，上下文塞满之后它会变笨

7. **不要把它当搜索引擎**：问"Spring 循环依赖怎么解决"不如问"我项目里 A 和 B 循环依赖了，怎么改"

---

## 8. 在 IntelliJ IDEA 里用（可选）

除了终端，也可以装 JetBrains 插件，在 IDE 里直接开面板：

1. IDEA → `Settings` → `Plugins` → 搜索 **Claude Code** → 安装 → 重启
2. 右侧边栏出现 Claude 图标，点开即用
3. 快捷键 `Ctrl+Esc` 唤起

终端和插件共用同一份配置和登录状态，装完不用重新配。

---

## 9. 常见问题

**Q：`claude` 命令找不到**
关掉终端重开。还不行跑 `claude doctor`，或检查安装目录是否在 PATH 里。

**Q：PowerShell 报「禁止运行脚本」**
```powershell
Set-ExecutionPolicy -Scope CurrentUser RemoteSigned
```

**Q：安装时报错 / 403 / 提示不支持当前地区**
Claude Code 有地区检测，走官方账号的话国内直连大概率不行，需要代理。**走公司网关则通常不受影响**，先把 5.1 节方式 B 配好再装或再启动。

**Q：连不上 / 一直转圈**
- 官方账号：检查网络/代理是否正常
- 网关：`/status` 看地址对不对；把 `ANTHROPIC_BASE_URL` 复制到浏览器里试试能不能通
- 令牌过期就找对接人换新的

**Q：它说改了但文件没变 / 改错文件了**
用 `Esc Esc` 或 `/rewind` 回退。改之前养成习惯：先 `!git status` 看工作区是否干净。

**Q：聊久了它开始胡说 / 忘记前面说的**
`/compact` 压缩，或者 `/clear` 重开然后把关键信息重新说一遍。

**Q：上下文窗口太小 / 没聊几句就自动压缩**
网关自带的模型 Claude Code 不认识，默认只给 20 万窗口。按 5.5 设 `CLAUDE_CODE_MAX_CONTEXT_TOKENS` + `autoCompactWindow`；**并检查 `modelPicker.behavesAs` 是不是把模型映射成了内置 Claude 模型** —— 那样两个配置都会失效。用 `/autocompact` 看实际生效值和来源。

**Q：每次都要点「允许」，烦**
两个办法，可以一起用：
1. 把默认权限模式设成 **auto**（见 5.3），让它自己判断，只在可疑操作时问你
2. `/permissions` 里加白名单规则，比如允许 `Bash(git status:*)`、`Bash(mvn test:*)`，常用的放进去就不用反复确认了

**Q：auto 模式是不是就是不检查了？**
不是。auto 会拦截有风险的操作（删重要文件、往项目外写、碰密钥等），`bypassPermissions` 才是完全不检查。两者差别很大，别搞混。

**Q：怎么让它按我们团队的规范写代码**
写进 `CLAUDE.md`（第 6 节），这是最有效的手段。

**Q：出问题了怎么反馈 / 看日志**
`/doctor` 看诊断；`/bug` 可以提反馈。

---

## 10. 进阶方向（用熟了再看）

- **Rule**：把项目约定拆成多个 Markdown 文件放进 `.claude/rules/`，自动加载；还能用 `paths` 只在相关文件上生效，详见 [`rule使用指南.md`](./rule使用指南.md)
- **MCP**：接入外部工具/数据源，比如让 Claude 直接查数据库、查 Jira、查内部文档，详见 [`mcp使用指南.md`](./mcp使用指南.md)
- **Hooks**：在特定时机自动执行脚本（如每次改完文件自动格式化、拦住改 `.env`、收尾前要求先跑测试），详见 [`Hook使用指南.md`](./Hook使用指南.md)
- **自定义 Slash 命令**：把一句话的固定提示词（可带参数）固化成一个 `/命令`，详见 [`command使用指南.md`](./command使用指南.md)
- **Skill**：把一类任务的流程、检查单和附带文件打包成可复用能力，详见 [`skill使用指南.md`](./skill使用指南.md)
- **子 agent（Subagents）**：定义专职 agent，比如"专门做代码 review 的"、"专门写单测的"
- **Headless 模式**：`claude -p "..." --output-format json` 塞进 CI/CD 流水线做自动 review

这些都能通过 `/config`、`~/.claude/` 目录下的配置文件搞定，需要的时候再研究。

---

## 附录：一页速查

```
安装        irm https://claude.ai/install.ps1 | iex
启动        cd 项目目录 && claude
继续上次    claude -c
恢复对话    claude -r
一行问答    claude -p "问题"
升级        claude update
体检        claude doctor

界面        @文件    引用文件
            !命令    执行 shell
            #内容    记入记忆
            /命令    斜杠命令
            Esc      打断
            Esc Esc  回退
            Shift+Tab 切权限模式（含 plan mode、auto）
            Ctrl+J   换行
            Ctrl+D   退出

权限模式    claude --permission-mode auto    本次会话用 auto
            ~/.claude/settings.json → permissions.defaultMode: "auto"
                                             永久默认（项目级不生效！）
            claude auto-mode config          看 auto 生效规则

Shell       env.CLAUDE_CODE_USE_POWERSHELL_TOOL: "1"   Claude 自己用 PowerShell
            defaultShell: "powershell"                 ! 命令走 PowerShell
            !$PSVersionTable.PSVersion                 验证

上下文窗口  env.CLAUDE_CODE_MAX_CONTEXT_TOKENS: "1000000"  模型窗口（字符串！）
            autoCompactWindow: 900000                    压缩阈值（顶层数字，10w–100w）
            ⚠️ modelPicker.behavesAs 一设，上面两项全失效，别设
            /context        看窗口总量
            /autocompact    看压缩窗口实际值 + 生效来源

配置        ~/.claude/settings.json   模型 / 网关 / 权限 / shell / 上下文窗口
            ./CLAUDE.md               项目级规矩（可提交 git）
```
