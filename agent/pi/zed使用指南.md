# Zed 使用 WSL 中的 Pi 指南

本文让 Zed（Windows 版）的 AI 面板直接使用 **WSL 里安装的 Pi**。原理是用 [`pi-acp`](https://www.npmjs.com/package/pi-acp) 适配器：Zed 通过 ACP（Agent Client Protocol）与它通信，它再启动 `pi --mode rpc`。

> 前提：WSL 已安装，见 [wsl.md](wsl.md)。Pi 本身的用法见 [README.md](README.md)。

---

## 1. 先弄清楚：Pi 要装在哪里

Zed 的 Agent 能用哪个 Pi，取决于 **`pi-acp` 在哪个环境里运行**。本文的做法是让 Pi、`pi-acp`、模型配置、会话全部放在 **WSL** 里，Windows 上只保留 Zed。

| 项目 | 位置 |
| --- | --- |
| Zed | Windows |
| Pi、`pi-acp`、Node.js | WSL（Ubuntu 等） |
| Pi 配置与会话 | WSL 的 `~/.pi/agent/` |
| 项目代码 | WSL 文件系统（推荐）或 `/mnt/c/...` |

> **注意**：Windows 上的 `C:\Users\你的名字\.pi\agent\models.json` 对 WSL 里的 Pi 无效。模型和 API Key 要在 WSL 里单独配置。

---

## 2. 在 WSL 中安装 Pi 和 pi-acp

### 2.1 检查 Node.js

Pi 和 `pi-acp` 都需要 **Node.js 22+**。Ubuntu 自带的 `nodejs` 版本常常过低，建议用 nvm 安装：

```bash
# 进入 WSL（在 PowerShell 中执行 wsl 即可）
wsl

# 安装 nvm 并装 Node 22
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
source ~/.bashrc
nvm install 22
node --version    # 应显示 v22.x 或更高
```

### 2.2 安装

```bash
npm install -g --ignore-scripts @earendil-works/pi-coding-agent pi-acp
```

### 2.3 验证

```bash
pi --version      # 需要 v0.81.0 以上
which pi-acp      # 应输出一个路径
```

在 WSL 里配置好模型（`/login` 或 `~/.pi/agent/models.json`），并用 `pi` 能正常对话后再继续。

---

## 3. 在 Zed 中添加 pi-acp

Zed 需要 **v0.225.0 以上**才支持会话历史。

打开 Zed 的 `settings.json`（命令面板搜索 `zed: open settings file`），在 `agent_servers` 中加入：

```json
"agent_servers": {
  "pi-acp": {
    "type": "custom",
    "command": "pi-acp",
    "args": [],
    "env": {}
  }
}
```

如果你已经有 `agent_servers`（比如 `codex-acp`、`omp-acp`），把 `"pi-acp": {...}` 加进去即可，不要重复写这个键。

### 3.1 让 Zed 在 WSL 中运行 pi-acp

上面的配置只有在 **`pi-acp` 能在 Zed 所在的环境里直接找到** 时才行。因此要用 Zed 的 WSL 远程模式打开项目，使 Zed 的 Agent 服务器跑在 WSL 中。打开方式以你当前 Zed 版本的菜单为准（一般在 `File` 菜单或命令面板里搜索 `WSL`）。

打开后，项目路径应是 Linux 路径（如 `/home/你的用户名/code/proj`），而不是 `C:\...`。

> **为什么不推荐用 `wsl.exe` 包一层？**
> 可以把 `command` 写成 `wsl.exe`，`args` 写成 `["-d", "Ubuntu", "--", "pi-acp"]`。但 Zed 会把 Windows 的项目路径（如 `C:\code\proj`）当作 `cwd` 传给 `pi-acp`，`pi-acp` 不做路径转换，WSL 里的 Pi 启动时找不到这个目录。这一点是按 `pi-acp` 源码推断的，未在本机实测。只有项目本身在 WSL 里、且 Zed 传来的是 Linux 路径时才可能成功。

---

## 4. 登录与认证

`pi-acp` 支持终端登录。在 WSL 中执行：

```bash
pi-acp --terminal-login
```

或直接进入 `pi` 后用 `/login` 完成账号登录。认证信息保存在 WSL 的 `~/.pi/agent/auth.json` 中，Windows 端的 Pi 读不到。

---

## 5. 使用

1. 在 Zed 中打开 WSL 里的项目。
2. 打开 Agent 面板，新建会话，在下拉菜单中选择 **pi-acp**（即 `pi-acp` 这一项）。
3. 直接提问、让它改代码。

在对话里输入 `/`，可以看到 `pi-acp` 支持的内置命令（如 `/compact`、`/session`、`/name`）以及 Pi 的技能（`/skill:技能名`，需在 Pi 设置中开启）。

`/model` 不可用，请用 Zed 面板里的模型选择器。

---

## 6. 常见问题

**1. Zed 列表里找不到 pi-acp**
检查 `settings.json` 是否合法（JSON 不能有语法错误），保存后重开 Agent 面板。

**2. 启动后报 `spawn pi ENOENT` 或找不到 pi**
在 WSL 中执行 `which pi`。若无输出，说明 Pi 没装在 WSL 里（只装在了 Windows 上）。回到第 2 节在 WSL 中安装。

**3. 报错提示项目目录不存在**
说明 Zed 传来的是 Windows 路径。按第 3.1 节改用 WSL 远程模式打开项目。

**4. 提示未登录或模型不可用**
这些配置都在 WSL 的 `~/.pi/agent/` 下。在 WSL 中直接运行 `pi`，确认能正常对话。

**5. Node 版本过低**
在 WSL 中执行 `node --version`。低于 22 时，按第 2.1 节用 nvm 升级。

---

## 7. 没有验证的部分

本文基于 `pi-acp` 0.0.34 的 README 与源码阅读撰写，以下内容尚未在本机实测（本机当前未安装 WSL 发行版）：

- Zed 的 WSL 远程模式是否会把 `agent_servers` 的命令放到 WSL 中执行。
- 第 3.1 节对 `wsl.exe` 方式的路径问题判断。
- `pi-acp` 的版本与 Pi 版本的兼容性会随版本变化，遇到问题先看它的 README。
