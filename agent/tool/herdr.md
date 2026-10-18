# herdr：管一堆 Agent 的终端运行时

**仓库**：<https://github.com/herdrdev/herdr>
**文档**：<https://herdr.dev/docs/>
**类型**：终端运行时（Rust 单二进制，Apache-2.0）

herdr 不是又一个终端模拟器，而是**coding agent 的运行时**：把散在不同目录里跑的 agent 装进一个后台 server，前端只负责显示和输入。

- **断连不停活**：关掉客户端或 SSH 掉线，pane 里的进程还在后台 server 里；重连 `herdr` 就回来了
- **多机一个窗口**：本地 + 保存的 SSH 机器放一起，统一的 agent 列表，各自独立重连
- **一眼看出谁卡住了**：每个 pane 标 working / blocked / done / idle，侧边栏按 workspace 汇总
- **agent 原生**：agent 自己能通过 CLI 和 socket API 开 pane、互相发消息、等另一个 agent 真正阻塞
- **不替换你的工具**：Claude Code、Codex、Pi、Cursor、OpenCode 照常用，herdr 只管它们的终端
- **键鼠都是一等公民**：tmux 风格前缀键 + 点选、拖拽分屏、右键菜单

> 本篇按 **Linux / WSL** 写，也是唯一建议的用法。Windows 原生版官方标了 "generally available"，但它把 PTY 换成了 ConPTY，官方自己列了一串 partial / unsupported，实际坑不少 —— 见第九节。

## 没有它之前

| 做法 | 代价 |
|---|---|
| 开几个终端窗口 / Windows Terminal 标签 | 跑两个 agent 就找不着谁在等审批；机器重启全部没了 |
| tmux 手动分屏 | 断线能撑住，但 agent 只是个普通进程 —— 谁是 working、谁是 blocked 全靠你自己看屏幕 |
| 写脚本轮询日志 / 输出文件 | 每个 agent 一套解析，UI 一变就全错 |
| 串行跑任务 | 反正并行也管不过来，干脆一个一个等 |

差的不只是布局，是**状态**：以前"哪个 agent 卡在提问上"需要你逐个 pane 看，现在是一列过滤条件。

## 优点 / 缺点

| 优点 | 缺点 |
|---|---|
| 后台 server 加持：关客户端、SSH 断线都不杀进程 | 多一层需要理解的概念（workspace / tab / pane / agent / session），上手要半小时 |
| 状态可用脚本查询（`agent wait --until blocked`），能真的做“人工审批停在正确的点” | 进程和对话恢复要装集成，不装就只能看屏幕内容；server 重启后 pane 进程全没 |
| 三家 agent 都有官方集成，Pi 也包含在内 | Windows 原生版一堆 partial / unsupported；只用 WSL 侧才舒坦 |
| CLI 对 agent 自己也开放，能自己开 pane 做并行 | 五个 agent 并行就是五份 token 消耗 —— 它只管终端，不管钱包 |
| 不替换现有工具，agent 照常跑 | 单 agent、一个窗口够用时，这层运行时纯属负担 |

## 一、先分清四层概念

| 概念 | 是什么 | 什么时候用 |
| --- | --- | --- |
| workspace | 项目级容器，持有 tab 和 pane | 一个 repo / 任务一个 |
| tab | workspace 里的一套布局 | 拆分「agents / logs / server / review」 |
| pane | 真正的终端，进程活在这里 | 一个 agent 或一条命令占一个 |
| agent | pane 里被识别出来的 agent 进程 | 有状态、能被等待和被提示 |
| session | 独立的 server 命名空间 | 需要完全隔离的 pane 和 socket 时 |

默认命令 `herdr` 连的是 **default session**；命名 session 是另一套 server：

```bash
herdr session list
herdr session attach work
herdr session stop work
```

架构永远是 **后台 server + 一个或多个客户端**。server 持有 pane 和进程，客户端只是 TUI。所以「关掉终端」和「结束任务」是两件事：

```bash
# 断开客户端，活继续干
ctrl+b q

# 结束 session，pane 里的进程全部退出
herdr server stop
```

agent 状态含义：

| 状态 | 意思 |
| --- | --- |
| `working` | 正在跑 |
| `blocked` | 在等你回答 / 审批 |
| `done` | 干完了，你还没看 |
| `idle` | 干完了或闲着，你看过了 |
| `unknown` | 有 agent，但认不状态 |

## 二、安装（Linux / WSL）

```bash
curl -fsSL https://herdr.dev/install.sh | sh

# 或者走包管理器
brew install herdr
mise use -g herdr
```

```bash
herdr --version
herdr status              # server / client 各自的版本和状态
herdr channel show        # stable 还是 preview
herdr update              # 仅限官方脚本安装；brew / mise / nix 用各自包管理器升
```

配置和状态都在 `~/.config/herdr/`：

```text
~/.config/herdr/
├─ config.toml            配置（键位、UI、实验项）
├─ session.json           布局、cwd、agent 会话引用
├─ session-history.json   pane 屏幕历史（默认关闭）
├─ session-snapshots/     最多 48 份布局快照
└─ herdr-server.log       日志，HERDR_LOG=herdr=debug 打开详细日志
```

Windows 原生安装方式见第九节。

## 三、十分钟上手

```bash
cd ~/work/project
herdr          # 启动或连接默认 session，当前目录成为第一个 workspace 的 cwd
claude         # 在 pane 里照常起 agent，herdr 自动识别
```

**鼠标能干掉一切**：点 pane / tab / workspace 就切焦点，拖边框改大小，右键出菜单，拖选文字直接复制（不用 ctrl+c）。

键位是可选的第二层，先记这几条：

| 动作 | 键 |
| --- | --- |
| 进前缀模式 | `ctrl+b` |
| 右分屏 / 下分屏 | `prefix+v` / `prefix+minus` |
| pane 之间移动 | `prefix+h/j/k/l` |
| 新 tab | `prefix+c` |
| 选 workspace（Go To） | `prefix+w` / `prefix+g` |
| 缩放当前 pane | `prefix+z` |
| 看全部键位 | `prefix+?` |
| 复制模式 | `prefix+[` |
| 断开客户端 | `prefix+q` |

几个实用的：

- `prefix+g` 的 Go To 列表按 workspace 分组，每个 agent / 终端一行，`/` 直接搜名字和路径，`b/w/i/d` 按 blocked / working / idle / done 过滤 —— 一堆 agent 的时候这就是主入口
- 分屏、建 tab、建 workspace **默认不抢焦点**，脚本里再显式加 `--focus` 就行
- `prefix+?` 打开的键位帮助里按 `/` 可以过滤

## 四、装 agent 集成：状态才准、重启能续

herdr 识别 agent 靠「进程检测 + pane 底部屏幕内容匹配」。想更准、并且 **server 重启后能把对话接回来**，装集成：

```bash
herdr integration install claude
herdr integration install codex
herdr integration install pi
herdr integration install opencode
herdr integration install cursor
herdr integration status                  # --outdated-only 只看过期的
herdr integration uninstall codex         # 后悔了
```

三家对下来：

| Agent | 集成命令 | 恢复对话靠 |
| --- | --- | --- |
| Pi | `herdr integration install pi` | `pi --session ...` |
| Claude Code | `herdr integration install claude` | `claude --resume <id>` |
| Codex | `herdr integration install codex` | `codex resume <id>` |

集成做两件事：

1. 用 hook 上报 `working` / `blocked` / `idle`，比屏幕匹配可靠（Pi、OpenCode、Kimi、Kilo 这类本来就会自报状态）
2. 上报**原生会话 id**，server 重启后 herdr 用 `claude --resume <id>`、`codex resume <id>`、`pi --session ...` 之类把对话恢复

状态判断错了先别怀疑人生，让它自己解释：

```bash
herdr agent explain w1:p1              # 当前匹配到哪条规则、最终状态
herdr agent explain w1:p1 --verbose    # 加上被评估的全部规则和证据
herdr agent explain --file screen.txt --agent codex --json   # 拿一段屏幕内容离线复现
```

官方 manifest 会从 herdr.dev 自动更新，新形态的审批 UI 一般等一轮就认识了：

```bash
herdr server update-agent-manifests    # 立刻拉取并热加载
```

> `blocked` 的判断故意保守：只有明确匹配到审批 / 提问 UI 才算，匹配不上就退回 `idle`。所以别把 `idle` 当成「可以放心替它点同意」—— 自动化脚本尤其别。

## 五、agent 原生编排

三个原语，别混用：

| 原语 | 管什么 | 用来做 |
| --- | --- | --- |
| layout（workspace / tab / pane 拓扑） | 终端摆在哪 | 开项目、开 tab、分屏 |
| pane | 裸终端 | 跑命令、发输入、读输出、等输出 |
| agent | 有生命周期的 agent 进程 | 发 prompt、等状态、按名字寻址 |

一个能跑的配方（pane ID 一定从返回的 JSON 里取，别猜）：

```bash
split=$(herdr pane split --current --direction right --no-focus)
pane=$(printf '%s\n' "$split" | jq -r '.result.pane.pane_id')

herdr agent start reviewer --kind codex --pane "$pane" -- -m gpt-5.4
herdr agent prompt reviewer "review 这个改动，只报问题" --wait --timeout 120000
herdr agent read reviewer --source recent-unwrapped --lines 120
```

等它卡在提问上、你看一眼再回答，是另一条主线：

```bash
herdr agent wait reviewer --until blocked --timeout 120000
herdr agent read reviewer --source recent-unwrapped --lines 80
herdr agent send-keys reviewer esc
```

普通进程（不是 agent）走 pane：

```bash
herdr pane run w1:p3 "just test --watch"
herdr pane wait-output w1:p3 --regex "passed|failed" --timeout 120000
```

命令怎么选：

| 想干的事 | 用什么 |
| --- | --- |
| 跑命令并回车 | `pane run` |
| 发文字但不回车 | `pane send-text` |
| 发键 / 组合键 | `pane send-keys`、`agent send-keys` |
| 等某段输出出现 | `pane wait-output` |
| 在已有 shell pane 里起 agent | `agent start`（要求该 pane 在 shell 提示符上） |
| 给 agent 发 prompt，可选等结果 | `agent prompt [--wait]` |
| 等 agent 生命周期状态 | `agent wait --until ...` |

几条容易踩的：

- `agent prompt` 遇到已经 `blocked` 的 agent 会直接报 `agent_blocked`，**不发输入**，避免替你点同意
- `--timeout` 不给就是**无限等**；超时和其他错误的 JSON 打到 stderr，退出码 1，语法错误退出码 2
- 超时**不代表没送达**，重发之前先 `agent read` 看一眼，别把同一个 prompt 塞两遍
- 读全屏 agent（Claude Code、OpenCode）的完整历史时，herdr 会自己翻页收集，但 agent 正在干活会报 `agent_not_idle` —— 等它 idle，或者改用 `--source visible`
- 这套 CLI 对 agent 自己也开放。可以在 `CLAUDE.md` / `AGENTS.md` 里写一句：「需要并行时用 `herdr pane split` + `herdr agent start`，等结果用 `herdr agent wait`」

## 六、会话与恢复：到底什么活下来了

| 情况 | 进程 | 布局 | 屏幕内容 | 对话 |
| --- | --- | --- | --- | --- |
| detach 再连回 | 活着 | 回来 | 回来（终端是活的） | 回来（进程没停） |
| server 重启 / 机器重启 | **全没了** | 回来 | 要开 pane history | 要装集成 |
| `herdr update`（默认） | 兼容的 server 继续跑 | 重启后回来 | 同上 | 同上 |
| `herdr update --handoff` | 尽量不杀 | 回来 | 活着就回来 | 进程没停就回来 |

补两句：

- 想让 server 重启后**屏幕内容也能重放**，开 `[experimental] pane_history = true`。注意 pane 输出会落盘，可能含 token 和密钥，把这个目录当成终端历史看待
- agent 对话恢复默认是开的，用 `[session] resume_agents_on_restore = false` 关掉
- 布局被自己搞乱了：`herdr session list --json` 找到 session 目录 → 停 server → 把 `session-snapshots/` 里某份覆盖 `session.json` → 再启动。快照只存布局和会话引用，不存进程
- `--handoff` 是实验特性，且只对官方脚本安装有效（brew / mise / nix 不行）

## 七、多机：本地 + SSH 一个窗口

```bash
herdr machine add workbox --label "构建机"    # 交互终端里跑
herdr machine list
herdr machine status workbox --json          # 非交互地探一下通不通
herdr machine disable <profile-id>

herdr --remote workbox                       # 临时直连一台，不保存
herdr --machine "构建机" agent list           # 把 CLI 命令切到那台机器
```

- 每台机器有自己的 server、session、进程，断一台不影响其他
- `machine add` 可能**装 / 升级远端 herdr，甚至重启远端 server**（要你确认，默认答案是 No，重启会杀掉那台的 pane 进程）。所以别在脚本里跑，用交互终端
- 两种远程姿势要分清：`ssh host` 之后在里面 `herdr`，就是 tmux 式用法（简单、手机 SSH 客户端够用）；`herdr --remote host` 是本地客户端连远端 server，能用本地主题、能把剪贴板图片带过去
- pane / workspace ID 和 agent 名字都是**每台机器自己一套**，两台机器都可能存在 `w1:p1`；忘了 `--machine` 前缀就是在本地乱敲

## 八、配置

```toml
# ~/.config/herdr/config.toml
[keys]
prefix = "ctrl+a"          # 默认 ctrl+b

[ui]
mouse_capture = true       # 关掉就把鼠标还给 pane 和终端

[experimental]
pane_history = true

[session]
resume_agents_on_restore = false
```

```bash
herdr --default-config > ~/.config/herdr/config.toml   # 全量默认值 + 注释
herdr server reload-config                             # 热加载（不是所有项都能热改）
herdr config reset-keys                                # 键位回默认，会先备份 config.toml
```

- **不想用前缀键**：同一个动作可以绑多个键，直连 chord 叠加在 `prefix+*` 上面即可。官方扫过主流终端和桌面环境，`ctrl+alt+*` 这一族最空；踩雷的是 `ctrl+alt+arrows`、`ctrl+alt+t`、`ctrl+alt+f1..f12`
- 直连键没反应，多半是操作系统或外层终端先吃了，改那边或换个键

## 九、Windows 原生：能用，但别当主力

先说结论：

- **日常把 herdr 放在 WSL 里**，Windows 侧只用 Windows Terminal 连进去（`wsl` → `herdr`）
- Windows 原生版换成了 ConPTY + Windows 进程模型，跟 Unix PTY 是两套东西，官方列了一堆 partial / unsupported，实际用起来毛病也明显更多

官方明确**不支持**的：

| 能力 | 影响 |
| --- | --- |
| `herdr terminal attach` | 直连单个终端用不了，只能走完整 UI |
| live handoff（`--handoff`） | 升级要保进程只能靠 Unix；Windows 上更新 server 就得重启，pane 进程全没 |
| Unix fd handoff / 前台进程组 | 进程检测走另一套（扫 pane shell 的后代进程），不是等价的 |
| 本地原生 pane 的剪贴板图片桥 | 图片粘贴看终端和 agent 心情 |
| 签名二进制 | 会撞 SmartScreen |

**partial** 的：shell `cd` 后的实时 cwd（PowerShell 尤其不跟）、CJK 输入法候选窗定位、host cursor 渲染、kitty graphics（Windows Terminal 不支持那条路径）、剪贴板图片粘贴。

具体的坑：

- **光标与输入法**：默认 `host_cursor = "auto"` 在 Windows / WSL 下用**画出来的假光标**（ConPTY 重绘时真光标会闪、会跳）。假光标稳，但**无法给输入法定位**，中日韩候选窗会飘。改 `[ui] host_cursor = "native"` 换回真光标，代价是偶发闪烁
- **ConPTY 自带版本**：Windows 安装包捆了 app-local ConPTY，因为老 Win10 的系统 ConPTY 会丢 Kitty 键盘序列（Kimi、Pi 这类会失灵）。`HERDR_WINDOWS_CONPTY=system` 只在排障时用
- **提权**：管理员身份起的 server 默认只认管理员客户端。要让普通终端连进去得配 `[server] allow_unelevated_clients = true`（Windows 专属，改完要重启 server）
- **插件**：Windows 下是 preview，命令必须是 Windows 能跑的 argv，文档里 `sh`、bash 的例子直接抄会挂
- **复制粘贴**：Windows Terminal 里粘贴是 `ctrl+shift+v`；想用外层终端的右键菜单就 `shift+右键`
- **SSH 多机**：Windows 客户端连远端没问题，但 **Windows 本地侧的 SSH 认证自动恢复还不支持**，`herdr machine reconnect` 得手动来
- **安装位置**：`%USERPROFILE%\.herdr\packages\standalone\releases`，PATH 指向版本目录，`%LOCALAPPDATA%\Programs\Herdr\bin` 是兼容别名 —— 别把 `herdr.exe` 单独拷出来，它要和同目录的 ConPTY 运行时在一起
- Windows ARM64 跑的是 x86_64 版本，靠系统模拟

装：

```powershell
powershell -ExecutionPolicy Bypass -c "irm https://herdr.dev/install.ps1 | iex"
```

端点安全拦 fileless PowerShell 时，改用 cmd：

```cmd
curl.exe -fsSLo install.cmd https://herdr.dev/install.cmd && install.cmd && del install.cmd
```

Windows 原生下的其它环境差异（shell、路径、hook 脚本）见 [`../windows.md`](../windows.md)。

报 Windows 问题请带上：herdr 版本、Windows 版本、终端、shell（PowerShell / cmd）、是否用了命名 session、日志、精确复现步骤 —— 少了这些基本没法定位。

## 十、踩坑清单

- **pane 里的 agent 认不出来**：shell 启动脚本自动进了 tmux。herdr 看到的是 `tmux` 而不是后面的 agent，而且嵌套会自动缩小终端、闪屏。给那段逻辑加个判断：

  ```zsh
  if [[ -z ${TMUX:-} && ${HERDR_ENV:-} != 1 ]]; then
    tmux new-session -A -s my-session
  fi
  ```

- **状态一直 `idle`，但 agent 明明在等审批**：`herdr agent explain <target>` 看匹配到哪条规则；官方 manifest 更新一般能跟上，实在不行就本地覆盖 `~/.config/herdr/agent-detection/<agent>.toml`
- **升级完还是老行为**：`herdr update` 只换客户端，跑着的 server 还是老版本。要 server 端改动就 `herdr server stop` 再 `herdr`（pane 进程会死）
- **`agent read` 报 `agent_not_idle`**：全屏 agent 的历史要从 alternate screen 翻页，herdr 不在它干活时替你翻。等 idle，或 `--source visible`
- **命令行的键位/按钮没反应**：外层终端或桌面环境先吃了 chord，见第八节
- **以为关窗口就完事了**：herdr 的 pane 是后台进程，`exit` 客户端不等于结束。用完 `herdr server stop`（命名 session 是 `herdr session stop <name>`）
- **一次塞太多 agent**：herdr 自己不烧 token，但五个 agent 并行就是五份消耗。按项目分 workspace，用 `blocked` 过滤找人该管的那一个

**适合谁**：已经在同时跑两三个 agent、或者要跑长任务 + SSH 远程的人。单个 agent、一个窗口够用的时候，这层运行时纯属负担。

> 本文命令和结论基于 2026-10 的官方文档（<https://herdr.dev/docs/>，`llms-full.txt`）整理，未在本机逐一复现。
