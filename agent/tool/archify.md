# archify：把方案讲成可交互的 HTML 图

**仓库**：<https://github.com/tt-a1i/archify>
**类型**：Skill（带校验器 + 渲染器）
**版本**：v3.1.0（MIT，skills.sh 上 130K+ 安装）

把"你想讲清楚的东西"变成**可交互的 HTML 图**：能点节点、能顺着路径追、能发链接让别人打开同一个视角。起点可以是一个想法、一个问题、一个方案，也可以让 agent 直接读仓库生成**带出处**的架构图（节点上标 `SRC n`，点开是某个 commit 下的具体文件行）。

产出是自包含的一个 HTML，双击就能开，不用装 archify。非技术内容也能画——社区里有人做了上海 CityWalk 四日行程，切 D1–D4、点开某个点跳到高德/小红书。

## 没有它之前

| 做法 | 代价 |
|---|---|
| draw.io / Excalidraw 手画 | 布局、对齐、连线全手调；方案一变就重画 |
| Mermaid 写在 Markdown 里 | 自动布局不可控，复杂图上会糊成一团；只能静态看，点不动 |
| PPT 里贴截图给领导 | 图改了要重新截图、重新贴；还是静态的 |
| 群里发一大段文字描述架构 | 对方要在脑子里重建拓扑，来回问三遍 |

差别在于**改图成本和讲解方式**：以前改一次等于重做一次，现在是"再加一条鉴权路径"一句人话；以前讲完对方没法自己往下钻，现在扔个链接他能自己点。

## 优点 / 缺点

| 优点 | 缺点 |
|---|---|
| 可交互 HTML，能点、能追路径、能发链接分享**同一个视角** | 只画 10 类图（架构、流程、时序、数据流、生命周期、ERD、类图、树、时间线、瀑布），象限/甘特/桑基这类没有 |
| **有 typed JSON IR**：图有可 diff 的源，改一个节点不用整张重画 | 产物要给人看才行：进不了 PR 的 Markdown 渲染，得单独找地方放 HTML 或导出图片 |
| 交付前跑校验（schema / 布局 / HTML / 路由 / 标签避让），失败给可机器读的修复单 | **图好看 ≠ 架构正确**，它会忠实画出它以为的结构，错了也画得很专业 |
| 能直接读仓库生成图，节点带出处，钉在某个 commit | 有更新检查（一次 GET 拿版本号），不想要就 `ARCHIFY_UPDATE_CHECK_DISABLED=1` |
| 导出 PNG / 动态版 / 分享卡（能只导出一条路径的 1200×630 卡） | 内部依赖 Node 跑校验器和渲染器，纯对话环境里会退化 |
| 非技术内容（行程、学习路径）也能画，社区改造成小工具的例子不少 | 文档偏平台化，README 里赞助商和付费服务位要自己分清 |

## 安装与使用

```bash
npx skills add tt-a1i/archify -g
```

用法就是直接说人话，然后继续追加：

```text
Use Archify to draw: Browser -> API -> Redis cache -> PostgreSQL fallback.
```
```text
分析这个仓库，用 archify 画一张运行时架构图：8–12 个核心组件、一条主路径、
外部依赖、信任边界，细节放卡片里别加边。
```

然后接着说 "add Redis"、"move auth to the left"、"highlight the rollback path"。它保留 JSON 源，所以这类改动是**局部改**，不是重画。

## 三个 agent 怎么接

| Agent | 装到哪 | 调用 |
|---|---|---|
| Pi | **官方不支持**。可以试 `~/.pi/agent/skills/archify/`，或者直接让它当普通目录文件用 | `/skill:archify` |
| Claude Code | `~/.claude/skills/` 或项目 `.claude/skills/` | `/archify` |
| Codex | `~/.codex/skills/`、`~/.agents/skills/` 或项目 `.agents/skills/` | `$archify` |

官方列的支持范围是 Cursor、Claude Code、Codex CLI、OpenCode，外加 Claude.ai / Project Knowledge 的 zip 装法、Hermes、DeepSeek Harness 两个社区集成。**没有 Pi** —— 它靠 Node 脚本干活，Pi 里能不能跑通没实测；第一次用先拿个小图验证，不行就当普通文件生成器用。opencode 和 Codex 共享 `~/.agents/skills/`，装一份两边都读。

## 图上能干什么

打开 HTML 后是一套浏览器内快捷键：

| 想干的事 | 操作 |
|---|---|
| 看这张图的事实说明 | `?` |
| 搜节点并聚焦 | `/` |
| 追上游 / 下游（作者显式写的关联，不编造运行时影响） | 选中节点 → `Upstream` / `Downstream` |
| 追一条有向路径 | `R` 或 `PATH` |
| 只对比两类角色（比如 backend vs database） | `L` 或 `LENS` |
| 全景雷达图 | `M` 或 `MAP` |
| 演示模式 | `F` |
| 换风格 / 换主题 / 导出 | `S` / `T` / `E` |

链接能带上状态，别人打开就是你看到的那一屏：`#focus=<id>&reach=downstream`、`#route=<a>~<b>`、`#lens=<k1>~<k2>`。动效跟随 `prefers-reduced-motion`，且不会混进正式导出。

## 用起来

1. 先让它读代码/背景，出第一版粗图（带出处的版本会说清查了哪个 commit）
2. 逐条补细节（"这条路径要标注超时时间"）
3. 定稿后导出，别提前调样式

**典型场景**：给同事/领导讲清一个系统怎么跑的 —— 比在群里贴一段文字描述有效得多；事故复盘、发布历史这种"时间轴上发生了啥"用 Timeline / Waterfall 也顺。

**不适合**：要进 Git、每个 sprint 都改的图（用 Mermaid）；只要一张静态配图、要进 PPT 的（用 [diagram-design](./diagram-design.md)，图型 45 种、审美更偏出版）。

## 踩坑清单

- **图画得挺好看但结构不对**：先给事实源（仓库路径、接口文档、调用链），别让它猜。
- **改一句全图重画**：现在有 JSON 源，一般是局部改；真重画了就是把源丢了，让它先读 `*.json`。
- **HTML 打不开 / 样式丢了**：产物是自包含的单文件，如果一个 HTML 拷出去就散架，说明它引用了外部资源，让它重生成。
- **想进文档体系**：定稿后导出 PNG 或分享卡贴进 Markdown；交互版单独存一份发链接。
- **不想让它联网**：`ARCHIFY_UPDATE_CHECK_DISABLED=1`。它只 GET 一次版本清单，不传提示词、项目数据、账号 ID。
- **想自己接手改**：输出就是 HTML，agent 可以继续加交互、加链接——社区那批"行程变工具"就是这么来的。

**适合谁**：需要反复给不同人讲同一个系统结构、并且希望对方能自己往下钻的人。一次性讲完就没人再看的场景，一张静态图就够了。
