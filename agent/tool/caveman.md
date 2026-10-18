# caveman：让 Agent 少说废话

**仓库**：<https://github.com/JuliusBrussee/caveman>
**类型**：Skill（提示词规则包，装完对所有对话生效，不涉及代码执行，风险最低）

让 agent 用"原始人语"回答：砍掉所有铺垫、客套和复述问题，只留结论和动作。

> 普通 agent（69 token）："你的 React 组件重新渲染的原因，很可能是因为每次渲染都创建了新的对象引用……"
> caveman（19 token）："新对象引用每次渲染。内联对象 prop = 新引用 = 重渲染。用 `useMemo` 包起来。"

关键点是**代码、命令、文件路径、报错原文不压缩**，只砍散文部分；安全警告和确认提示也保持完整。

## 没有它之前

| 现象 | 以前怎么对付 |
|---|---|
| 一句话能说清的事，agent 先复述问题、再列三点、再总结一遍 | 手动滑过去，或者打断它"直接说结论" |
| 多轮排错后上下文被"好的，我明白了"塞满 | 中途 `/compact`、开新会话重新描述背景 |
| 长输出里真正的 diff / 命令被散文淹没 | 自己往上翻找代码块 |
| 想全局改语气 | 在 `AGENTS.md` / `CLAUDE.md` 手写"回答简短些"，效果随模型心情 |

本质是**只省输出 token，不省思考**：agent 该读的文件一行没少，省的是一轮轮复述和寒暄。

## 优点 / 缺点

| 优点 | 缺点 |
|---|---|
| 零代码、零进程、零配置，卸载就是删目录 | 风格化输出，天天看会累；给同事贴答复前要自己翻译一遍 |
| 代码、路径、报错不压缩，不影响照抄执行 | 散文压缩后有时丢掉"为什么这么改"的推论，接手的人得追问 |
| 三家 agent 通用（都是标准 Skill 格式） | 只减输出，不减输入：日志、测试输出、diff 该烧的 token 还在烧 |
| 额外提供 proxy / middleware 压**输入**（日志、测试输出、JSON、diff），跟在哪个 agent 后面都能用 | proxy 那套要改 base URL，和公司网关叠在一起容易多一层排障面 |

## 安装

```bash
npx skills add JuliusBrussee/caveman -g     # -g = 全局安装，所有项目生效
```

README 说支持 30+ 种 agent。`skills` CLI 会自动探测你装了哪些 agent，往各自的 Skill 目录里写一份。

## 三个 agent 怎么接

三种 agent 的 Skill 目录互不相通（Claude Code **不读** `.agents/skills/`），同一份规则想三家都用，要么让 `skills` CLI 各装一份，要么手动拷：

| Agent | 装到哪 | 调用 |
|---|---|---|
| Pi | `~/.pi/agent/skills/caveman/`（项目级是 `.pi/skills/`，需项目受信任） | `/skill:caveman` |
| Claude Code | `~/.claude/skills/caveman/`（项目级 `<仓库>/.claude/skills/`） | `/caveman` |
| Codex | `~/.codex/skills/caveman/`，或项目 `.agents/skills/caveman/`（Codex 和 Pi 都读这个位置） | `$caveman` |

手动拷的前提是目录里有 `SKILL.md`，且**目录名和 frontmatter 里的 `name` 一致**。

```bash
# 手动装一份给三家（先确认 skills CLI 已经把包拉到本地，或直接从仓库 clone）
git clone --depth 1 https://github.com/JuliusBrussee/caveman /tmp/caveman
cp -r /tmp/caveman ~/.claude/skills/caveman
cp -r /tmp/caveman ~/.codex/skills/caveman
cp -r /tmp/caveman ~/.pi/agent/skills/caveman
```

验证方式：Pi 里 `/` 菜单或 `/skill:caveman`；Claude Code 里 `/skills` 看列表；Codex 里直接 `$caveman 说说这个函数干嘛的`。

> Skill 是提示词，不是硬约束。真要"以后都别啰嗦"，还是得在项目 `AGENTS.md` 里也写一条 —— 三家都读这个文件。

**我的看法**：最适合先装的一个。零风险，效果立刻能感知。真嫌它太糙，`/ponytail` 那种"结论型简洁"通常更好用（见 [`ponytail.md`](./ponytail.md)）。
