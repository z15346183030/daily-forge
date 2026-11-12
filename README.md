# Daily Forge

**One small, sharp tool a day — built to do one thing, and explain itself.**

Every tool lives in its **own repository** — clone it, install it, use it on its own. This repo only indexes them.

| Project | What it does |
|---|---|
| [**squeeze**](https://github.com/z15346183030/squeeze) | Shrinks JSON / logs before they hit an LLM context — 60–87% smaller, error lines never dropped |
| [**depmap**](https://github.com/z15346183030/depmap) | Module dependency graph for Python, with cycles and dead code flagged |
| [**doc2md**](https://github.com/z15346183030/doc2md) | Zero-dependency `.docx` / `.xlsx` / `.pptx` / `.csv` / `.html` → Markdown |
| [**astdiff**](https://github.com/z15346183030/astdiff) | Structural diff for Python — indentation counts as a change, reformatting doesn't |
| [**spellcode**](https://github.com/z15346183030/spellcode) | Source spell checker — flags typos, never hashes or version strings |
| [**commitlog**](https://github.com/z15346183030/commitlog) | Git history → CHANGELOG, zero config — next version number included |
| [**csvsift**](https://github.com/z15346183030/csvsift) | One CSV command: select, filter, sort, limit, group, summarise — values stay exactly as written |
| [**yamlsplice**](https://github.com/z15346183030/yamlsplice) | Read and write YAML by path — only that value's bytes change, comments and indentation stay |
| [**shapefind**](https://github.com/z15346183030/shapefind) | Structural search and rewrite for Python — comments and spacing never affect a match |
| [**mddeck**](https://github.com/z15346183030/mddeck) | Markdown → slides, one self-contained HTML — layout measured at build time, overflow shrunk or reported |
| [**benchloop**](https://github.com/z15346183030/benchloop) | A/B commands run interleaved — paired speedup with a 95% interval, plus the process-spawn floor |
| [**leakcheck**](https://github.com/z15346183030/leakcheck) | Hardcoded credentials found by shape, key name and context — samples and placeholders never fire |
| [**pngslim**](https://github.com/z15346183030/pngslim) | PNG, smaller, pixel-identical — only filters and deflate rewritten, header and metadata untouched |
| [**copyfind**](https://github.com/z15346183030/copyfind) | Duplicated code, grouped into clone classes — 12 copies is one entry, not 11 pairs |
| [**linkcheck**](https://github.com/z15346183030/linkcheck) | Docs link checker — a 403 is not a dead link, and neither is a page it could not finish reading |
| [**fuzzrank**](https://github.com/z15346183030/fuzzrank) | Ranks lines by match quality — every score printed as a sum you can check |
| [**findwhy**](https://github.com/z15346183030/findwhy) | File finder that names the rule which removed a path — file and line included |
| [**cronsay**](https://github.com/z15346183030/cronsay) | Cron expression → one sentence, then the next five times it really runs |
| [**httptrace**](https://github.com/z15346183030/httptrace) | One HTTP request, described byte for byte — header case and JSON digits untouched |
| [**importsift**](https://github.com/z15346183030/importsift) | Import sections decided by evidence — stdlib, installed, or yours — and printed line by line |
| [**jumpd**](https://github.com/z15346183030/jumpd) | Directory jumper — exact match always wins, smart case, every score point printable |
| [**quietwatch**](https://github.com/z15346183030/quietwatch) | Runs a command only when file content changes — one save, one run, no ignore rules |
| [**unformat**](https://github.com/z15346183030/unformat) | Reads text back into values with the same format spec that wrote it — `format()` is the only judge |

All MIT licensed. All runnable — no "coming soon", no "planned support".

<sub>中文说明见下方 · Chinese documentation below</sub>

---

每天交付一个自包含的小工具：单一职责、零依赖、能独立安装、能解释自己的每一个判断。

**每个工具都是一个独立仓库**，可以单独 clone、单独安装、单独使用。这个仓库只做一件事：把它们列出来。

---

## 项目列表

| 日期 | 项目 | 用途 |
|------|------|------|
| 2026-10-20 | [**squeeze**](https://github.com/z15346183030/squeeze) | 把喂给大模型的 JSON / 日志压小 60-87%，错误行永不丢弃 |
| 2026-10-20 | [**depmap**](https://github.com/z15346183030/depmap) | 扫描 Python 项目生成模块依赖图，标出循环依赖与死代码 |
| 2026-10-23 | [**doc2md**](https://github.com/z15346183030/doc2md) | 零依赖把 docx / xlsx / pptx / csv / html 转成 Markdown，只读 OOXML |
| 2026-10-25 | [**astdiff**](https://github.com/z15346183030/astdiff) | 按 Python 语法树比较两个文件，缩进变更算数、重排缩进不算噪音 |
| 2026-10-25 | [**spellcode**](https://github.com/z15346183030/spellcode) | 源码拼写检查，哈希与版本号一律不碰，放行只需一行注释 |
| 2026-10-25 | [**commitlog**](https://github.com/z15346183030/commitlog) | 零配置把 git 历史变成 CHANGELOG，顺手算出下一个版本号（含预发布） |
| 2026-10-25 | [**csvsift**](https://github.com/z15346183030/csvsift) | 一个命令完成切列 / 筛行 / 排序 / 截断 / 分组 / 统计，值永远是原始文本 |
| 2026-10-26 | [**yamlsplice**](https://github.com/z15346183030/yamlsplice) | 按路径读写 YAML，只替换那个值的字节，注释 / 空行 / 缩进 / emoji 键名原样保留 |
| 2026-10-26 | [**shapefind**](https://github.com/z15346183030/shapefind) | 按语法树结构搜索和重写 Python 代码，注释与空格永不干扰，零配置零依赖 |
| 2026-10-27 | [**mddeck**](https://github.com/z15346183030/mddeck) | Markdown 一键变幻灯片，构建时算好每页排版，溢出自动缩字号或直接报出还差几像素 |
| 2026-10-27 | [**benchloop**](https://github.com/z15346183030/benchloop) | 交错跑 A/B 命令，用配对数据算加速比与 95% 区间，顺带量出进程创建地板 |
| 2026-10-28 | [**leakcheck**](https://github.com/z15346183030/leakcheck) | 按形状 / 键名 / 上下文找硬编码密钥，占位符和文档示例永不误报 |
| 2026-10-28 | [**pngslim**](https://github.com/z15346183030/pngslim) | 无损瘦身 PNG，只重排滤波和 deflate，图像头 / 调色板 / 元数据一字不动 |
| 2026-10-29 | [**copyfind**](https://github.com/z15346183030/copyfind) | 找重复代码并按克隆类分组，12 处重复只报 1 条；许可证头单独排除 |
| 2026-10-29 | [**linkcheck**](https://github.com/z15346183030/linkcheck) | 检查文档链接，只有「能证明坏掉」的才算死链；403 / 429 / 5xx / 超时一律不判死 |
| 2026-10-30 | [**fuzzrank**](https://github.com/z15346183030/fuzzrank) | 按匹配质量给行排序，每一分的来源都能逐字符打印出来，各项之和恒等于总分 |
| 2026-11-02 | [**findwhy**](https://github.com/z15346183030/findwhy) | 找文件，并说出每条被跳过路径是被哪个文件的哪一行拿掉的 |
| 2026-11-02 | [**cronsay**](https://github.com/z15346183030/cronsay) | 把 cron 表达式读成一句话，再列出真实的下次运行时间；步长不整除时绝不说「每 N 分钟」 |
| 2026-11-02 | [**httptrace**](https://github.com/z15346183030/httptrace) | 发一个 HTTP 请求并把交换逐字节讲清楚：头部大小写原样、JSON 数字不动、dry-run 与真实发送共用同一份字节 |
| 2026-11-03 | [**importsift**](https://github.com/z15346183030/importsift) | 把 import 分成标准库 / 第三方 / 本项目并排序，每条判断都打印依据；写回前重新解析校验，import 集合变了就不写 |
| 2026-11-05 | [**jumpd**](https://github.com/z15346183030/jumpd) | 带解释性评分的目录跳转——精确匹配永远赢，智能大小写，尾部斜杠精确匹配，每分都能打印 |
| 2026-11-10 | [**quietwatch**](https://github.com/z15346183030/quietwatch) | 只按文件内容变化触发命令，一次保存只跑一次，零配置零忽略规则 |
| 2026-11-12 | [**unformat**](https://github.com/z15346183030/unformat) | 用写出文本的那段格式描述把值读回来，宽度 / 填充 / 千分位由 format() 自己判定 |

---

## 为什么做这个

日常开发里真正高频的需求，往往只需要一个很小的工具。但一旦一个工具要服务所有人，它就会长出配置系统、插件系统、兼容层，最后你为了做一件很简单的事，先要读三十页文档。

Daily Forge 反过来做：

1. **从一个具体需求出发**，一句话写清楚输入、输出和边界
2. **设计最小接口**，默认参数就能用，配置项越少越好
3. **用标准库写出完整实现**，不引入不必要的依赖
4. **拿真实数据验证**，把效果量化出来，而不是写完就说「更好」

每天一个，日拱一卒。

---

## 三条硬规矩

**1. 一个工具只解决一个问题。**

如果一个工具需要三种模式、两层配置、四个插件才能用，那它就该被拆开。简单不是营销词，是产品指标。

**2. 每个判断都要能解释。**

排序要能打印每一分的来源，分类要能说出依据，跳过要能指出是哪条规则。工具可以有自己的偏好，但不应该有黑箱。

**3. 每个仓库都必须能独立跑起来。**

没有「规划中」「即将支持」。仓库建出来的，就是装好依赖、跑起来、有测试、有示例的。跑不通、测不过的工具不会建仓库。

---

## 许可证

每个项目仓库独立采用 MIT 许可证。

代码与文档均为本仓库作者所有，第三方许可证与归属信息（如有）随对应仓库一并提供。

---

## 关于「每天一个」的节奏

这个仓库靠一套固定流程推进，不靠临场发挥。

每天走一遍同样的路：确定需求与边界 → 设计最小接口 → 写单文件实现 → 用真实数据测出数字 → 跑通测试 → 建独立仓库 → 提交推送。

每一步都有明确标准，写在 [docs/PLAYBOOK.md](docs/PLAYBOOK.md) 里。
