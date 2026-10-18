# diagram-design：让 agent 画出不像 AI 画的图

**仓库**：<https://github.com/cathrynlavery/diagram-design>
**类型**：Skill（部分 agent 有插件 / 命令）

跟 agent 说人话，它产出一个**自包含的 `.html`（内联 SVG）**，双击就能开。作者是设计师，整个 skill 核心是"把品味写下来"：一个强调色、1px 细线、无阴影、所有坐标对齐 4px 网格。有 45 种图型，每种带 minimal light / minimal dark / full-editorial 三个静态版本。

设计规则里那句最值钱：**最高质量的动作通常是删掉。** 每个节点都要有存在的理由，强调色只留给读者最该先看的 1~2 个东西。

## 没有它之前

| 做法 | 代价 |
|---|---|
| 直接让 agent 画 SVG / Mermaid | 一堆圆角框 + 渐变 + 阴影，跟你的文档、PPT、网站完全不搭 |
| Mermaid 写进 Markdown | 自动布局不可控，稍微复杂就糊成一团；样式只有渲染器主题 |
| draw.io / Excalidraw 手拖 | 对齐、连线、间距全手调，方案一变整张重画 |
| 在 Figma 里做 | 半小时起，本来只是想给博客配一张图，最后干脆不配了 |

模型本来就把内容理解对了，缺的是**审美**。这个 skill 就是把审美固化成规则。

## 优点 / 缺点

| 优点 | 缺点 |
|---|---|
| 一图一 HTML，内联 SVG，无 JS、无外部图片，离线可开 | 输出质量取决于你用的模型，**同一个提示词两次布局会不一样** |
| 45 种图型，覆盖架构、时序、象限、甘特、桑基、Wardley、爆炸轴测…… | 产物不是可编辑源格式：**没有文本可以 diff**，PR 里没法评审 |
| 能把现有 draw.io / Mermaid / Excalidraw **重画**成这个设计体系 | 排版再好看也不保证结构对，画错了照样画得挺专业 |
| 能抓你网站取色取字体，一键套品牌色 | 默认从 Google Fonts 加载三款字体，是唯一的网络请求（可改 system 字体全离线） |
| 每个模板自带 `role="img"` + `<title>`/`<desc>`，无障碍不是事后补的 | 仓库自己 CI 有重叠 / 裁切检查，但那是给仓库产物用的，不保证你机器上刚生成的那张没重叠 |
| 还带 `/doctor`、`/profile`、导入导出命令，不是一次性玩具 | 迭代很快，文档里的命令半年就可能变 |

## 安装

通用装法（任何认 Agent Skills 的宿主）：

```bash
npx skills add cathrynlavery/diagram-design
```

多个 skill 目录时它会让选 Symlink 还是 Copy —— **选 Symlink**，以后更新一次全生效。

只想要文件、不想挂插件的话，也可以直接拷 / 软链 `skills/diagram-design/` 到目标 skill 目录。想改 `references/style-guide.md` 定制风格的，就克隆下来装本地路径，否则包更新会覆盖你的改动（`~/.diagram-design/profiles/` 里存的品牌 profile 不受影响）。

## 三个 agent 怎么接

| Agent | 装法 | 调用 |
|---|---|---|
| Pi | `pi install https://github.com/cathrynlavery/diagram-design`，然后 `/reload` | `/skill:diagram-design`；额外加载 `/export-diagram`、`/import-mermaid`、`/import-excalidraw`、`/profile`、`/doctor` 这几个 prompt 模板 |
| Claude Code | `/plugin marketplace add cathrynlavery/diagram-design` → `/plugin install diagram-design@diagram-design`，再在 `/plugin` 里开 auto-update | `/diagram-design:export-diagram`、`/diagram-design:import-drawio` 等 |
| Codex | `codex plugin marketplace add cathrynlavery/diagram-design` → `codex plugin add diagram-design@diagram-design` | 自然语言，或插件自带的命令 |

Pi 没有自动更新，拉更新要 `pi update --extensions`。注意 **Pi 侧只有 mermaid / excalidraw 的导入模板，没有 `/import-drawio`**，要重画 draw.io 就在对话里直接说。

## 用起来

```text
"给我画个我们服务的架构图：前端、后端、PG、Redis 缓存"
"要一张 Q2 项目按影响 vs 成本的象限图"
"画个 bearer 调用 + 401 刷新 token 的时序"
"按代码库结构画架构图，标出同步调用"
```

它的习惯是：**先说清选了什么图型、什么尺寸、准备删掉哪些，再开始画**。看到这段计划不对，这时候拦一句最省事。

### 让它长得像你的品牌

```text
"onboard diagram-design to https://yoursite.com"
```

它会抓首页 → 提取主色和字体栈 → 映射成 `paper / ink / muted / accent / link` → **给你看 diff** → 你确认后写进 `references/style-guide.md`。之后所有图都是你的底色、你的强调色、你的正文衬线体。还会检查 `ink` 在 `paper` 上的 WCAG AA 对比度，不合规就提个调整值。

多个客户的场景：品牌存成具名 profile（`~/.diagram-design/profiles/<slug>.md`），项目里放个 `.diagram-design` 文件写 `profile: acme`，各项目互不干扰。

### 重画老图

```text
/diagram-design:import-mermaid README.md --diagram=all --size=slide-16x9 --detail=simplified
```

四个旋钮比"转换"重要得多：

| 旋钮 | 选项 | 影响 |
|---|---|---|
| Format | `html` / `svg` / `png` / `html+png` | 交付物 |
| Size | `doc-inline` / `slide-16x9` / `social-og` / `print-a3-landscape` / `fit` …… | `viewBox` **和字号阶梯**（投屏用 16px 节点名，不是 12px） |
| Detail | `faithful`(≤24 节点) / `balanced`(≤12) / `simplified`(≤7) | 按固定顺序降级：装饰 → 重复 → 叶子簇 → 基础设施 |
| Audience | `engineer` / `mixed` / `executive` | 只改**措辞**不改数量（`JWT · RS256 · :8443` → `token check` → `Sign-in`） |

每次导入都给一张**保真账**：哪些合并了、哪些丢了、哪些完整保留。原来坐标、原配色、原字体、draw.io 的斜线蜘蛛网、Mermaid 的自动布局，一律不带过来。

### 导出

```text
/export-diagram path/to/diagram.html --png-only --scale=3   # Pi 侧
```

- SVG：抽出 `<svg>` 并注入字体，Figma / Illustrator 里能直接开；`--system-fonts` 走离线。
- PNG：Playwright 渲染，默认 2×。一次性准备：`pip install playwright && playwright install chromium`。
- 两种都只导**图本身**，`-full` 变体里的编辑体卡片和页眉不含在内 —— 那个用浏览器打印成 PDF。

## 踩坑清单

- **图能开但字体不对**：默认要联网拿 Google Fonts。要纯离线就让它改用 system 字体。
- **同一个提示词两次排版不一样**：正常。定稿就存下来，别指望"再生成一次一模一样"。
- **让它进 PR / 进文档体系**：定稿后导出 SVG 或 PNG 再贴进 Markdown；HTML 版单独存个位置发链接。Git 里每 sprint 都改的图还是用 Mermaid。
- **第一次用在新项目里它会停下来问**：style guide 还是默认值，让你选"跑 onboarding / 手贴 token / 就用默认"。别当报错。
- **装了 `npx skills add` 又想跟插件更新**：两套是分开的，得先删掉独立安装的再走 marketplace。
- **PNG 导出失败**：多半没装 Playwright 的 chromium。
- 下面这些命令是抄仓库 README 的，我没在三个 agent 上逐个实测。

**不适合**：要进 Git 天天改的图（用 Mermaid）、推文/终端里的 unicode 图、"一个框加一行字"这种一图不如一句话的场合。要**能点、能追路径、能发链接**的交互图，用 [archify](./archify.md)。

**适合谁**：要把架构、流程、方案讲给人看、而且图是要**发出去**的人 —— 博客配图、汇报 PPT、方案文档。一次性只要个能看的草图，让 agent 直接画就够了。
