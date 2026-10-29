# linkcheck · 2026-10-29

## 一、我拆的是哪个项目

[lycheeverse/lychee](https://github.com/lycheeverse/lychee)

- 星标：4,058
- 许可证：Apache-2.0（仓库里同时有 `LICENSE-MIT` 和 `LICENSE-APACHE`，是 MIT / Apache 双许可）
- 语言：Rust
- 一句话定位：用 Rust 写的高速链接检查器，能查 Markdown 和 HTML 里的坏链接和坏邮箱地址。

**为什么选它。** 今天的 GitHub Trending 又是一整页 agent 和 skill 仓库——JetBrains 的 thinkrail、obra/superpowers、Nutlope/hallmark，全是提示词和技能包，没有可拆的核心算法。所以照旧走搜索接口挑候选。

候选里有 fzf（83k★）、bat（61k★）、glow（27k★）、miller（10k★）、tokei（15k★）、lychee（4k★）。筛掉前几个的理由分别是：fzf 和 bat 的痛点在交互和主题，不在算法；miller 是表格工具，上个月刚做过 csvkit 的 csvsift；tokei 是纯统计，做出来只能比速度。

lychee 赢在两点。第一，它的 issue 区里有一整类抱怨是**同一个病**：工具把「服务器没给我 200」当成「链接坏了」。403、429、503、超时，全都变成红色的错误。第二，它的配置面大得离谱——`--help` 数出来 72 个选项，`lychee.example.toml` 里 64 个配置键。这两点合起来，正好是一个可以一刀切进去的位置。

硬性门槛逐条对过：4,058 星（≥1000）；`pushed_at` 是 2026-10-26（90 天内）；许可证 Apache-2.0 在允许列表里；语言 Rust 不在我们的实现栈里，但它解决的是「检查 HTTP 链接」，用 Python 标准库完全够，不构成障碍；上游没做过（之前做的是 jscpd、oxipng、gitleaks、hyperfine 这些，跟链接检查没有重叠）。

---

## 二、它怎么做的

lychee 拆成两个 crate：`lychee-lib` 是库，`lychee-bin` 是命令行。链路很清楚，五段：

```
输入（文件 / 目录 / 通配符 / URL / stdin）
   │
   ▼
collector.rs          从每种输入里抽出链接
   │  extract/markdown.rs  41KB，pulldown-cmark 事件流
   │  extract/html/        html5gum
   │  extract/css.rs       14KB，抓 url()
   │  extract/xml.rs、wikilink
   ▼
filter/               排除规则：正则、scheme、私有 IP、example.com
   │
   ▼
checker/              一个链接 → 一个状态
   │  website.rs  21KB   HTTP 检查
   │  file.rs     27KB   本地文件 + 锚点
   │  mail.rs            邮箱
   ▼
types/status.rs       十种状态
   │
   ▼
formatters/           compact / detailed / json / markdown / raw
   │
   ▼
退出码 2（有错误）
```

**核心是 `checker/website.rs` 里那条请求链。** 它不是一个请求定生死，而是一串决策：

1. 先发 HEAD。注释里写得很直白——有些服务器处理不好某些方法，所以不能只发一种请求就下结论。
2. 如果 HEAD 的结果可疑，换 GET 再试一次。我实测过这一步的价值：语料里有一条路由，HEAD 返回 405、GET 返回 200；另一条 HEAD 返回 404、GET 返回 200。两条 lychee 都判对了。
3. 3xx 就跟着 `Location` 走，最多 10 跳，每一跳记进 `redirect_history`，最后报告里能写「跟了 10 次跳转」。
4. 失败就重试，间隔由 `retry_wait_time` 控制，默认 1 秒。
5. 每个主机走一个 host pool，避免把人家打挂。

**锚点检查是单独一条支路，而且只在 GET 时做。** 源码里写得很清楚：`check_request_fragments = fragment_checker_options.any_enabled() && method == GET && fragment 非空`。因为要判断 `#anchor` 存不存在，必须拿到响应体，HEAD 拿不到。

**它甚至给 GitHub 开了后门。** `handle_github` / `check_github` 两个函数：遇到 github.com 的链接，会去调 GitHub API，因为有些仓库是私有的，普通 HTTP 请求只能拿到 404。这个特例说明作者很务实，也说明「不同站点的规则不一样」这件事，用通用逻辑是盖不住的。

**状态类型设计得比大多数工具好。** `Status` 是个枚举，十个变体：`Ok` / `Error(ErrorKind)` / `RequestError` / `Timeout` / `UnknownStatusCode` / `UnknownMailStatus` / `Excluded` / `Unsupported` / `Cached`。也就是说，在类型层面，作者一开始就知道「失败」不止一种。

但问题就出在最后一步：这十种状态到了报告层，被压成了一个布尔值——「有错误」或「没错误」，对应退出码 2 或 0。类型分得再细，出口只有一个。

---

## 三、它哪里让人难受

**痛点一：403 不是死链，但它被算成死链。**

> Ther are a lot of sites that use cloudflare as protection, checking sites that use cloudflare can result in 403, would be nice to have support for [FlareSolverr](https://github.com/FlareSolverr/FlareSolverr) to fix this situation.
>
> —— [#1439](https://github.com/lycheeverse/lychee/issues/1439)，2024-06-10，至今 open

这条 issue 提的解法是「接一个浏览器求解器去绕过 Cloudflare」——为了一个链接检查器拿到 200，得在 CI 里跑一个几十 MB 的浏览器自动化服务。

方向反了。403 说的是「服务器不想跟这个客户端说话」，它关于客户端，不关于页面。一个真人用浏览器打开那个链接，八成是好的。用户真正需要的不是绕过它，是**别把它算成错误**。这条 issue 开了两年多没有结论，因为它要的东西（一个更细的判决）在原设计里没有位置。

**痛点二：一千个链接跑 15 分钟，中途收不了手。**

> I have more than 600 Markdown pages and more than a thousand of links. It takes 15+ minutes to check them all. Sometimes it may take even longer. Since the checking process is rather non-deterministic, it won't hurt if it stops after the first five minutes. Some links won't be checked, but some 404 links may be detected.
>
> —— [#2076](https://github.com/lycheeverse/lychee/issues/2076)，2026-03-05，至今 open

注意作者用的词：**non-deterministic**。同一个仓库，同样的链接，今天跑 8 分钟，明天跑 20 分钟，取决于网络。在 CI 里这就是最难受的一类失败——你不知道该不该重试。

他要的东西很小：跑满 5 分钟就停，报告已经查到的，并说明还有多少没查。这个功能至今没有。

**痛点三：锚点检查在 GitHub 上直接误报。**

> ```
> [ERROR] https://github.com/googleapis/googleapis/blob/6a8c7914.../error_details.proto#L119 | Cannot find fragment
> ```
> This used to work until last month.
>
> —— [#1729](https://github.com/lycheeverse/lychee/issues/1729)，2025-06-13，40 条评论，至今 open

`#L119` 是代码托管站用 JavaScript 画出来的行号锚点，HTML 里没有这个 `id`。工具在 HTML 里找不到，就报「找不到锚点」。这条 issue 有 40 条评论、跨了几个月，中间还 bisect 到具体哪个提交引入的——因为「锚点在不在」这件事，不同站点规则完全不同，而工具只有「在」和「不在」两个答案，没有第三个答案叫「我查不了」。

**痛点四：三个开关都叫「只查本地」，没有一个真的是。**

> As a new user, I tried to use `lychee` for checking links from local files only. No external requests was expected, I tried to troubleshoot why they were being made and thought `--offline` would apply to my inputs (which were local files with incorrect path, that `lychee` tried to request externally). I was mistaken and `--skip-missing` was the more appropriate option as it applied to inputs, whereas `--offline` does not.
>
> —— [#668](https://github.com/lycheeverse/lychee/issues/668)，2022-06-30，至今 open

同一个需求还有一条 [#1646](https://github.com/lycheeverse/lychee/issues/1646)：「我只想查外链，怎么配？」——4 年过去，答案仍然是「组合几个开关试试」。

`--offline` 管的是「从输入里抽出来的链接」，`--skip-missing` 管的是输入文件本身，`--scheme file` 又是第三个维度。三个概念覆盖一个需求，而用户只想说一句「别联网」。

**痛点五：本地路径的解析有坑。**

> It seems that lychee skips links with absolute path or spaces, and it doesn't handle links with `~` as absolute links.
>
> —— [#1503](https://github.com/lycheeverse/lychee/issues/1503)，2024-09-21，至今 open

带空格的路径、绝对路径、`~` 开头的路径，三种在 Markdown 里都合法，处理方式各不相同。这类 bug 特别烦人，因为它只在特定写法下出现，而写文档的人根本不知道自己在踩。

把这五条放在一起看，会发现它们是同一个问题的五个切面：**工具输出的是「请求的结果」，用户要的是「链接的状态」。** 这两者之间隔着一层判断，而 lychee 把这一层留给了用户——要么你手动把 403 加进 `--accept`，要么你学会忽略红色输出。

---

## 四、我们做了什么

**一句话：一个只把「能证明是坏」的链接算作死链的检查器，8 个选项，零依赖，单文件。**

### 砍了什么，留了什么，加了什么

| 处理 | 内容 | 为什么 |
|---|---|---|
| 砍 | 远程输入（抓网页再爬） | 文档检查器的输入是自己的仓库。支持远程输入就得处理 robots.txt、爬取深度、域名去重，工具会变成爬虫 |
| 砍 | 配置文件的 64 个键 | 一个都不留。所有能改的都在命令行上，且都有能用的默认值 |
| 砍 | `--accept 200..=204,429` 区间语法 | 状态码到判决的映射是一张常量表，十秒能读完。要改判决就改表，不用学一门小语言 |
| 砍 | 邮箱 SMTP 探测 | 探测邮箱是否存在的可靠性极低、误报极高，还要发信。`mailto:` 直接标 `skipped` |
| 砍 | 自定义头 / Basic Auth / Cookie / 代理 / PAC | 查公开文档用不上。要登录才能看的页面本来也不该进链接检查 |
| 砍 | 5 种输出格式 | 留两种：给人看的文本，给机器看的 JSON |
| 砍 | 跨运行缓存文件 | 单次运行内按 URL 去重已经拿到大部分收益，跨运行缓存会引入「结果过期」这个新问题 |
| 留 | 五种判决 | 这是整个工具存在的理由 |
| 留 | HEAD 先探、失败换 GET | 一条注释就解释清楚了，而且是真问题 |
| 留 | 锚点检查（GitHub slug + `id`/`name` + `#L12`） | 改名标题是链接最常见的死法，而且悄无声息 |
| 留 | 重定向跟随与环检测 | 环是死链的一种，只是死法比较绕 |
| 加 | `--only local\|external` | 一个开关表达「只查本地」，一个表达「只查外链」。`--only local` 保证一个 socket 都不开 |
| 加 | `--deadline` | 跑满 N 秒就停，没查的标 `skipped` 并计数。这是 #2076 要的东西 |
| 加 | **根路径探针** | 404 只有在「主机还愿意正常响应自己的根路径」时才算死链 |
| 加 | **读满上限就不下结论** | 页面超过 512KB 时，前 512KB 里没有的锚点只报 `unknown` |
| 加 | 坏链接按目标聚合 | 同一个 URL 出现在 12 个文件里，报告一行，行内列 12 个位置 |

### 核心算法怎么工作

**第一层：状态码 → 判决。** 一张表，纯函数，可以十秒读完：

```
2xx                    → ok
400 404 410 414 451    → dead
401 403 405 406 407 429 503 999 → blocked
408 425 5xx 其它       → unknown
```

这张表是整个工具的观点。403 落在 `blocked` 而不是 `dead`，是因为它关于客户端；500 落在 `unknown` 而不是 `dead`，是因为它可能只是这一分钟不好；404 落在 `dead`，因为它是唯一一个「这个资源不存在」的明确回答。

**第二层：404 要过两道复核。**

第一道是换方法。很多服务器对 HEAD 返回 403 或 404，对 GET 返回 200。语料里我特意放了两条这样的路由：`/head405`（HEAD→405，GET→200）和 `/head404`（HEAD→404，GET→200）。工具在 HEAD 拿到 4xx 时会自动换 GET 再问一次。两条都判成 `ok`，lychee 也判对了——这一条它做得对，我照抄了思路。

第二道是问主机自己的门。规则是：**一个 404 只有在这个主机还能对别的东西正常响应时，才算死链。** 实现上是发一次 `scheme://host/` 的探针，如果根路径也是 404/410，就把该主机所有 404 降级成 `blocked`，理由写成「host returns 404 for its own root - refusing this client, not a missing page」。

这条规则不是想出来的，是被 crates.io 打出来的（见第七节）。

**第三层：锚点。**

Markdown 的标题按 GitHub 的 slug 规则算：小写、去标点、空格转连字符、重复的加 `-1` `-2`。CJK 保留。

```
## Two Words        → #two-words
## What's new?      → #whats-new
## 第一部分 安装      → #第一部分-安装
## Usage（第二个）   → #usage-1
```

三种锚点分别处理：

- 普通锚点：在目标文档里找 slug 或 `id=`/`name=`。
- `#L119` 行号锚点：本地文件按行数验证；远程页面直接报 `unknown`——代码托管站的行号是 JavaScript 画的，HTML 里没有，找不到不代表不存在。
- 页面太大读不完：报 `unknown`，理由里写清楚「page larger than 512 KB, #xxx not verified」。

**第四层：去重与聚合。**

抽取出来的链接先按「目标」折叠：`(去掉锚点的 URL, 锚点)` 作为键。语料里 104 个链接折叠成 24 个目标，也就是说网络请求只有 24 次，不是 104 次。报告时再按目标聚合，一个坏链接占一行，行内列出所有出现位置：

```
dead  404 Not Found   http://127.0.0.1:51999/missing   corpus/02_dead.md:3, corpus/06_dupes.md:4,13,22 (+7 more)
```

这一条是抄自己的：上个月做 copyfind 时学到的——12 处重复打印 11 行，是把「组」当成了「对」。

---

## 五、和原版差在哪

| 维度 | lychee 0.24.2 | linkcheck 1.0.0 |
|---|---|---|
| 判决种类 | 成功 / 失败（内部有十种状态，对外仍是一个错误） | 五种，只有 `dead` 影响退出码 |
| 403 / 429 / 503 | 报错，退出码 2 | `blocked`，退出码 0 |
| 5xx / 超时 | 报错，退出码 2 | `unknown`，退出码 0 |
| 主机根路径 404 | 不检查 | 探一次，命中则把该主机的 404 降级 |
| 页面超过读取上限 | 全量读，不存在这个问题 | 锚点结论降级为 `unknown` |
| 锚点查不到时 | 「找不到锚点」 | 能证明才算死链，否则 `unknown` |
| 命令行选项 | 72 个 | 8 个（+ `-h` / `--version`） |
| 配置文件 | `lychee.toml`，64 个键 | 无 |
| 输出格式 | 5 种 | 2 种 |
| 同一 URL 多处出现 | 分别报告 | 一次检查，一行报告，列出所有位置 |
| 超时预算 | 无 | `--deadline` |
| 远程输入 | 支持 | 不支持（明确取舍） |
| 实现 | Rust，8 个 crate | Python 单文件，标准库 |
| 依赖 | reqwest / tokio / pulldown-cmark / html5gum … | 无 |

**同一份语料的实测对照。** 语料是 6 个 Markdown 文件、104 个链接、24 个不同目标，跑在本机 HTTP 服务上，每条路由的期望判决是人工标注的：6 条真死链、3 条 blocked（403/429/503）、1 条 unknown（500）、其余 ok。

| | linkcheck | lychee 0.24.2 |
|---|---|---|
| 报出的不同问题 | 12 行 | 10 个不同 URL，50 行 |
| 真死链命中 | 6/6 | 6/6 |
| 误报 | 0 | 4（403 / 429 / 503 / 500） |
| 精确率 | 100% | 60% |
| 耗时（timeout 5s，三次） | 4.8 / 4.9 / 5.0 s | 6.1 / 20.0 / 20.1 s |
| 退出码 | 1 | 2 |

两件事值得单独说。

一是 **50 行 vs 12 行**。lychee 对重复链接是分别报告的：语料里有一个文件把 8 个 URL 各重复了 10 次，于是 403 报了 10 行、410 报了 10 行、404 报了 10 行。真问题只有 3 个，报告有 50 行。这个比例在真实仓库里只会更糟。

二是 **耗时 6.1 到 20.1 秒**。同样输入、同样参数、同一台机器，三次跑出三个完全不同的数，最大差 14 秒。我们三次是 4.8 / 4.9 / 5.0。稳定性本身就是 #2076 那条 issue 在抱怨的东西。

---

## 六、怎么用

**零安装，零配置。**

```bash
curl -O https://raw.githubusercontent.com/z15346183030/linkcheck/main/linkcheck.py
python linkcheck.py --version
```

**跑仓库自带的示例**（3 个 Markdown 文件，10 个链接，其中 3 个是坏的）：

```bash
python linkcheck.py examples --only local
```

```
  dead     file not found                   troubleshooting.md          examples/docs/index.md:7
  dead     no #setup in guide.md            guide.md#setup              examples/docs/index.md:8
  dead     file not found                   CHANGELOG.md                examples/docs/index.md:10
  skipped  not a checkable link             mailto:docs@example.com     examples/docs/index.md:11

linkcheck 1.0.0 - 3 files, 10 links (10 distinct), 0.01s
ok 6  dead 3  skipped 1
3 dead links (3 unique) - exit 1
```

三个坏链接各有各的死法：两个文件被删了，一个锚点 `#setup` 在标题改名成 `## Install` 之后失效了。而 `guide.md` 里那两个写在反引号和代码块里的 `[not a link](nowhere.md)`，一个都没报——它们不是链接。

**跑标注语料**（本机 HTTP 服务，每条路由一个特定状态）：

```
linkcheck 1.0.0 - 7 files, 104 links (24 distinct), 4.21s
ok 52  dead 26  blocked 23  unknown 1  skipped 2
26 dead links (6 unique) - exit 1
blocked links were refused by the server, not by the reader; --strict fails on them too
```

**接进 CI**，只要退出码：

```bash
python linkcheck.py docs/ --deadline 120 --exclude 'localhost'
echo "exit=$?"     # 0 干净，1 有死链，2 参数错
```

**要机器可读的结果：**

```bash
python linkcheck.py docs/ --json | python -c "import json,sys; d=json.load(sys.stdin); print(d['counts'])"
```

```json
{"target": "http://127.0.0.1:51999/missing", "doc": "corpus/02_dead.md", "line": 3,
 "verdict": "dead", "code": 404, "reason": "Not Found", "final_url": null}
```

**只想确认自己没联网**：

```bash
python linkcheck.py . --only local
# 32 个文件、15 个本地链接，0.05 秒，一个 socket 都没开
```

**其他开关：**

| 开关 | 默认 | 作用 |
|---|---|---|
| `--only all\|local\|external` | all | `local` 完全不联网 |
| `--timeout SECONDS` | 10 | 单次请求超时 |
| `--jobs N` | 8 | 并发数，每主机最多 4 个在飞 |
| `--deadline SECONDS` | 无 | 跑满就停，剩下的标 skipped |
| `--no-fragments` | 关 | 跳过锚点检查 |
| `--exclude REGEX` | 无 | 可重复 |
| `--strict` | 关 | blocked / unknown 也算失败 |
| `--json` | 关 | 机器可读 |

---

## 七、开发中踩的坑

**坑一：读了一半的页面，被我判成了「锚点不存在」。**

第一版对 `MAX_BODY_BYTES = 512KB` 之外没有处理。拿 lychee 自己的 README 做实测时，它报了一个死链：

```
dead   page has no #tester-lycheeLinkCheck (HTTP 200)   https://nixos.org/manual/nixpkgs/stable/#tester-lycheeLinkCheck
```

我以为是真发现，差点写进报告。抓下来一看，这个页面 2,829,293 字节，锚点在**第 1,417,646 字节**处——正好在我读不到的那一半里。

这不是「链接坏了」，是「我没读完」。修法是：`read(MAX_BODY_BYTES + 1)`，多读一个字节来判断是否被截断；如果截断了，找不到锚点只能报 `unknown`，理由写「page larger than 512 KB, #xxx not verified」。

回归测试：`test_anchor_past_the_read_cap_is_unknown_not_dead`（服务端返回 600KB 填充 + 末尾锚点）和 `test_anchor_inside_the_read_cap_is_still_ok`（锚点在前 512KB，仍然判 `ok`）。

这个坑的教训比 bug 本身重要：**「没找到」和「不存在」是两件事，中间差一个「我找全了吗」。** 上一节那个「页面太大就报 unknown」的规则，就是从这里来的。

**坑二：crates.io 对整站返回 404，包括它自己的首页。**

修完坑一，同一个 README 里还有一条 `https://crates.io/crates/mailify-lib`，404。

我先用 curl 验证，结果 curl 拿到的是 403（crates.io 挡爬虫）。再用工具自己的请求头发，`serde`（肯定存在）、`tokio`（肯定存在）、`mailify-lib`（大概率拼错）**三个全是 404**。

三个都 404，说明 404 在这里不是「不存在」的意思。继续试：`https://crates.io/` 根路径，也是 404；但 `https://crates.io/api/v1/crates/serde` 返回 200。

也就是说，这个站点对不喜欢的客户端，把所有 HTML 路由——包括首页——统统返回 404，只有 API 是通的。这是比 Cloudflare 403 更阴的一种屏蔽：**它用「不存在」的语法，说「不欢迎你」的话。**

修法就是根路径探针：一个 404 出现时，问一次这个主机的根路径；根路径也 404，就把该主机所有 404 降级成 `blocked`。代价是每个出过 404 的主机多一次请求。

回归测试：`test_404_from_a_host_that_404s_its_root_is_blocked`（起一个对所有路径返回 404 的服务，断言判决是 `blocked`、`downgraded == 1`、`dead` 计数为 0）。

代价也要说清楚：如果某个站点的根路径本来就 404（比如某些 API 域名），而它下面确实有真死链，我们会把它判成 `blocked`——漏报。这是有意的取舍：宁可漏一个，不可错杀一片。错杀的代价是整个工具被关掉。

**坑三：加完坑二的规则，语料里的死链全变成了 blocked。**

规则写完，我先跑了标注语料，输出从「6 条 dead」变成「6 条 blocked」，一条死链都没有了。

原因是测试服务器自己：它只认那十几条路由，`/` 落进 default 分支，返回 404。于是根路径探针认为这个主机在屏蔽我们，把所有 404 都降级了。

修法不是改规则，是改测试服务器——给两个服务器（`tests/` 里的和用来跑语料的）都加上 `"/" → 200`。真实站点的根路径几乎都能正常响应，测试环境应该模拟这个，而不是模拟一个异常。

这件事提醒我：**一条依赖「别的请求」的规则，它的测试环境必须把这个「别的请求」也摆对。** 否则测出来的绿灯是假的。

**坑四：锚点检查一度是恒真的。**

写完 `check_local` 之后自查，发现这一行：

```python
if frag in anchors or frag in _dedupe_slugs([frag]):
    return Verdict(OK)
```

`_dedupe_slugs([frag])` 对单个元素去重，返回的就是它自己，所以 `frag in _dedupe_slugs([frag])` 永远为真。整行等价于 `return Verdict(OK)`——锚点检查形同虚设，任何 `#xxx` 都会通过。

这个 bug 有个讨厌的特点：它让工具**更少报错**，所以所有「应该通过」的测试都是绿的。只有「应该失败」的测试才会红，而当时 `test_missing_anchor_is_dead` 还没写。

修法是把那半个条件删掉。教训是：**一个只会让结果变宽松的分支，必须有对应的「必须失败」用例，否则它不会被发现。**

**坑五：每主机串行，一个挂死的主机拖垮全场。**

第一版为了礼貌，每个主机只允许一个请求在飞。逻辑上没问题，实测很惨：拿 lychee 的 README（89 个链接，绝大多数指向 github.com，而这台机器到 github.com 的 HTTPS 基本不通）跑，跑了 **107 秒**还没跑完，最后撞上 `--deadline` 跳过了 14 个。

一个不可达的主机，把所有线程都堵在它那把锁上。这就是 #2076 说的「non-deterministic」的来源——不是工具慢，是它把等待串起来了。

修法：每主机并发从 1 改成 4（常量 `PER_HOST`）。同一批链接重跑，`--deadline 60` 就能跑完。留 4 而不是无限制，是因为文档仓库的链接高度集中在几个域名上，无限制等于对这少数几个站点发压测。

**坑六：裸 `#fragment` 被当成目录。**

`[跳到某节](#things-that-are-not-links)` 这种写法，拆出来是 `base=""`、`frag="things-..."`。`resolve_local("")` 会拼成文档所在的目录，于是检查结果是「fragment on a directory → unknown」。

正确行为是指向当前文档自己。修法是 `build_targets` 里加一个判断：`base` 为空且有 `frag` 时，路径取 `os.path.abspath(doc)`。回归测试：`test_self_anchor_resolves_to_own_file` 和 `test_missing_self_anchor_is_dead`。

**坑七：`KeyError: 'loop'`。**

重定向环检测里，`_fetch` 用字符串 `"loop"` 表示「这是个环」，`check_remote` 直接把它当成判决种类塞进 `Verdict`，然后 `SEVERITY` 表里没有 `"loop"` 这个键，排序时崩了。

修法是在 `check_remote` 里显式映射：`kind = DEAD if err_kind == "loop" else ...`。环是死链的一种，只是死法比较绕——读者永远到不了。回归测试：`test_redirect_loop_is_dead`。

**坑八（小）：`--only local` 的计数是错的。**

第一版 `total += 1` 放在过滤之前，于是 `--only local` 会报「32 files, 119 links」，但实际只查了 15 个本地链接——另外 104 个是外链，被跳过了。数字对不上会让人怀疑工具在偷懒。

修法是把计数挪到每个分支真正加入目标的地方。现在 `--only local` 报的是「15 links (13 distinct)」，跟实际检查数一致。

---

## 八、一句话总结

链接检查器最容易犯的错，不是漏掉坏链接，是**把「我查不到」说成「它坏了」**——403 是服务器不想理你，404 可能是服务器不想理你，页面太大是你没读完；把这三件事分开，CI 才不会因为别人家的 Cloudflare 变红。

---

**仓库**：https://github.com/z15346183030/linkcheck

**实测数据**：标注语料 6 条死链全部命中、0 误报（lychee 同语料报 10 个不同问题，其中 4 个是 403/429/503/500，精确率 60%）；104 个链接折叠成 24 个目标、4.8 秒（lychee 6.1–20.1 秒，三次波动 14 秒）；1000 个链接单主机 0.72 秒；本地 32 个文件 15 个链接 0.05 秒。

**测试**：102 个用例，`python -m unittest discover -s tests`，不依赖外网（自己起 HTTP 服务）。
