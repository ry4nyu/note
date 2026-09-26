# Codex Plugin 使用指南

本文面向使用 Codex CLI 的团队成员。命令和界面会随版本更新，遇到不一致时先执行 `codex --help`。

## 1. Plugin 是什么

Plugin 是可复用能力的打包方式，可以把多个能力放在一起分发和安装。一个 Plugin 可以包含：

| 能力 | 用途 |
|---|---|
| Skill | 为特定任务提供可复用的工作规则、步骤和参考资料 |
| MCP Server | 连接 GitHub、Slack、网盘等外部工具和数据 |
| Hook | 在 Codex 生命周期事件中运行命令 |
| Browser Extension | 为工作流提供浏览器能力 |

Plugin 适合团队共享或跨项目复用的工作流。只需要项目内规则时，优先使用 `AGENTS.md` 或项目级 Skill，不要为了一个任务安装 Plugin。

## 2. 在 Codex CLI 中安装

启动 Codex 后输入：

```text
/plugins
```

在插件浏览器中：

1. 按 marketplace 浏览或搜索 Plugin。
2. 打开详情页，确认来源、包含的能力和权限。
3. 选择安装。
4. 如果需要连接 MCP Server，按提示登录或授权。
5. 新开一个 Codex 会话，再使用已安装的 Skill 或工具。

插件浏览器还支持：

- 查看已安装的 Plugin；
- 卸载可卸载的 Plugin；
- 按空格键启用或停用已安装的 Plugin。

## 3. 使用 Plugin

安装并重新开会话后，直接描述目标即可，例如：

```text
检查当前项目中可能的安全漏洞，并列出需要人工确认的发现。
```

需要指定能力时，明确写出 Plugin 或 Skill 名称：

```text
使用 <plugin-name> 中的 <skill-name>，只检查当前分支，不修改文件。
```

使用外部服务前，确认三件事：

- MCP Server 是否已经完成登录或授权；
- 当前沙箱和审批策略是否允许所需操作；
- 外部服务账号是否有足够权限。

## 4. 权限和安全

Plugin 不会绕过 Codex 的安全控制。通过 Codex 主机运行的能力仍受当前沙箱和审批策略约束；外部服务还会使用自己的认证和权限系统。

安装前检查：

- Plugin 来源是否可信；
- MCP Server 会访问哪些数据；
- Hook 会执行哪些命令；
- 是否会代表你写入、删除或发送外部内容。

不要把 API Key、Token、私钥或其他敏感信息写入 Plugin 配置、Skill 文档或仓库。对 Hook 尤其要先审查脚本，再允许它运行。

如果 Plugin 需要外部服务连接，该服务的条款和隐私政策也适用。不要因为 Plugin 已安装，就默认批准所有外部操作。

## 5. 常见问题

### 安装后 Skill 或工具不可用

先新开 Codex 会话。官方说明中，Plugin 附带的 Skill 会在新会话或新 CLI 会话中生效；MCP Server 还可能需要单独认证。

### 找不到 Plugin

确认当前 CLI 配置了正确的 marketplace，并检查账号或工作区是否有访问权限。企业工作区的 Plugin 可能由管理员导入、同步或统一管理。

### IDE 中不能使用

Codex IDE 扩展目前不支持 Plugin。请改用 Codex CLI 或 ChatGPT 桌面应用中的 Plugin 入口。

### 不想继续使用

在 `/plugins` 中打开对应 Plugin，选择卸载（如果该 Plugin 允许卸载）。工作区安装或默认安装的 Plugin 可能由管理员控制，不能由个人移除。

## 6. 什么时候自己构建 Plugin

只有在以下需求出现时再构建：

- 需要把多个 Skill 或 MCP 能力一起分发；
- 需要让团队从 marketplace 统一安装；
- 需要把一个稳定的外部服务集成到 Codex 工作流。

构建和发布请参考官方文档：

- [Plugins](https://developers.openai.com/codex/plugins/)
- [Build plugins](https://developers.openai.com/plugins/build/plugins)
- [Build an MCP server](https://developers.openai.com/plugins/build/mcp-server)

## 一页速查

```text
打开插件浏览器    /plugins
安装后            新开 Codex 会话
查看/切换状态      /plugins 中操作
项目规则          AGENTS.md 或项目级 Skill
外部服务          MCP Server + 单独认证
IDE 扩展           当前不支持 Plugin
```
