# leakcheck · 2026-10-28

## 一、我拆的是哪个项目

[gitleaks/gitleaks](https://github.com/gitleaks/gitleaks)

- **星标**：30,410
- **许可证**：MIT
- **语言**：Go
- **最近提交**：2026-10-23
- **一句话定位**：把「这个仓库里有没有硬编码的密钥」变成一个能塞进 CI、有退出码的命令。

为什么选它，有三个理由。

第一，需求被验证得足够彻底。3 万星，GitLab、GitHub Action、pre-commit 生态里到处都是它，
说明「提交前拦住密钥」这件事真的有人需要，不需要我再论证一遍。

第二，它有一个非常干净的「可精简」切口。整个工具有 222 条规则、3,209 行 TOML 配置、
22 个命令行参数、5 种报告格式、4 级配置优先级、454 个源文件。而它做的事，
用一句话就能说完：扫文件，找像密钥的字符串。

第三，也是最关键的——它的 README 第一段现在写着：

> Gitleaks is feature complete. I'm not merging new features into Gitleaks.
> Future releases will be security patches only.

作者宣布功能冻结了，重心转向新项目。而它 issue 列表里 516 个 open issue 全都还在，
其中好几个是 2022 年开的、原因已经被人写在正文里的老问题。这不是批评——
一个项目能走到「功能完备」是成功——但它意味着这些痛点不会在上游被解决。

我要做的不是「更好的 gitleaks」。我要验证一个具体的猜想：
**密钥扫描的难点不是找出像密钥的东西，而是别在没密钥的地方喊狼来了。**
如果这个猜想对，那么一个只有 16 条形状规则的工具，应该能在误报为零的前提下抓到更多。

## 二、它怎么做的

### 核心链路

gitleaks 的入口有三个命令（`git` / `dir` / `stdin`），但它们最后都汇到同一个地方：
`detect/detect.go` 里的 `Detect(fragment)`。所谓 fragment，就是「一段文本 + 它来自哪个文件哪一行」。

```
文件 / git 补丁 / stdin
        │
        ▼
  ┌───────────────────┐
  │ Aho-Corasick 预筛 │  把所有规则的 keywords 建成一棵前缀树
  └───────────────────┘  文件里一个关键词都没有 → 直接跳过，一条正则都不跑
        │
        ▼
  ┌───────────────────┐
  │ 逐条规则跑正则     │  r.Regex.FindAllStringIndex(整个文件, -1)
  └───────────────────┘  222 条规则 × 整个文件
        │
        ▼
  ┌───────────────────┐
  │ 取 secretGroup     │  正则里第 N 个捕获组才是「密钥那一段」
  └───────────────────┘
        │
        ▼
  ┌───────────────────┐
  │ 熵阈值             │  130 条规则带 entropy，低于阈值丢掉
  └───────────────────┘
        │
        ▼
  ┌───────────────────┐
  │ allowlist 四连     │  path 正则 / 内容正则 / stopwords / commit 白名单
  └───────────────────┘  全局 allowlist 还有 26 条路径规则
        │
        ▼
  ┌───────────────────┐
  │ 行内 gitleaks:allow│  同一行里有这个字符串就放过
  └───────────────────┘
        │
        ▼
   Finding(ruleID, 文件, 行, 列, 密钥, 熵)  →  5 种报告格式之一
```

在这条主链路之外还有几个旁路：`detect/codec/` 会把 base64 / hex / percent / unicode 解码后再跑一遍
（`--max-decode-depth` 控制层数）；`required` 规则要求两个命中在若干行内同时出现才算数；
`baseline` 和 `.gitleaksignore` 用指纹记住「这些我看过了」。

### 用大白话说

它本质上是一个**关键词触发的正则匹配器**。

每个文件进来，先问一句「这里面有没有 `password`、`token`、`api` 这些词」。没有就走人。
有的话，222 条正则轮流上，每条都在整个文件上找「长得像我认识的某个厂商密钥」的片段。
找到之后，如果这条规则配了熵阈值，就再算一下这段字符串有多随机——太不随机就当成误报丢掉。
最后过一遍白名单。

这个设计有一个隐含的前提：**「像密钥」这件事可以被一条正则完整描述。**

绝大多数时候它是对的。`AKIA` 开头 20 位、`ghp_` 开头 40 位、`xoxb-` 开头一串数字，
这些格式确实一条正则就够。但正则描述不了另外三件事：

1. 这个值出现在什么**语法位置**上（`key: value`、`<tag>value</tag>`、`f(key="value")`）；
2. 这个值是不是**占位符**（`changeme`、`AKIAIOSFODNN7EXAMPLE`、`${DB_PASSWORD}`）；
3. 这个值是**引用**还是**值本身**（`os.getenv("X")` 和 `"hunter2"` 长得一样长）。

后面会看到，它漏报和误报的每一个具体案例，都能归到这三件事上。

### 它做对了什么

**关键词预筛。** `NewDetector` 里把 221 条规则的 keywords 建成 Aho-Corasick 前缀树。
这是纯正则方案能跑大仓库的唯一原因——先花 O(n) 判断值不值得花 O(n×规则数)。

**跳过第三方代码。** 全局 allowlist 里有 26 条路径规则：`node_modules`、`vendor/`、各种 lock 文件、
`*.min.js`、`site-packages`、`go.sum`。这些地方的东西 99% 是别人写的、你改不了的。
跳过它们不是偷懒，是让剩下的告警有意义。

**用指纹而不是行号记住「放行」。** `.gitleaksignore` 记的是 `commit:file:rule:line`。
行号会随代码改动漂移，指纹不会。

**会往下解码。** base64 藏一层密钥是最常见的伪混淆，这一层做了就比没做强。

**单二进制、零运行时依赖。** 一个安全工具如果需要先装一堆东西，它在 CI 里活不过第一个季度。

## 三、它哪里让人难受

### 痛点一：正则不知道值在什么语法位置上

issue [#908](https://github.com/gitleaks/gitleaks/issues/908)，2022 年 6 月开的，到现在还开着：

> If a generic API key is stored in an XML tag, the secret is not being detected.
> ... Modifying the REgex and adding `'<'` as one of the end characters of the secret is catching it.

提 issue 的人自己把原因写出来了。`generic-api-key` 这条正则的结尾是：

```
(?:[\x60'"\s;]|\\[nr]|$)
```

翻译过来就是：密钥后面必须紧跟反引号、引号、空白、分号，或者直接到行尾。
而 `<apiKey>xxx</apiKey>` 里，`xxx` 后面是 `<`——不在这个字符集里，于是整条匹配失败。

改一个字符类就能修。但四年没人改，因为这条正则是 222 条规则里被依赖最多的一条，
动它可能让别的规则多报。

同一个根因还有两个 issue：
[#1361](https://github.com/gitleaks/gitleaks/issues/1361)（Spring Boot 项目里的 `application.yaml` 扫不到密钥）、
[#1078](https://github.com/gitleaks/gitleaks/issues/1078)（用户为了扫 `.properties` 手写了一条 200 多字符的正则，还是不行）。

我在自己造的语料上完整复现了这件事。40 个真密钥文件，gitleaks 漏了 13 个，
XML 那个就是被这个字符类挡住的。

### 痛点二：一条告警只告诉你「这行有个东西」

issue [#1766](https://github.com/gitleaks/gitleaks/issues/1766)：

> Gitleaks findings report the line, match, and secret, however, this isn't always enough context to be useful.
> Some false-positives ... require _n_ lines of surrounding context for the allowlist to be effective.
> This is also required for `gitleaks:allow` to work with findings where `gitleaks:allow` cannot be place on the same line.

这条说得很准。输出是这样的：

```
Finding:     "export BUNDLE_ENTERPRISE__CONTRIBSYS__COM=cafebabe:deadbeef",
Secret:      cafebabe:deadbeef
RuleID:      sidekiq-secret
Entropy:     2.609850
File:        cmd/generate/config/rules/sidekiq.go
Line:        23
```

它告诉你「这一行里有个叫 cafebabe:deadbeef 的东西，命中 sidekiq-secret 规则，熵 2.61」。
它不告诉你「这是被赋给谁的」。而在真实项目里，判断一条告警真假的第一依据
就是「它被赋给了哪个变量」——`SESSION_SECRET` 和 `content_type` 是两回事。

没有这层信息，你只能去读规则；读不懂就只能加白名单；白名单是按正则写的，
写宽了误杀，写窄了不生效。这是一个闭环。

### 痛点三：示例密钥被反复误报，排除规则要每个项目写一遍

issue [#1864](https://github.com/gitleaks/gitleaks/issues/1864)：

> With the integration of gitleaks into gitlab and other tools its especially frustrating to have to add
> an exclusion to vendor dependancies each time I create a new project when using Go and things like the AWS SDK for Go.

AWS SDK 的测试数据里带着 `AKIAIOSFODNN7EXAMPLE`——AWS 官方文档第一页那个示例 key。
任何 Go 项目 `go mod vendor` 之后都有一份。工具认不出「这是文档示例」，
用户就只能每个新项目手动加一条排除。

这个例子特别有意思：一个找密钥的工具，被另一个找密钥的工具的测试数据误伤，
而解法是给每一个下游项目都写一遍排除。

我拿 gitleaks 扫我自己那 11 个仓库（103 个文件），它报了 4 条。
3 条是我演示目录里故意造的假密钥（真阳性），
第 4 条是 `tests/test_leakcheck.py` 里这行字符串常量：

```python
text = "-----BEGIN RSA PRIVATE KEY-----\n" + b64(48) + ...
```

只有头、没有身子的一段 base64，它照样当成私钥报出来。这就是误报的形状：
**正则匹配到了「像密钥的片段」，但那个片段其实是描述密钥的代码。**

### 痛点四：配置本身变成了一件需要维护的事

issue [#1557](https://github.com/gitleaks/gitleaks/issues/1557)：

> The `--config` flag doesn't support remote download so there is an expectation that the rules file is already
> present in the local dir... This requires a little more effort to make sure teammates are using the latest custom rules.

而配置的优先级有四级：`--config` → `GITLEAKS_CONFIG` → `GITLEAKS_CONFIG_TOML` → 目标目录的 `.gitleaks.toml`。
谁覆盖了谁，出问题的时候要查一阵。

再叠上 22 个命令行参数、5 种报告格式、3 个命令（还有 2 个已废弃但没删的 `detect` / `protect`），
一个「扫一下有没有密钥」的动作，前置知识比很多人想象的多。

## 四、我们做了什么

**定位一句话：不看正则看证据——形状、键名、上下文三样对上才算数，说得出凭什么。**

### 砍了什么 / 留了什么 / 加了什么

| | 内容 | 为什么 |
|---|---|---|
| **砍** | 222 条厂商规则 → 16 种 | 规则越多越难维护，而误报的大头恰恰来自那些为了「尽量全」写得最松的通用规则 |
| **砍** | TOML 配置 + 4 级优先级 | 一个必填项都没有的工具有资格谈零配置 |
| **砍** | git 历史扫描 | 工作区覆盖了 pre-commit 和 CI 两个主场景；历史扫描要处理重写和大仓库，塞进来核心逻辑会翻倍 |
| **砍** | 5 种报告格式 → 2 种 | 文本给人看，JSON 给机器看 |
| **砍** | 编码层解码、压缩包遍历、baseline、SARIF | 每层解码要重跑全部规则，代价是线性的 |
| **留** | 关键词预筛的思路 | 换成一条字面量门槛，效果一样，三行代码 |
| **留** | 熵作为一条证据 | 但不再当唯一门槛，也不再一刀切 3.5 |
| **留** | 输出打码 | 而且做得更彻底：连上下文那一行也替换掉 |
| **留** | 单文件、零依赖、有退出码 | 这是它能被塞进任何 CI 的前提 |
| **加** | 结构化取值（名字 + 值） | 不管值是写在 `=` 后面、`:` 后面还是 `<tag>` 里面 |
| **加** | 61 个占位词 + 模板语法 + 掩码 + 键盘串 | `AKIAIOSFODNN7EXAMPLE`、`changeme`、`${DB_PASSWORD}` 直接不报 |
| **加** | 引用识别 | `os.getenv(...)`、`DB_PASSWORD`、`/etc/ssl/server.key`、代码里的裸标识符 |
| **加** | 英文单词识别 | 标准库第一版扫出 56 条误报，全是 `token = "unstructured"` 这种 |
| **加** | 每条结论带证据和分数 | 一眼看出凭什么报 |
| **加** | `--paranoid` | 列出「被判不是」的候选和原因，回答「为什么这个你没报」 |

### 核心算法

一个候选值进来，走四步：

**第一步，硬性排除。** 任何一条命中就丢：长度 < 6、占位词、模板语法（`${}` / `{{}}` / `%()`）、
掩码（`****` / `xxxx`）、纯数字、键盘串（`qwertyuiop` / `abcdef`）、
名字在值里出现两次（`pwd` → `pwd#module-pwd`）、引用（环境变量名 / 点号路径 / 函数调用 / 文件路径）。

**第二步，形状匹配。** 16 种公开格式，命中就 +0.55。其中三种要过验证器：

- JWT 的头部必须能 base64 解出带 `alg` 的 JSON——三段点号谁都会写，头部解得开才算；
- URL 的密码位不能是占位词，也不能是纯数字（`redis://:password@host` 是文档）；
- 私钥必须真的有身子——头后面至少 40 个 base64 字符。只有头的那是模板。

**第三步，加权。**

| 证据 | 分值 |
|---|---|
| 形状命中 | +0.55 |
| 强键名（`password` / `secret` / `token` / `api_key` / `access_key` …，整词匹配） | +0.30 |
| 弱键名（`key` / `sig` / `salt` / `hash`） | +0.10 |
| 熵 ≥ 3.0 / ≥ 2.0 | +0.25 / +0.10 |
| 长度 ≥ 16 / ≥ 8 | +0.10 / +0.05 |
| 密码类键名（人挑的密码又短又弱，靠名字撑） | +0.15 |

**第四步，阈值。** ≥ 0.65 高置信，≥ 0.45 报，< 0.45 压掉并记一笔原因。

举个具体的例子，`DB_PASSWORD=hunter2`：

```
name:DB_PASSWORD      +0.30    整词匹配到 password
entropy>=2.0          +0.10    2.81 bit
password-name         +0.15
                       0.55  → 报，medium
```

这个值长度只有 7、熵只有 2.81，按「随机串」的标准根本不够格。
但键名是 `DB_PASSWORD`，人挑的密码就是这个样子。**这就是为什么分数要加权而不是一刀切。**

换三个反例：

- `DB_PASSWORD=changeme` → 第一步出局（占位词）；
- `auth_header = "Authorization"` → 第一步出局（英文单词，长度 13、无数字无符号）；
- `self.passwd = password_mgr` → 第一步出局（值的右边是裸标识符，文件是 `.py`，判为引用）。

## 五、和原版差在哪

| 维度 | gitleaks 8.30.1 | leakcheck |
|---|---|---|
| 规则 | 222 条，3,209 行 TOML | 16 种形状 + 1 条通用路径，写死在代码里 |
| 判定依据 | 单条正则 + 熵阈值 + allowlist | 形状 / 键名 / 随机度 加权 |
| 配置 | 4 级优先级的 TOML | 无配置文件 |
| 命令行参数 | 22 个 | 6 个 |
| 报告格式 | json / csv / junit / sarif / 模板 | 文本 + JSON |
| 扫描范围 | 工作区 / git 历史 / stdin / 压缩包 / 编码层 | 工作区 / stdin |
| 占位符识别 | 靠 allowlist 正则 | 61 个词 + 模板 + 掩码 + 键盘串 + 名字自引用 |
| 结论可解释性 | RuleID + 熵值 | 每条证据及权重 |
| 代码量 | 454 个文件（132 个规则源文件，215 个测试样本） | 1 个文件 901 行 |
| 吞吐 | 11.7 MB/s（Go） | 2.2 MB/s（纯 Python） |

两个数字需要说清楚，不然这个对比不公平。

**召回 40/40 对 27/40，是因为语料是我造的。** 40 个正样本全部是「赋值式泄漏」——
`.env`、YAML、properties、XML、JSON、Terraform、元组。这恰好是 gitleaks 通用规则最弱的地方。
如果换成 40 个各家厂商特有的 token 格式，结论很可能反过来：它 222 条规则，我 16 条。
这份语料证明的是「在赋值式泄漏这个子集上，结构化取值 + 占位符识别更有效」，
不是「leakcheck 比 gitleaks 强」。

**速度慢五倍是语言差距。** 同一份 107 个文件的目录（1.35 MB），
gitleaks 168 毫秒，leakcheck 620 毫秒。Go 编译型 + 前缀树预筛，对上 Python 逐行多正则。
我没有为了追这个数字去优化算法，因为在这个量级上 0.6 秒和 0.17 秒都不影响任何人的工作流。

## 六、怎么用

```bash
git clone https://github.com/z15346183030/leakcheck.git
cd leakcheck
python examples/make_demo.py
python leakcheck.py examples/demo
```

真实输出（本机，Windows，Python 3.13）：

```
examples/demo/.env
  2:19   high   0.90  AWS access key id
         AKIA********6FM
         why  shape:aws_access_key +0.55 · entropy>=3.0 +0.25 · len>=16 +0.10
         >    AWS_ACCESS_KEY_ID=AKIA********6FM
  3:23   high   0.65  疑似密钥
         dktA********GPW
         why  name:AWS_SECRET_ACCESS_KEY +0.30 · entropy>=3.0 +0.25 · len>=16 +0.10
         >    AWS_SECRET_ACCESS_KEY=dktA********GPW
  4:13   high   0.65  疑似密钥
         dktA********6dk
         why  name:DB_PASSWORD +0.30 · entropy>=2.0 +0.10 · len>=16 +0.10 · password-name +0.15
         >    DB_PASSWORD=dktA********6dk

examples/demo/config.yaml
  4:15   high   0.80  疑似密钥
         dipu********nty
         why  name:password +0.30 · entropy>=3.0 +0.25 · len>=16 +0.10 · password-name +0.15
         >    password: dipu********nty

examples/demo/deploy.xml
  3:11   high   0.65  疑似密钥
         dnxG********epy
         why  name:apiKey +0.30 · entropy>=3.0 +0.25 · len>=16 +0.10
         >    <apiKey>dnxG********epy</apiKey>

examples/demo/settings.properties
  2:13   high   0.75  疑似密钥
         dqBN********nzL
         why  name:db.password +0.30 · entropy>=3.0 +0.25 · len>=8 +0.05 · password-name +0.15
         >    db.password=dqBN********nzL

6 leaks · 6 high  ·  6 files · 588 B · 0.28 s · 6 条候选被判为不是密钥（no-context 3、placeholder 3）
```

演示目录里有 6 个文件，报了 4 个。没报的两个是 `docs.md`（里面是 AWS 官方示例
`AKIAIOSFODNN7EXAMPLE`）和 `template.env`（`changeme` 加 `${API_TOKEN}`）。

想知道它们为什么被放过，加 `--paranoid`：

```
dismissed  6 条（证据不足，--paranoid 才列出）
  .../examples/demo/docs.md         3  placeholder    AKIA********PLE
  .../examples/demo/template.env    1  placeholder    ch******
  .../examples/demo/template.env    2  placeholder    ${AP********KEN
  .../examples/demo/.env            1  no-context     eu-w********t-1
  .../examples/demo/config.yaml     3  no-context     jdbc********app
  .../examples/demo/deploy.xml      2  no-context     http********/v1
```

这一屏是我最满意的部分。它把「工具想了什么」摊开给你看：
`eu-west-1` 被看了、被算过、被判成「没有上下文」；
`AKIAIOSFODNN7EXAMPLE` 被看了、被判成占位符。**沉默也是要有交代的。**

日常用法：

```bash
python leakcheck.py                  # 扫当前目录，在 git 仓库里就跟着 .gitignore 走
python leakcheck.py src/ .env        # 指定路径
python leakcheck.py --json           # 机器可读
python leakcheck.py --quiet          # 一行一条，给 CI 用
```

退出码：有发现返回 1，干净返回 0。当 pre-commit 钩子：

```bash
# .git/hooks/pre-commit
#!/bin/sh
python path/to/leakcheck.py --quiet .
```

## 七、开发中踩的坑

这次踩得比较实，六个坑，每个都有对应的回归测试。

**1. 88 个字符的 base64，和 sha512 校验值长得一模一样。**

Azure 存储密钥是 88 个 base64 字符、以 `==` 结尾。
`package-lock.json` 里的 `"integrity": "sha512-..."` 也是 88 个 base64 字符、以 `==` 结尾。
第一版在仿冒语料上直接报了一条 Azure 密钥，而那是个 npm 依赖的校验值。

修法是在正则开头加一个 `(?<![A-Za-z0-9+/\-])`——排除前面紧跟 `-` 的情况，
因为 `sha512-` 后面正好是 `-`。回归测试：`test_sha512_integrity_is_not_azure_key`。

**2. 英文单词的熵比真密钥还高。**

第一版拿标准库试跑，543 个文件报了 56 处。抽样一看全是这种东西：

```python
token = 'unstructured'
token = 'dot-atom-text'
```

`unstructured` 的香农熵是 3.65 bit，比 `nYHqlKJuU5WVWj`（3.66）只低一点点。
**熵根本分不开英文单词和随机串**——因为英文单词的字母分布也很均匀，只是分布在不同字母上。

真正把它们分开的是**字符类**：`unstructured` 只有小写一类，随机密钥至少混两类（大小写、数字、符号）。
加上这条判据，56 处降到 11 处。回归测试：`test_english_word_is_dismissed`。

**3. 验证器只拿到了「匹配到的那一段」，拿不到上下文。**

私钥的规则只匹配 `-----BEGIN RSA PRIVATE KEY-----` 这个头，
而「有没有身子」这件事在头后面。第一版的验证器签名是 `check(matched)`，
于是所有私钥都被判成「只有头」，全部漏掉。

改成 `check(matched, full_text)`，私钥的验证器看全文，JWT 和 URL 的验证器看匹配段。
回归测试：`test_pem_with_body`。

**4. 自己的报告把密钥又泄露了一遍。**

`--json` 输出里有个 `line_text` 字段，本意是「给上下文」。
第一版直接放了原始行，于是 `DB_PASSWORD=dktAHQX6dktAHQX6` 原封不动躺在 JSON 里——
一个专门用来找密钥的工具，把密钥写进了自己的报告，而这份报告会被贴进 issue。

修法是在 `redacted_text()` 里把值替换成掩码，文本和 JSON 两条输出路径都走它。
回归测试：`test_json_output`（断言完整密钥不出现在输出里）。

**5. `"" in "\"'"` 是 True。**

`clean_value` 里我写的是 `if v[:1] in "\"'`":`。看着没问题，
但空字符串是任何字符串的子串，所以 `"" in "\"'"` 返回 True，
下一行 `v.find(v[0], 1)` 就抛 `IndexError: string index out of range`。

这个 bug 没被单元测试抓到，是拿标准库试跑时崩在半路才发现的——
54 万个候选里总有一个值是空的。修完补了 `test_empty`。

**6. 代码里的裸标识符不是字面量。**

`self.passwd = password_mgr` 这一行，键名 `passwd` 是强证据，值 `password_mgr` 长度 12、有下划线、
熵 3.4，全部达标，于是被报了出来。但右边那个是变量名，不是密码。

修法是给 `Candidate` 加一个 `quoted` 字段：值是不是被引号包着的。
在代码文件（`.py` / `.js` / `.go` / …）里，没有引号的裸标识符判为引用。
配置文件（`.env` / `.ini` / `.properties`）不受影响，因为那里的值本来就不带引号。
回归测试：`test_bare_identifier_in_code_is_reference`。

## 八、一句话总结

**一个扫描器最该学会的不是「找出更多」，是「忍住不喊」——
把 222 条正则换成了三条证据，代价是少认 206 家厂商，换来的是 40 个真密钥一个不漏、
40 个仿冒品一个不报，以及每条结论都说得出凭什么。**
