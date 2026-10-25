# Daily Forge

**One GitHub project taken apart per day — then rebuilt smaller, in our own code.**

Not a fork, not a mirror, not a reskin. Read how someone else solved a problem that a thousand stars already validated, then write our own take on it.

Every tool lives in its **own repository** — clone it, install it, use it on its own. This repo only indexes them.

| Project | What it does |
|---|---|
| [**squeeze**](https://github.com/z15346183030/squeeze) | Shrinks JSON / logs before they hit an LLM context — 60–87% smaller, error lines never dropped |
| [**depmap**](https://github.com/z15346183030/depmap) | Module dependency graph for Python, with cycles and dead code flagged |
| [**doc2md**](https://github.com/z15346183030/doc2md) | Zero-dependency `.docx` / `.xlsx` / `.pptx` / `.csv` / `.html` → Markdown |
| [**astdiff**](https://github.com/z15346183030/astdiff) | Structural diff for Python — indentation counts as a change, reformatting doesn't |

All MIT licensed. All runnable — no "coming soon", no "planned support".

<sub>中文说明见下方 · Chinese documentation below</sub>

---

每天从 GitHub 上挑一个千星以上的项目，拆解它的设计，然后**用我们自己的代码**做一个更小、更聚焦、更好用的版本。

不是 fork，不是搬运，不是换个皮。是看完别人怎么解决问题之后，自己动手写一遍。

**每个工具都是一个独立仓库**，可以单独 clone、单独安装、单独使用。这个仓库只做一件事：把它们列出来。

---

## 项目列表

| 日期 | 项目 | 用途 | 灵感来源 |
|------|------|------|----------|
| 2026-10-20 | [**squeeze**](https://github.com/z15346183030/squeeze) | 把喂给大模型的 JSON / 日志压小 60-87%，错误行永不丢弃 | [headroom](https://github.com/headroomlabs-ai/headroom)（76k★，Apache-2.0） |
| 2026-10-20 | [**depmap**](https://github.com/z15346183030/depmap) | 扫描 Python 项目生成模块依赖图，标出循环依赖与死代码 | [graphify](https://github.com/Graphify-Labs/graphify)（129k★，Apache-2.0） |
| 2026-10-23 | [**doc2md**](https://github.com/z15346183030/doc2md) | 零依赖把 docx / xlsx / pptx / csv / html 转成 Markdown，只读 OOXML | [markitdown](https://github.com/microsoft/markitdown)（192k★，MIT） |
| 2026-10-25 | [**astdiff**](https://github.com/z15346183030/astdiff) | 按 Python 语法树比较两个文件，缩进变更算数、重排缩进不算噪音 | [difftastic](https://github.com/Wilfred/difftastic)（26k★，MIT） |

---

## 每日解析

代码之外，每天还有一份文字版解析——讲清楚原项目怎么做的、哪里让人难受、我们改了什么、为什么这么改。比 README 长，比 ANALYSIS.md 好读。

- [squeeze · 2026-10-20](reports/2026-10-20-squeeze.md) — 为什么一个压缩工具不该引入神经网络
- [depmap · 2026-10-20](reports/2026-10-20-depmap.md) — 分析工具报 7 个问题、6 个是假的，比一个都不报更糟
- [doc2md · 2026-10-23](reports/2026-10-23-doc2md.md) — 为了猜一个 .docx 是不是 docx，它加载了一个神经网络
- [astdiff · 2026-10-25](reports/2026-10-25-astdiff.md) — 一个把缩进当噪音的 diff 工具，在 Python 里会把语义变更报成「没有变化」

---

## 为什么做这个

GitHub 上有大量优秀的工具，但它们往往解决的是「所有人的问题」——所以功能臃肿、配置复杂、上手门槛高。

Daily Forge 的逻辑很简单：

1. **找一个被验证过的需求**（千星就是最好的需求验证）
2. **读它的代码和 issue**，搞清楚它做对了什么、又在哪里让人痛苦
3. **砍掉 80% 的功能，只保留最核心的那 20%**，然后把它做到极致简单
4. **用我们自己的实现写出来**，独立成库

每天一个，日拱一卒。

---

## 三条硬规矩

**1. 只学思路，不抄代码。**

拆解阶段会读参考项目的源码，目的是理解它的架构决策和取舍。但写出来的每一行都是自己的。

MIT / Apache-2.0 / BSD 允许在保留版权声明的前提下复用片段——但仍然优先自己实现，实在需要时会在项目的 `ANALYSIS.md` 里明确标注来源和许可证。

**GPL / AGPL 项目的代码一行都不会用。** 这类项目只看思路，实现完全独立完成。

**2. 每个工具必须能跑。**

没有「规划中」「即将支持」。仓库建出来的，就是装好依赖、跑起来、有测试的。

**3. 每个工具必须比原版更简单。**

如果一个工具做完之后，配置项比原版还多、上手比原版还慢，那它就不该存在。简单是唯一的产品指标。

---

## 许可证

每个项目仓库独立采用 MIT 许可证。

各项目的 `ANALYSIS.md` 记录了其灵感来源项目的许可证信息。参考项目的许可证不影响自有代码的授权，但复用的代码片段会单独标注。

---

## 关于「每天一个」的节奏

这个仓库靠一套固定流程推进，不靠灵感。

每天走一遍同样的路：抓 GitHub Trending → 按「千星以上 + 许可证干净 + 能在一天内做出有价值的精简版」筛出候选 → 拆解架构和 issue → 写一个自包含的最小实现 → 跑通测试 → 建独立仓库 → 提交推送。

每一步都有明确标准，写在 [docs/PLAYBOOK.md](docs/PLAYBOOK.md) 里。跑不起来、测试不过的项目不会建仓库——这是唯一不能让步的一条。
