# spellcode · 2026-10-25

## 一、我拆的是哪个项目

[crate-ci/typos](https://github.com/crate-ci/typos)

- 星标：4,195
- 许可证：Apache-2.0（仓库同时提供 MIT / Apache-2.0 双许可）
- 语言：Rust
- 一句话定位：**在源码里找拼写错误，快到能塞进每次 CI，准到不需要人盯着看。**

为什么选它？三个理由。

第一，它的需求是被验证过的，但痛点也是被验证过的。打开它的 issue 列表，排在前面的几条——哈希被当成单词、没有「这一行别查」的开关、配置项在子目录里失效——全是同一个根源：**它做了一套足够通用的东西，于是必须为通用性付出代价**。这正是适合做精简版的形状。

第二，它的核心是一个可以完整想清楚的问题。拆词、判断「这像不像词」、查表，三步，没有隐藏的复杂度，不需要数据库、不需要网络、不需要模型。这意味着我能在一天里做出一个**逻辑上说得通**的实现，而不是一个「大概能跑」的玩具。

第三，它跟我这几天做的东西不重样。前面几个是压缩、依赖图、文档转换、结构 diff，都在「数据的形状」上做文章；这个是**判断「这个词是不是错的」**，是语义层面的，换了个脑子。

顺带说一句，今天 GitHub Trending 上一水儿的 AI agent 工具，一个都不符合「能砍掉 80% 功能还能更好用」这个前提。所以走了搜索接口，按星标和最近提交筛的。

## 二、它怎么做的

### 先看它的形状

```
crates/
├── typos/            核心库：拆标识符、拆词、查表
├── typos-cli/        命令行外壳：文件遍历、输出格式、退出码
├── typos-dict/       错拼对照表（生成产物）
├── typos-vars/       词的变体处理
├── varcon/           varcon 语料（美式/英式拼写变体）
├── varcon-core/      varcon 解析
├── dictgen/          从上游语料生成词典
├── wikipedia-dict/   维基语料
├── codespell-dict/   codespell 语料
└── misspell-dict/    misspell 语料
```

十个 crate。看起来吓人，但真正跑一次检查只用到前三个：`typos-cli` 遍历文件，`typos` 干活，`typos-dict` 提供数据。剩下七个是「怎么把词典造出来」的流水线。

这条流水线是它最聪明的地方：词典不是手写的，是从 varcon（英美拼写变体）、codespell、misspell、Wikipedia 几份语料里**生成**出来的。语料更新，跑一遍生成器，核心代码一行不用动。

### 核心链路

```
文件 ──→ 按 XID_Continue 抓标识符 ──→ 查「标识符级」词典
                                          │
                                          ├─ 命中 → 报
                                          └─ 没命中 → 拆成词
                                                        │
                                                        ├─ 按大小写变化拆
                                                        ├─ 按字母数字边界拆
                                                        └─ 连续大写当缩写，不拆
                                                              │
                                                              └─ 查「词级」词典 → 报
```

两个关键决策值得单独说。

**决策一：用「错拼对照表」，不用「正确词表」。**

这是整个项目的立身之本，官方 design 文档第一节就在讲这件事：

> Corrections: Known misspellings that map to their corresponding dictionary word
> Dictionary: A confidence rating is given for how close a word is to one in a dictionary

传统拼写检查是「这个词在不在词表里」，判不出来就给个置信度让人选。这在编辑器里没问题，在 CI 里就是灾难——词表里没有的专有名词、缩写、变量名，全变成待办事项。

反过来做，只认「我确定这是错的」，代价是覆盖率（冷僻错拼一律放过），收益是**每一条报告都值得看一眼**。这个取舍一旦定了，后面所有设计都是它的推论。

**决策二：标识符和词分两级，缩写整体保留。**

源码里绝大多数文本是标识符，不是句子。所以先按标识符查一遍，不行再拆词。拆的时候有个特殊规则：

| Identifier | 拆出来 |
|---|---|
| `snake_case` | `snake`, `case` |
| `CamelCase` | `Camel`, `Case` |
| `First10HTMLTokens` | `First`, `HTML`, `Tokens` |

第三行是关键。如果按「大写字母开头就断」的朴素规则，`HTMLParser` 会被拆成 `H`/`T`/`M`/`L`/`Parser`——四个单字母，全被跳过，等于这段代码没检查。所以必须把连续大写识别成一个整体。

### 配置那一层

`typos.toml`（或 `_typos.toml`）里能写的东西不少：

- `[default] extend-words`：给某个词指定「正确拼法」，等于放行
- `[default] extend-identifiers`：放行整个标识符
- `[default] extend-ignore-identifiers-re`：用正则放行
- `[type.po] extend-glob` + `check-file = false`：某种文件只查文件名不查内容
- `[files] extend-exclude`：排除文件
- `[files] ignore-vcs`：跟 `.gitignore` 联动

再配上 `--files` / `--identifiers` / `--words` / `--dump-config` 这一套观察工具。设计得很完整——完整到普通人第一次用，得先花二十分钟搞清楚「为什么我改的词没生效」。

## 三、它哪里让人难受

### 1. 哈希和版本号还是会被当成词

issue [#415](https://github.com/crate-ci/typos/issues/415)，2022 年 1 月开的，19 条评论，到现在还开着：

> ```
> error: `Ba` should be `By`, `Be`
>   --> ./content/blog/2017/08/2017-08-08-introducing-jenkins-minute.adoc:39:14
>    |
> 39 | video::FhDomw6BaHU[youtube, width=852, height=480]
> ```

`FhDomw6BaHU` 是个 YouTube 视频 ID。拆词器把它切成了 `Fh` / `Domw6` / `Ba` / `HU`，然后拿两字母的 `Ba` 去查表——表里确实有 `ba -> by`。

同一个毛病在 2025 年 8 月的 [#1363](https://github.com/crate-ci/typos/issues/1363) 里换个场景又出现一次：

> ```
> github.com/dprotaso/go-yit v0.0.0-20220510233725-9ba8df137936 // indirect
> ```
> ### Incorrect correction
> error: `ba` should be `by`, `be`

一个 12 位十六进制哈希，只因为里面有 `ba` 两个字母。报告者的结论是：

> I will just exclude the file from being checked, as there is not much use to it anyways.

这就是误报的真实代价。不是「多看一眼」，是**整个文件从此不再被检查**。一个工具让人主动关掉它，比漏报还糟。

根因很清楚：两字母的条目在自然语言里是词（by、be），在源码里几乎总是碎片。「这段像不像词」的关卡没拦住它，直接进了查表。

### 2. 没有「这一行别查」的开关

issue [#316](https://github.com/crate-ci/typos/issues/316)，2021 年 7 月开的，28 条评论，open 到现在：

> I'm wondering why there is no directive that can be used to ignore a specific line? I have some files that have intentional typos in them that should be ignored, while the rest of the file should still be checked!
> I'm thinking of a solution like `clang-format`s:
> ```C++
> // clang-format off
> ```

官方给的 workaround 写在 issue 最上面：

> Workaround: use `extend-ignore-re` to create your own file-level, line-trailing, or line-leading ignores

翻译一下：想忽略一行，去 TOML 里写条正则，然后祈祷这条正则只命中那一行。

文件级开关有，全项目开关有，唯独「这一行」没有。

为什么这是个问题——**放行的理由和放行的动作分居两地**。半年后回来看配置文件里那条 `"AttributeID.*Supress.*"`，没人知道它为什么在那儿。README 自己的示例里甚至给这条正则配了句注释：`*sigh* this just isn't worth the cost of fixing`。这句话很诚实，也很说明问题：当工具的表达方式不够用时，用户会用它来留下「我放弃了」的痕迹。

### 3. 配置项在子目录里失效

issue [#593](https://github.com/crate-ci/typos/issues/593)：

> When `$DIR/typos.toml` contains `[files] extend-exclude = ["a/*.txt"]` and we run `typos` in `$DIR`, the file `a/test.txt` is correctly excluded. But when `typos` is run in `$DIR/a`, the exclude is not respected

排除规则是相对**当前工作目录**算的，不是相对配置文件位置。同一条规则，换个目录跑就失效。

这类问题只有在「有一套配置系统」的前提下才存在。路径解析、继承、优先级——每加一层，就多一组这样的边角。而对一个拼写检查器来说，配置系统本身不产生任何价值，它只负责让工具**看起来**能适应所有情况。

### 4. 修正复数会改出更糟的结果

issue [#745](https://github.com/crate-ci/typos/issues/745)：

> `typos` seems to accept `POSTs`, `HEADs`, `GETs`, but not `DELETEs` (nor `OPTIONs`, but doesn't make much sense so it's OK):
> ```
> error: `DELET` should be `DELETE`
> ```
> When I run `typos -w`, I end up with: `DELETEEs` and `OPTIONNs`.

`DELETEs` 被剥成 `DELET` + `s`，`DELET` 命中词表，修正成 `DELETE`，后缀再拼回去，就成了 `DELETEEs`。

一个「发现错误」的工具，`-w` 之后把文件改得更错了。根因是词典里存在「某个正确词的前缀」——`delet` 是 `delete` 的前缀——剥后缀这一步会把正确词拆坏。

## 四、我们做了什么

### 一句话定位

**一个只在有把握时开口的源码拼写检查器：零配置，行内放行，哈希和版本号一律不碰。**

### 砍了什么，留了什么，加了什么

| 动作 | 项目 | 为什么 |
|---|---|---|
| 砍 | TOML 配置系统 | 90% 的用法是默认跑一遍。剩下 10% 用行内注释就够，不需要一门能表达正则、glob、文件类型、路径继承的小语言 |
| 砍 | 十个 crate 的模块拆分 | 核心只有三步。拆 crate 是为了让编辑器复用和词典生成，不是为了能跑 |
| 砍 | 语料生成流水线 | 那是几万条词条的工程。我要的是「435 条确定不会错的」，覆盖率低一个数量级可以接受 |
| 砍 | 文件类型识别 | 判断「是不是文本」用一次 UTF-8 解码加空字节检查就够 |
| 砍 | Git 历史扫描、`--diff` | 工作区 + `--fix` 覆盖实际用法，扫历史是另一个工具 |
| 留 | 错拼对照表（不是正确词表） | 项目成立的前提，改了就不是同一个东西 |
| 留 | 标识符 → 词的两级拆分，缩写整体保留 | 没有它，`HTMLParser` 这种最常见的命名会被拆碎 |
| 留 | 变形直接进表（`recieves`、`supressed`） | 比运行时做词干还原准，代价只是表长一点 |
| 留 | 退出码 0 / 2 / 1 | 进 CI 的前提，跟原版一致，迁移零成本 |
| 留 | `--json` 逐行输出 | 接入其他工具的唯一接口 |
| 加 | **判废层** | 直接对上 #415 和 #1363 |
| 加 | **全大写三字母当缩写** | `TEH` 是阿拉伯字母名，不是 `the` 的错拼 |
| 加 | **行内指令** | 对上 #316 |
| 加 | **前缀保护** | 对上 #745，也修掉自己引入的 `standard -> standarddd` |

### 判废层：这次改动的核心

一段文本要被报出来，得先证明它「像个人写的词」。四条判据，命中任何一条直接跳过：

| 判据 | 例子 | 拦住了什么 |
|---|---|---|
| 数字占比 > 30% | `9ba8df137936`（10/12 = 83%） | go.mod 里的伪版本哈希（#1363） |
| 纯十六进制且带数字，长度 ≥ 6 | `deadbeef12`、`0xff00aa` | 摘要、颜色值、内存地址 |
| 无元音且长度 ≥ 4 | `xkcdwrt`、`HTTP` | 缩写、压缩过的名字 |
| 长度 ≥ 24 | `a3f9c2e1b7d4...` | base64 片段、长哈希 |

外加一条最短词长：**少于 3 个字母的词根本不查**。`ba` 这个条目在表里都不存在了，从源头断掉。

为什么是 30% 而不是 50%？因为我拿标准库的 40 万行试出来的：30% 能拦住所有版本号和哈希尾巴，同时没有一个正常标识符被误杀。50% 会漏掉 `v1a2b3` 这类。

### 前缀保护：一行代码修掉两个 bug

剥后缀查词干是这样工作的：

```
recieves  --剥 s-->  recieve  命中  --> receives
supressed --剥 ed--> supress  命中  --> suppressed
standard  --剥 d-->  standar  命中  --> standardd   ← 灾难
```

`standar` 是错拼（少个 d），`standard` 不是。但剥掉 `d` 之后，`standard` 正好变成了 `standar` 的「变形」，于是被「修正」成 `standardd`。

修法是一句判断：

```python
if stem in TYPOS:
    if TYPOS[stem] == low:   # 修正结果正好等于原词，说明原词就是对的
        continue
    return TYPOS[stem] + suffix
```

这也顺带解决了 `DELETEs` 的问题：`deletes` 剥 `s` 得 `delete`，`delete` 不在表里（我刻意没收 `delet -> delete` 这种前缀条目），于是不报。原版在这里报 `DELET`，`-w` 之后变成 `DELETEEs`。

### 拆词，举例

`def recieve_payload(raw):`

1. 抓标识符：`def`、`recieve_payload`、`raw`
2. `recieve_payload` 过判废层：数字占比 0，有元音，长度 15 → 放行
3. 拆词：按 `_` 断成 `recieve`（偏移 4）、`payload`（偏移 12）
4. `recieve` 整词命中 → `receive`
5. 输出 `5:4 recieve -> receive`，插入符 7 个

`COMMIT = "9ba8df137936"`：

1. 抓标识符：`COMMIT`、`9ba8df137936`
2. 第二个数字占比 83% → 判废，整个跳过，连拆都不拆
3. `COMMIT` 不在表里，无输出

## 五、和原版差在哪

| 维度 | typos | spellcode |
|---|---|---|
| 实现 | Rust，10 个 crate | Python，单文件，零依赖 |
| 词典 | varcon / codespell / misspell / Wikipedia 生成 | 手工 435 条，只收无歧义的 |
| 配置 | `typos.toml`，可表达正则、glob、文件类型、路径继承 | 无配置文件 |
| 忽略单行 | 写 `extend-ignore-re` 正则 | `# spell:ignore` |
| 哈希误报 | 有（#415、#1363 至今 open） | 判废层拦掉；两字母条目不存在 |
| 复数修正 | `DELETEs` 可能变 `DELETEEs`（#745） | 前缀保护，不报 |
| 修正大小写 | 跟随原词 | 跟随原词（`RECIEVE -> RECEIVE`） |
| 速度 | Rust，百万行级秒级 | 3.8 万行/秒，40 万行 10.5 秒 |
| 编辑器集成 | LSP、VSCode、pre-commit、GitHub Action | 只有 CLI 和 JSON |
| 覆盖率 | 几万条 | 435 条，冷僻错拼一律放过 |

最后一列不是谦虚，是取舍：**覆盖率是它的强项，精确率是我的强项**。一个 435 条的表，在标准库 40 万行上报了 9 处，9 处全是真的；换成一个几万条的表，能多抓十几处错拼，但也会多出几百条「这是什么鬼」。

## 六、怎么用

```bash
git clone https://github.com/z15346183030/spellcode
cd spellcode
python spellcode.py examples
```

真实输出（Windows 终端，本机跑出来的）：

```
examples\demo.py:9:5  recieve -> receive
    def recieve_payload(raw, timeout=30):
        ^^^^^^^
examples\demo.py:10:15  seperate -> separate
        """从上游取数据。seperate 这里故意拼错。"""
                         ^^^^^^^^

2 处可疑拼写，分布在 1 个文件；共检查 1 个文件、37 行，用时 0.00s
确认无误的可以加一行注释 spell:ignore 跳过。
```

同一份 `examples/demo.py` 里还埋了：版本号 `v2.1.3`、哈希 `9ba8df137936`、随机 ID `FhDomw6BaHU`、URL 里的错拼、`DELETEs` 这种复数、以及一行加了 `spell:ignore` 的——一个都没被报出来。

修掉：

```bash
python spellcode.py --fix examples
```

```
examples\demo.py  已修正 2 处

修正 2 处，涉及 1 个文件；共检查 1 个文件、37 行，用时 0.01s
```

修完再查一次：

```
没有发现问题：检查 1 个文件、37 行，用时 0.00s
```

进 CI：

```bash
python spellcode.py --quiet src || { echo "有拼写问题"; exit 1; }
```

退出码 `0` 干净、`2` 有发现、`1` 路径或参数错误。

跑在自己身上（本仓库不含示例，加索引仓库）：

```
没有发现问题：检查 16 个文件、3420 行，用时 0.04s
```

跑在 CPython 标准库上（含 site-packages，1000 个文件）：

```
9 处可疑拼写，分布在 6 个文件；共检查 1000 个文件、404927 行，用时 10.52s
```

9 处分别是：`seperator` ×4（pygments）、`prefered`（resolvelib）、`occuring`（zipimport 注释）、`minumum`（`_pyrepl`）、`ommit`（uuid.py 注释）、`compatability`（`_aix_support.py` 里一条 IBM 文档链接）。**没有一处是哈希、版本号或缩写被误伤。**

## 七、开发中踩的坑

这次不算顺利，一共六个坑，五个是自己写出来的。

**坑一：`standard` 被建议改成 `standarddd`。**

第一次在真实仓库上跑，扫出两条莫名其妙的结果：

```
doc2md\README.md:3:146  standard -> standarddd
depmap\README.md:3:113  standard -> standarddd
```

`standard-library` 里那个正确的 `standard` 被剥掉 `d` 成了 `standar`，命中表里 `standar -> standard`，再把 `d` 拼回去，就成了 `standarddd`。

修法是前缀保护那一行。回归测试：`test_valid_word_that_is_prefix_of_typo_target`。

**坑二：`seperators` 被改成 `separatorss`。**

改完查词逻辑，`lookup` 返回的是整词建议（`separators`），但 `check_line` 里还留着一句「把后缀补回去」——后缀已经被 `lookup` 拼进建议里了，于是补了两次。

修法是把 span 从「只覆盖词干」改成「覆盖整个词」，建议直接用整词替换。这个改动顺手把 `fix_text` 里的偏移计算也简化了一半。回归测试：`test_span_covers_whole_word`、`test_fix_keeps_plural`。

**坑三：标准库上冒出 27 条 `TEH -> THE`。**

这个坑测试抓不到，只有扫 40 万行真实代码才会露头。27 条全部来自同一类东西：

```
Lib\encodings\cp1256.py:250  TEH -> THE
    '\u062a'   #  0xCA -> ARABIC LETTER TEH
```

`TEH` 是阿拉伯字母名，不是 `the` 的错拼。还有一条来自 certifi 的 `cacert.pem`——一段 base64 里的 `...tTEH6KRt...`。

修法：全大写且长度 ≤ 3 的词，一律当缩写跳过。`TEH` 从此不查，`teh` 小写照样查。回归测试：`test_three_letter_acronym_is_skipped`、`test_acronym_inside_base64_is_skipped`。

**坑四：`spell:ignore=foo` 把整行都放行了。**

指令是这么写的：

```python
DIRECTIVE_LINE = re.compile(r"spell:ignore\b")
```

`spell:ignore=recieve` 也会命中 `spell:ignore\b`（`=` 是非单词字符，`\b` 成立），于是整行被跳过，而不是只放行 `recieve`。

修法是加个负向先行断言，把它跟「整行放行」区分开：

```python
DIRECTIVE_LINE = re.compile(r"spell:ignore\b(?![=-])")
```

`-` 排除 `spell:ignore-file` 和 `spell:ignore-next-line`，`=` 排除按词放行。回归测试：`test_ignore_specific_words_leaves_others`。

**坑五：中文注释里的插入符对不齐。**

```
        """从上游取数据。seperate 这里故意拼错。"""
```

第一版按「列号 - 1」补空格，中文占两格，插入符整体左移。修法是按终端显示宽度算：`unicodedata.east_asian_width` 返回 `W` 或 `F` 的算两格，制表符算四格。

**坑六：40 万行要跑 24 秒。**

拿 cProfile 看，热点很清楚：`is_opaque` 1.1 秒（8.7 万次调用）、`split_identifier` 0.73 秒、`lookup` 0.72 秒。两处改完降到 10.5 秒：

1. `is_opaque` 里绝大多数标识符是纯字母，先走 `ident.isalpha()` 这条路，省掉 `sum(1 for c in ident if c.isdigit())` 的开销。
2. `lookup` 里原来对每个词试 9 个后缀（`endswith` 被调用 87 万次）。改成按后缀末字母建索引，只试可能命中的那几个。

外加一个快路径：纯小写或纯大写的单个标识符不拆，直接查表。

最终 404,927 行 / 10.52 秒 ≈ 3.8 万行/秒。单核、纯 Python、单文件。

## 八、一句话总结

拼写检查器的难点不是「找出错拼」，是**忍住不要报**——一个 435 条词条、宁可漏报的实现，在 40 万行真实代码上报了 9 处、9 处全对，这比一个几万条词条、每次运行要你翻二十屏红字的工具，更有可能活过第二周。
