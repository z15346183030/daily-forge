# commitlog · 2026-10-25

## 一、我拆的是哪个项目

[orhun/git-cliff](https://github.com/orhun/git-cliff)，12,335 星，Apache-2.0，Rust 写的。

一句话定位：**从 git 历史自动生成 CHANGELOG，并按 Conventional Commits 算出下一个版本号。**

选它有三个理由。

第一，需求是真的。几乎每个要发版的项目都得写 CHANGELOG，而手写 CHANGELOG 是那种「不做不行、做起来又毫无乐趣」的活。12k 星说明这个需求被验证过很多次。

第二，它的复杂度是**自己长出来的**，不是被需求逼出来的。这一点很关键——很多工具之所以重，是因为它解决的问题本来就重（比如浏览器、数据库）。git-cliff 解决的是「把提交按类型分个组、打印成 Markdown」，这件事的天然复杂度大概是一百行代码。它做到了 1.5 万行，多出来的部分几乎全在「可配置」上：Tera 模板、五个远程平台的 API 集成、贡献者统计、多格式输出。

第三，issue 里的抱怨方向高度一致。不是「功能不够」，是「配置太绕」「模板报错看不懂」「标签名被削坏了」。这正好是「砍掉 80% 功能」能解决的问题。

顺带一提，它的默认配置里有一条 `replace_command = 'typos --write-changes -'`——默认配置会去调外部二进制检查提交信息的拼写。我们前两天刚做过 `typos` 的精简版（`spellcode`），看到这行的时候有点想笑：一个 CHANGELOG 生成器，默认配置依赖一个拼写检查器。

## 二、它怎么做的

先看规模。GitHub 的 tree API 里，它有 32 个 `.rs` 文件、570 KB 源码，按抽样测出来的平均行长（38 字节/行）换算，大概 1.5 万行。两个 crate：`git-cliff-core` 是库，`git-cliff` 是 CLI。

核心链路是这样的：

```
git 仓库
   │
   ├─ repo.rs      读提交、读标签、读远程信息（5 万字节）
   │
   ├─ commit.rs    解析提交：拆出 type / scope / breaking / 正文（5.6 万字节）
   │      │
   │      └─ 用 commit_parsers 里的一串正则，逐条比对提交信息，
   │         命中哪条就决定「归到哪组」或者「跳过」
   │
   ├─ release.rs   把提交按 tag 切成一个个版本（10.8 万字节）
   │      │
   │      └─ tag_pattern 决定哪些 tag 算版本
   │         skip_tags 决定哪些 tag 的提交并入下一个版本
   │
   ├─ template.rs  把版本 + 提交塞进 Tera 模板渲染（2.7 万字节）
   │
   └─ remote/*.rs  GitHub / GitLab / Gitea / Bitbucket / Azure 五个平台
          └─ 拿 PR 号、贡献者、头像、首次贡献标记
```

用大白话讲：**它把「怎么读 git」和「怎么打印」彻底分开了，中间夹了一层「配置」。**

读 git 那层是硬的，必须做对。打印那层被抽成了模板，用户可以随便改。中间那层配置，就是所有痛苦的来源——因为它既不是代码（不能调试），也不是数据（没有类型检查），而是一堆正则和模板语法的混合体。

`release.rs` 有 10.8 万字节，是最大的一个文件。为什么一个「按 tag 切段」的功能需要这么大？因为它要处理的东西比想象中多：tag 排序、tag 过滤、跨 tag 的提交归属、`topo_order` 拓扑序、版本区间渲染、`--latest`、`--unreleased`、`--strip`、预发布标签的处理……每一条都是某个用户提的需求，最后叠在一起。

这正是「功能臃肿」的典型形态：**没有一个功能是多余的，但它们加起来就没人能一眼看懂了。**

## 三、它哪里让人难受

### 1. 想藏掉六种提交，得写十五条正则

[#816](https://github.com/orhun/git-cliff/issues/816)，18 条评论。用户的目标只有一句话：只保留 feat 和 fix。他写了这些：

```toml
commit_parsers = [
  { message = "^feat", group = "<!-- 0 -->New Features" },
  { message = "^fix", group = "<!-- 1 -->Bugfixes" },
  { message = "^doc", skip = true },
  { message = "^perf", skip = true },
  { message = "^refactor", skip = true },
  { message = "^style", skip = true },
  { message = "^test", skip = true },
  { message = "^chore\\(release\\): prepare for", skip = true },
  { message = "^chore\\(deps.*\\)", skip = true },
  { message = "^chore\\(pr\\)", skip = true },
  { message = "^chore\\(pull\\)", skip = true },
  { message = "^chore|^ci", skip = true },
  { message = "build", skip = true },
  { body = ".*security", skip = true },
  { message = "^revert", skip = true },
]
```

十五行，没有一行包含业务信息，全是在复述「`feat` 是新功能、`chore` 不重要」——而这两件事，Conventional Commits 规范早就规定好了。

更糟的是 `{ message = "build", skip = true }` 这一条没有 `^` 锚定。它会匹配任何正文里带 `build` 字样的提交，包括 `fix: 修复 build 脚本`。用户大概率没意识到自己埋了个坑。

而且这不是用户写歪了。工具自带的默认配置 `config/cliff.toml` 就是 96 行、15 条 `commit_parsers` 的形状。**默认配置本身就在教用户「你得一条条列出来」。**

### 2. 默认模板会把标签名削坏

[#1291](https://github.com/orhun/git-cliff/issues/1291)，9 条评论：

> I use git tags of the form: `version/1.2.3` or `version/v2.4.6`. Recently when I run `git cliff > CHANGELOG.md` the word `version` becomes `ersion` in the `CHANGELOG.md` file.

输出长这样：

```
[ersion/1.0.16] - 2025-08-21
```

根因就在内置默认模板的第 12 行：

```jinja
## [{{ version | trim_start_matches(pat="v") }}] - {{ timestamp | date(format="%Y-%m-%d") }}
```

`trim_start_matches("v")` 的目的是把 `v1.2.3` 前面的 `v` 去掉。但它不检查这个 `v` 后面是不是跟着数字——`version/1.0.16` 的第一个字符也是 `v`，于是被一起削掉了。

这个 bug 活了很久，因为 99% 的项目用 `v` 前缀，碰不到。但碰到的时候，输出**看起来正常、实际错误**：`ersion/1.0.16` 不会报错，只会安静地出现在 CHANGELOG 里，直到有人发现标签链接全是 404。

我认为这不是手滑，是设计层面的问题：**版本号不该靠「削字符串」得到，该靠「认出数字」。** 前者对输入有隐含假设，后者没有。

### 3. 模板报错不告诉你哪里错了

[#785](https://github.com/orhun/git-cliff/issues/785)：

> Detailed template config errors out with template render error when running on falcosecurity/falco.
>
> **Expected behavior**: Changelog created without error, or the error to explain why the source data is invalid.

用户拿到的就是一句 `template render error`。是哪个变量？是自己的配置写错了，还是上游数据不对？不知道。

同类 issue 还有 [#791](https://github.com/orhun/git-cliff/issues/791)（提议换掉 Tera）和 [#659](https://github.com/orhun/git-cliff/issues/659)（不要在模板变量不存在时直接失败）。三个 issue 指向同一件事：模板层带来的表达力，已经被它的调试成本吃掉了。

还有一个细节很说明问题：**内置默认模板不输出提交链接。** 只有 `- 提交信息`，没有 `([a1b2c3d](url))`。想要链接，得在配置里写 Jinja 宏——项目自己用的 `cliff.toml` 里就有 `{%- macro remote_url() -%}` 和 `{% macro print_commit(commit) %}`。一个「生成 CHANGELOG」的工具，默认输出不带链接，是因为带链接就得写宏。

### 4. 两种格式 = 两个配置文件 + 跑两遍

[#572](https://github.com/orhun/git-cliff/issues/572)，12 条评论：

> A lot of the binary Rust projects I write use two changelog files, one in markdown format for GitHub/GitLab display and one in Debian format for cargo-deb Debian packages. […] It might be possible to call git-cliff twice with different config files but then some of the work is done twice.

配置里塞了输出格式，输出格式又和配置文件一一对应。这不是致命问题，但它是「配置驱动」架构的必然结果。

### 5. 预发布版本得自己算

[#1380](https://github.com/orhun/git-cliff/issues/1380)，37 条评论，是评论最多的 open issue：

> Today, git-cliff can compute and bump versions, but there's no explicit mechanism to mark the next version as a pre-release and increment the pre-release sequence.

`--bumped-version` 只会算 `1.3.0`，算不出 `1.3.0-rc.1`，也算不出「上一个是 rc.1，这次该 rc.2」。想走预发布流程的团队只能在外面用 shell 拼字符串——而这正是版本推算该负责的事。

## 四、我们做了什么

定位一句话：**默认值足够好，好到不需要写配置。**

### 砍了什么

| 砍掉 | 理由 |
|---|---|
| TOML 配置文件 | 类型到章节的映射是约定，不是偏好。让用户用正则复述规范，是在收「重复描述税」。 |
| Tera / Jinja 模板 | 模板的表达力换来的是不可读的报错。固定一种格式、把它打磨好，比给用户一个能写任何格式的语言更实用。 |
| 五个远程平台集成 | 要 token、要网络、要处理限流（[#422](https://github.com/orhun/git-cliff/issues/422) 就是 API 限流）。CHANGELOG 需要的信息全在本地 git 里。 |
| 贡献者统计 / 头像 / 首次贡献 | 属于「顺便做」，砍掉之后主路径短了一半。 |
| 多格式输出（Debian / JSON） | 一种格式做到能用，胜过五种都能凑合。 |
| 预处理器 / 后处理器 | 在提交信息上做正则改写，是把问题往后推一层。 |

### 留了什么

| 保留 | 理由 |
|---|---|
| Conventional Commits 解析（type / scope / `!` / `BREAKING CHANGE:`） | 这是整个工具的输入契约。 |
| 按 tag 分段 + 未发布段 | CHANGELOG 的骨架就是「每个版本一段」。 |
| 版本号推算 | 和 CHANGELOG 天然耦合，拆开反而要写更多胶水。 |
| 幂等写入 | 一个会重复写内容的生成器是不可用的。 |
| 提交链接 / PR 链接 | 从 `origin` 推断，零配置就有。 |

### 加了什么

| 新增 | 解决什么 |
|---|---|
| 标签名用正则**认出**末尾版本号，不做前缀裁剪 | #1291。`version/1.2.3`、`release-1.2.3`、`pkg-core@2.0.1` 都认；标签就叫 `version` 的直接忽略，不猜。 |
| 内置「该藏哪些类型」的默认值 | #816。零配置 = 十五条正则的效果。 |
| `--pre` 支持预发布序列 | #1380。`--bump --pre rc` 给 `1.3.0-rc.1`，再跑给 `1.3.0-rc.2`。 |
| 错误信息说人话 | #785。「不是 git 仓库」「区间里没有提交」「找不到 git 命令」，每条都带下一步怎么办。 |
| 解析不了的提交进「其他变更」 | 宁可多一行噪音，也不要让一次真实改动凭空消失。 |

### 核心算法怎么工作的

整条链路是四个纯函数，输入输出都是字符串，没有一个函数碰 IO：

```
git log --format=%H␟%h␟%an␟%ae␟%aI␟%s␟%b␞
        │
        ▼
   parse_log()      按 ␟ 切字段、按 ␞ 切记录 → [Commit]
        │
        ▼
   build_releases() 从新到旧走一遍，遇到身上有版本 tag 的提交就开新段
        │
        ▼
   render()         每个版本：破坏性变更置顶 → 按类型分章节 → 章节内按时间正序
        │
        ▼
   upsert_section() 按版本号找段落：有就替换，没有就插到最上面
```

**版本号认出来，不是削出来。** 用一个末尾锚定的正则：

```
(?<![0-9.])(\d+)\.(\d+)\.(\d+)(?:-([0-9A-Za-z.\-]+))?(?:\+[...])?$
```

`version/1.2.3` 从末尾往前找，`1.2.3` 匹配上了，前面是什么都不影响。`version` 单独一个词找不到三段数字，返回 `None`，当成普通标签忽略。`1.2.3.4` 也匹配不上——因为 `1.2.3` 后面还有 `.4`，`$` 不成立，而 `2.3.4` 前面是 `.`，被 lookbehind 挡掉。

**版本推算看的是「上一个 tag 之后的提交」。** 举个真实例子，演示仓库的当前状态：

```
$ python commitlog.py --bump
0.3.0
```

为什么是 `0.3.0` 而不是 `1.0.0`？因为上一次 tag 是 `v0.2.0`，之后有一条 `feat!: 返回结构从 list 改成 dict`。有破坏性变更 → 主版本 +1，但主版本是 0，0.x 还没定型，所以只进次版本：`0.2.0` → `0.3.0`。

同一时刻加 `--pre rc`：

```
$ python commitlog.py --bump --pre rc
0.3.0-rc.1
```

这条如果已经在 `0.3.0-rc.1` 上，再跑一次会给 `0.3.0-rc.2`——序列号自己会涨。

**幂等写入靠「按版本号认段落」。** 每次渲染出来的 `## [0.2.0] - 2026-09-12` 是一个键，文件里已有同键段落就整段替换，没有就插到最前面。所以重跑一百次结果都一样：

```
$ md5sum CHANGELOG.md && python commitlog.py --write CHANGELOG.md && md5sum CHANGELOG.md
4cbdba8211505d58f71fc283d0757ca5 *CHANGELOG.md
已写入 CHANGELOG.md（3 个版本段落）
4cbdba8211505d58f71fc283d0757ca5 *CHANGELOG.md
```

文件开头你手写的内容（项目简介、许可证说明）不会被碰——只有 `## ` 开头的段落归工具管。

## 五、和原版差在哪

| 维度 | git-cliff | commitlog |
|---|---|---|
| 配置 | 可选；内置默认配置 96 行、15 条 `commit_parsers` | 0 行，没有配置文件这个概念 |
| 输出格式 | Tera 模板，任意格式 | 固定 Markdown |
| 默认带提交链接 | 不带（要写 Jinja 宏） | 带，从 `origin` 推断 |
| 代码规模 | 32 个 Rust 文件，约 1.5 万行，两个 crate | 1 个 Python 文件，735 行 |
| 依赖 | Rust 工具链 + 十余个 crate | Python 3.9 标准库 |
| 远程集成 | GitHub / GitLab / Gitea / Bitbucket / Azure | 无 |
| 预发布版本 | 不支持（[#1380](https://github.com/orhun/git-cliff/issues/1380) 仍 open） | `--bump --pre rc` 直接给 |
| 标签名解析 | `tag_pattern = "v[0-9].*"` + 模板削 `v` | 正则认末尾版本号，任意前缀 |
| 写入 | `--prepend` 插入 | `--write` 按版本号替换或插入 |
| 非规范提交 | `filter_unconventional` 决定丢弃 | 进「其他变更」，`--strict` 才丢 |
| 运行开销 | Rust 二进制，毫秒级 | 12 条提交的仓库约 0.50 秒，几乎全在启动 git 子进程 |
| 库 API | 有（`git-cliff-core`） | 无，纯 CLI |

运行开销这一行得说清楚：0.50 秒不是算出来的，是量出来的——三次 git 子进程（`git log` 0.18 秒、`git for-each-ref` 0.16 秒、`git remote get-url` 0.16 秒），Python 侧的分组和渲染加起来不到 1 毫秒。在 Windows 上每次创建进程都要 0.16 秒，这钱省不掉。真要看计算性能，5 万条提交的基准是：解析 0.458 秒、分组 0.043 秒、渲染 0.120 秒，合计 0.620 秒。

## 六、怎么用

```bash
# 装：一个文件，复制走就行
curl -O https://raw.githubusercontent.com/z15346183030/commitlog/main/commitlog.py

# 在任意 git 仓库里
python commitlog.py
```

真实输出（演示仓库，两个 tag、九条提交）：

```
# Changelog

## 未发布

### 破坏性变更

- 返回结构从 list 改成 dict —— 字段 data 改名为 items

### 新功能

- 返回结构从 list 改成 dict ([a9f547a](https://github.com/o/r/commit/a9f547a3063d6f4c66907dc210f2fbbe61b668fb))

### 问题修复

- **api**: 分页越界返回空列表 ([#42](https://github.com/o/r/pull/42)) ([71b9d2c](https://github.com/o/r/commit/71b9d2cb749eb19159b8d3efd28573515a19ef1d))

## [0.2.0] - 2026-09-12

### 新功能

- **api**: 增加分页参数 ([d29225d](https://github.com/o/r/commit/d29225dedfff9e123e97a7652610c8e24a20d8b1))

### 问题修复

- 时间戳时区错乱 ([83b5338](https://github.com/o/r/commit/83b5338d53f456bfeeb39f109edc4055bdb59a39))

### 性能优化

- 导入速度提升 3 倍 ([2a0d844](https://github.com/o/r/commit/2a0d844a694935bb417bbf19f41507fe139cb8bd))
```

注意三件事：`docs: 补一段安装说明`、`test: 补 12 个用例`、`chore: 初始化项目` 都没出现（默认藏掉）；破坏性变更被单独提到最前面；`(#42)` 自动变成了 PR 链接。

放进 CI：

```bash
NEXT=$(python commitlog.py --bump)
git tag "v$NEXT"
python commitlog.py --write CHANGELOG.md
```

在真实的 daily-forge 仓库上跑（12 条提交，全部不是规范提交）：

```
$ python commitlog.py --all
# Changelog

## 未发布

### 其他变更

- 初始化仓库：README、MIT 许可证与实现规范 ([cad9cba](...))
- 统一换行符为 LF ([9051e3e](...))
- 新增工具 squeeze：压缩喂给模型的 JSON 与日志 ([20dff4a](...))
...
```

全部落进「其他变更」，一条都没丢。这就是「解析不了的提交不静默丢弃」的意义——如果按 git-cliff 的 `filter_unconventional` 默认行为，这十二条会全部消失，输出一个空的 CHANGELOG，而且不报错。

## 七、开发中踩的坑

**坑一：lookbehind 写太严，`v1.2.3` 全被挡掉。**

版本号正则一开始写的是 `(?<![0-9A-Za-z.])`，本意是防止 `1.2.3.4` 里的 `2.3.4` 被误认。结果 `v1.2.3` 的 `1` 前面是字母 `v`，也被挡掉了——一次跑出 14 个用例变红、2 个直接报错，报的还都是 `None != '1.2.3'` 这种看不出根因的错。

修法是把 lookbehind 收窄成 `(?<![0-9.])`：只拦数字和点，字母放行。`1.2.3.4` 依然拦得住，因为 `2.3.4` 前面是 `.`。回归测试 `test_v_prefix`、`test_highest_version`、`test_prerelease`。

**坑二：`--write` 把最老的版本顶到了最上面。**

`upsert_section` 的逻辑是「新段落插到最前面」，我按「最新 → 最旧」的顺序逐个写进去，结果每一次插入都把前一个顶下去，最后 `0.1.0` 跑到了文件第一段。

第一次跑完 `grep '^## '` 看到 `0.1.0` 排在最前面才反应过来。这个 bug 的阴险之处在于它不报错、输出也不难看——段落顺序反了，但每一段本身都是对的，不逐行比对根本看不出来。

修法是倒序遍历：先写最旧的，再写新的，新的自然落在上面。回归测试 `test_write_orders_newest_first`，断言三段的相对位置。

**坑三：预发布换标签时，版本号多进了一位。**

`1.3.0-beta.4` 上执行 `--bump --pre rc`，期望 `1.3.0-rc.1`，实际给了 `1.3.1-rc.1`。原因是我把「上一个版本」直接丢给了 `bump_core()`，而它看到的是 `1.3.0-beta.4`，按「有 fix → 修订号 +1」算出了 `1.3.1`。

逻辑上说不通：你还在准备 1.3.0，换个预发布标签不代表这个版本已经发出去了。修法是「上一个版本本身是预发布时，核心版本不动」。回归测试 `test_different_label_restarts_series`。

**坑四：空仓库直接被判成错误。**

`git log` 在刚 `git init`、一个提交都没有的仓库里退出码是 128，我的 `_run_git` 一律当失败抛异常，用户看到的是「git log 执行失败：fatal: your current branch 'main' does not have any commits yet」。

这不是错误，是「还没东西可写」。修法是识别这条特定信息返回空列表，让上层打印「这个区间里没有提交，没什么可记录的」。回归测试 `test_empty_repo`。

**坑五（差点踩）：版本段的日期该用哪个。**

一开始我以为该用 tag 的创建时间，写了个用例断言 `## [0.1.0] - 2026-10-01`，跑出来是 `2026-10-02`——因为那个 tag 打在第二条提交上，而这一版里最新的提交是 10-02。

想了一下，用例该改，代码不该改。CHANGELOG 里的日期是「这一版代码截止到哪天」，不是「你哪天想起来打 tag」。改了用例，并在 README 里写明这个规则。

## 八、一句话总结

git-cliff 把「怎么读 git」和「怎么打印」分得很干净，却在中间塞了一层配置，于是所有痛苦都堆在那里——commitlog 的做法是删掉那一层，因为 Conventional Commits 已经把类型规定好了，工具没有理由再问一遍。
