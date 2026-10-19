# Pi 扩展推荐

少数几个装上就懒得再卸的包。全是 npm 源，`pi install` 一条一条装，装完重启 `pi`（或在会话里 `/reload`）。

> 扩展在 Pi 进程内以**你的系统权限**执行代码 —— 能读凭据、能跑命令。装之前先扫一眼源码，风险说明见 [`Extension使用指南.md`](./Extension使用指南.md) 第 9 节。

```bash
pi install npm:@juicesharp/rpiv-ask-user-question
pi install npm:@juicesharp/rpiv-todo
pi install npm:@juicesharp/rpiv-btw
pi install npm:@dietrichgebert/ponytail
```

## 装上以后多了什么

| 包 | 形态 | 给你什么 | 怎么触发 |
|---|---|---|---|
| `rpiv-ask-user-question` | 扩展（工具） | 模型拿不准时开一个最多 4 问的选项卡问卷，每个选项带一句含义说明，还留一行 `Type something.` 让你自己写 | 模型主动调 `ask_user_question` |
| `rpiv-todo` | 扩展（工具 + 命令） | 编辑器上方常驻的待办面板，从对话里重建，`/reload` 和压缩都不丢 | 模型调 `todo`；`/todos` 打印；`Ctrl+Shift+T` 折叠 |
| `rpiv-btw` | 扩展（命令） | 拿当前会话的只读副本问一句就跑，答案只显示在面板里，**不进上下文、不落盘** | `/btw 上次为什么把 socket 换成 SSE` |
| `ponytail` | Skill 包（不是扩展） | "懒资深开发"评审视角：最小实现、找过度设计、列技术债 | 说 "ponytail" / "be lazy"，或用 `/ponytail-review` 等 skill |

## 为什么是这几个

- **省来回**：ask-user-question 和 todo 都把"模型猜错方向"的成本提前挪到你身上，改起来最便宜的时机。
- **不污染上下文**：btw 的问答是临时的，适合"顺手确认一下"而不用 `/new` 或污染长会话。
- **没有配置负担**：四个装完即用，不像 workflow / web-tools 那类要填 API key 或写配置。

`@juicesharp/rpiv-*` 是同一个作者的 monorepo，还有 `rpiv-advisor`（找更强的模型复核）、`rpiv-web-tools`（搜索抓取）、`rpiv-workflow`（多阶段工作流）等，需要再加，别一上来全装 —— 每个扩展都往系统提示和工具列表里塞东西。

## 验证和管理

```bash
pi list                    # 已配置的包
pi config                  # TUI 里逐个开关资源，Tab 切 scope
pi update --extensions     # 已装包重新对齐资源
pi remove npm:@juicesharp/rpiv-btw
pi -ne                     # 一次性全关（含自动发现），排查"是不是扩展干的"
```

装完不确定生效没有，看 `pi` 启动头部列出的资源，或直接敲 `/todos`、`/btw 测试`。

> `pi -p` / `--mode json` 下没有 UI，问卷和 todo 面板不显示；`/btw` 在非交互模式里也没意义。

安装、卸载、包结构的完整说明见 [`Extension使用指南.md`](./Extension使用指南.md) 第 10 节。
