# yamlsplice · 2026-10-26

## 一、我拆的是哪个项目

[mikefarah/yq](https://github.com/mikefarah/yq)

- 星标：16,186
- 许可证：MIT
- 语言：Go
- 最近提交：2026-10-22（四天前，还在活跃开发）
- 一句话定位：一条命令处理七种配置格式（YAML / JSON / XML / CSV / TOML / HCL / properties），用 jq 风格的表达式查询和改写它们。

**为什么选它。**

三条硬门槛它都过得很干净：星标一万六，MIT，四天前还有提交。但真正让我决定做它的，是它 issue 列表的形状。

翻它按评论数排序的 open issue，前五名里有三个是同一件事：

- #515「写回时把空行删了」，42 条评论，2020 年开的，到现在还开着；
- #462「列表项的缩进被强行改掉」，18 条评论，2020 年开的，还开着；
- #465「原地写会改注释前的空格」，12 条评论，2020 年开的，还开着。

一个六年的项目、一万六千个星、作者一直在维护，却有一类问题跨五年解决不掉。这种情况通常不是作者懒，而是**根因在架构上**——沿着现在的路走，修不好。

对每天找题目的人来说，这就是最好的信号：不是「这个工具有个 bug 我去修」，而是「这条路走到头了，换条路能不能绕过去」。

## 二、它怎么做的

yq 的核心链路是这样的：

```
  file.yaml
      │
      ▼
  ┌─────────┐   读成节点树。注释被拆出来挂在节点上
  │  解析器  │   （headComment / lineComment / footComment），
  └─────────┘   缩进宽度、引号风格、flow 还是 block——不进树
      │
      ▼
   节点树  ──►  表达式求值（jq 风格：.a.b | select(...) | .c = 3）
      │
      ▼
   节点树被改写
      │
      ▼
  ┌─────────┐   按自己的规则重新排版。缩进用几格、
  │ 序列化器 │   `- ` 放哪、什么时候加引号、什么时候用 | 块，
  └─────────┘   全在这一步决定
      │
      ▼
  新文本（或者 -i 写回文件）
```

看清楚这条链就明白问题在哪了：**「改一个值」在 yq 里等于「把整棵树重新写一遍」**。大部分节点写出来跟原来一样，所以你感觉不到。但只要有任何一个节点的「写法」和序列化器的默认规则不一致，它就会被改。

哪些东西不在树里？——缩进宽度、列表项 `- ` 的位置、引号风格、flow 集合还是块集合、空行的位置。这些恰恰是人和工具审美分歧最大的地方。

举个具体的：你写

```yaml
this:
- a
- b
```

YAML 规范允许 `- a` 和 `this:` 齐平，这是很多手写配置的风格（#462 里的原始输入）。解析成树之后，树里只有「this 是一个包含两个元素的数组」，**没有「短横线在第 0 列」这个信息**。序列化的时候，工具按默认规则缩进两格，于是变成 `  - a`。语义没变，字节全变了。

我们的做法是把这条链砍短：

```
  file.yaml
      │
      ▼
  ┌─────────┐   不构造对象，只产出一张表：
  │  扫描器  │   每个值在原文里从第几个字节到第几个字节
  └─────────┘
      │
      ▼
   位置表  { "spec.replicas" → (1204, 1205), ... }
      │
      ▼
   只替换目标那一段，其余字节原样拼回去
      │
      ▼
  新文本
```

没有「重新排版」这一步，所以也没有「排版规则」可以违反。

代价是扫描器得自己判断每个值的边界在哪——哪儿是值的开头，哪儿是结尾，注释算不算，空行算不算，块标量的内容到哪一行结束。这活儿比听起来麻烦，这个工具全部的复杂度都在这儿。

## 三、它哪里让人难受

**#515 —— 空行被删掉，42 条评论，五年没解决**

> 输入：
> ```yaml
> foo:
>   bar: 1
>
>   baz: 2
> ```
> 跑 `yq w - foo.baz 3`，输出：
> ```yaml
> foo:
>   bar: 1
>   baz: 3
> ```
> "Keep my extra blank line (it's better for readability / produces less of a diff)"
>
> —— [mikefarah/yq#515](https://github.com/mikefarah/yq/issues/515)

提需求的人自己把理由说得比我还清楚：**空行不是为了好看，是为了 diff 小**。

这句话值得停一下。在一个 800 行的配置里，空行是分段用的。删掉之后人还能读，但 git 会认为你改了十几行。评审的人得一行行确认「这行真的只是空行被删了，不是漏了什么」。42 条评论堆在这下面，说明踩到的人一抓一把。

**#814 —— emoji 键名被写坏**

> "The way yq v4 handles emoji is odd, inconsistent, unpredictable (which did not occur on earlier yq 2x versions which had other limitations). yq should (imho) pass utf8/emoji through unmolested."
>
> 用户给的例子是一串这样的键：`"bash.🔨/init.10级.🥾.b00t.sh": ""`
>
> —— [mikefarah/yq#814](https://github.com/mikefarah/yq/issues/814)

键名里带 emoji 听起来像玩具场景，但只要你的配置用中文、符号、emoji 做命名空间，就会撞上。而且注意这位用户说的「v2 是好的，v4 坏了」——**这不是从来做不到，是重构之后丢了**。

这类问题最能说明架构决定命运：一个只读不写的工具永远不可能犯这个错，问题只出在「写回去」那一步。

**#462 —— 列表缩进被强制改掉**

> 输入 `this:` / `- a` / `- b`，输出变成 `this:` / `  - a` / `  - b`。
> "yq 3.3.1 is breaking an original indentation level for list items"
>
> —— [mikefarah/yq#462](https://github.com/mikefarah/yq/issues/462)

两种写法在 YAML 里完全等价。但 diff 不认等价，只认字节。一个 200 行的配置里如果有 20 个这样的列表，改一个值就是 40 行 diff。

**#2025 —— 平白多出一个 `|-`**

> `yq -i '.test.this = "bla: bla\ntwice: bla"' test.yaml` 之后，那行变成了：
> ```yaml
> test:
>   this: |-
>     bla: bla
>     twice: bla
> ```
> "Why did the `|-` get there and how can I remove it?"
>
> —— [mikefarah/yq#2025](https://github.com/mikefarah/yq/issues/2025)

这个最典型。用户要的是「把这个字符串的值换掉」，工具顺手把**值的写法**也改了。语义完全正确，但文件已经不是用户那份文件了。而用户还得反过来问「怎么去掉」——他没法去掉，那是序列化器的决定。

**#465 —— 注释前的空格被改**

> "`yq w -1` command removes blank lines and changes spacing before comments."
>
> —— [mikefarah/yq#465](https://github.com/mikefarah/yq/issues/465)

同一类问题还有 #566（块标量尾部空格丢失）、#1277（尾部空格导致整个值被改成引号形式）、#1007（多文档的 headComment 错位）。**光是这一族，我数出来至少七个 issue，跨五年。**

我特意确认过：这些不是「实现得不好」。作者回得很认真，有些还给了 workaround。是这条路本身有墙——要让 `yq -i` 做到零 diff，它得放弃节点树，改成在原文上做手术，那等于重写整个写回路径。

## 四、我们做了什么

**一句话定位：给我一个路径，我改那一个值，别动别的。**

### 砍了什么、留了什么、加了什么

| | 内容 | 为什么 |
|---|---|---|
| 砍掉 | 表达式语言（`select` / `map` / `reduce` / `|=`） | 这是另一个产品的核心。要这个能力的人应该用 yq，我做不到比它好 |
| 砍掉 | 七种格式里的六种 | 每个格式一套解析器，等于七个工具。YAML 的坑已经够挖一天了 |
| 砍掉 | `-o=json` / `-o=props` 这类导出转换 | 那是「导出」，不是「编辑」。`get --json` 只解决「让脚本读得懂」 |
| 砍掉 | merge / multiply / 通配路径 | 都是表达式语言的延伸，语言砍了它们自然没了 |
| 砍掉 | 配置文件 | 一个选项都不需要记，才配叫零配置 |
| 留下 | 点 + 下标的路径 | 这是 90% 用法的全部。`spec.template.spec.containers[0].image` 一眼能看懂 |
| 留下 | `-i` 原地写 / 默认打到 stdout | 前者是效率，后者是安全——先看一眼再决定 |
| 留下 | 退出码分级 0/1/2/3 | 能进脚本的前提 |
| 留下 | `--json` 读 | 脚本消费 YAML 最常见的那件事 |
| 加上 | **位置表解析** | 只记字节区间，不构造对象树 |
| 加上 | **风格继承** | 新值沿用旧值的引号/块标量写法 |
| 加上 | **会指路的报错** | 路径写错时列出同级键；键里带点就告诉你方括号怎么写 |
| 加上 | **CRLF / BOM / 末尾无换行 / 多文档共存** | 在「只替换字节」的流程里，这些是免费的 |

### 核心算法怎么工作的

拿这段配置举例（就是仓库里 `examples/deployment.yaml` 的开头）：

```yaml
# 值班信息
ops:
  owner: platform-team   # 谁负责
  oncall:
    - 15346183030
```

扫描完，我手上不是「一个 dict」，而是这张表：

```
ops        map     字节 12..70
  owner    scalar  字节 19..38    slot = (19, 38)
  oncall   seq     字节 44..70
    [0]    scalar  字节 50..61
```

`slot` 是父节点给的范围，意思是「这块内容可以整体替换掉」。`owner` 的 slot 是 `(19, 38)`，正好是 `platform-team` 这 13 个字节。

执行 `set ops.owner infra-team`：

1. 沿路径找到 `owner` 这个 scalar；
2. 取它的 slot `(19, 38)`；
3. 把原文的这 13 个字节换成 10 个字节的 `infra-team`；
4. 剩下的 68 个字节原样拼回去。

结果：

```yaml
# 值班信息
ops:
  owner: infra-team   # 谁负责
  oncall:
    - 15346183030
```

行尾那个 `# 谁负责` 还在原位。**不是因为「注释被保留了」，是因为注释根本不在替换范围内**——slot 的右边界卡在值的最后一个字节，注释在它右边 3 个字节开外。

这一步是整个设计的支点。想通它之后，后面所有「保真」需求都变成免费的：空行？没碰。缩进？没碰。emoji 键名？它在 key 的位置，压根不在 value 的 slot 里，碰不到。

### 几个不那么显然的判断

**块标量的右边界。** `s: |` 的内容跨多行，slot 得从冒号后面一直延伸到「最后一行非空内容的末尾」。注意是**非空**——中间的空行算块内容，紧跟块后面的空行是分隔符，不能算进去。这个边界一开始我写错了，见第七节。

**风格继承的顺序。** 写回去的值长什么样，按这个顺序判定：

```
旧值是双引号  → 新值也用双引号（\n \t " \\ 都转义）
旧值是单引号  → 新值也用单引号（内部 ' 变 ''）
旧值是 | 块   → 新值还是 | 块，缩进跟着父键 + 步长
旧值是 > 块   → 新值多行就改用 |（折行还原不回去，保内容优先）
旧值没引号    → 类型能认出来就写裸标量，认不出且不安全就加双引号
```

「不安全」的判定：以 `-?:,[]{}#&*!|>'"%@\`` 或空格开头、以空格结尾、含 `: `、含 ` #`、是 `true/false/null/yes/no/on/off/~`、或者长得像数字。

这里有个细节值得说：`nginx:1.27` **不算**不安全。冒号后面没有空格，在 YAML 里它就是普通字符串，不需要引号。很多工具在这一步会过度加引号，把 `image: nginx:1.27` 写成 `image: "nginx:1.27"`——又一个无意义的 diff。

**新建键插在哪。** 父映射的最后一条后面，缩进跟同级对齐，换行符用文件自己的（CRLF 文件就用 CRLF）。文件末尾没有换行的话，正好用插入时补的那个换行符收尾，不会把新行粘到旧行上。

**flow 集合怎么办。** `annotations: {}` 里加一个键，我选择就地插入变成 `{team: core}`，而不是展开成块。因为展开会改掉用户选的写法，这正是我们要避免的事。

## 五、和原版差在哪

| 维度 | yq | yamlsplice |
|---|---|---|
| 支持格式 | YAML / JSON / XML / CSV / TOML / HCL / properties | YAML |
| 查询能力 | jq 风格表达式，能 `select` / `map` / `reduce` / 管道 | 路径寻址，只到「那个值」 |
| 写回方式 | 解析成节点树，改完重新序列化 | 只替换值的字节区间，其余原样 |
| 改一个值的 diff | 视文件而定，同类方法实测平均 118.7 行 | **1 行（11 个真实文件全部如此）** |
| 注释 / 空行 | 尽量保留，至少 7 个相关 issue 未解决 | 从未被读入，不可能丢 |
| 列表缩进风格 | 会重排（#462） | 保留原样 |
| emoji 键名 | 有写坏的记录（#814） | 保留原样 |
| 多文档 | 支持，可用 `select(document_index)` | 只改第一份，其余原样保留 |
| 运行方式 | Go 静态二进制，约 10 MB | 单个 `.py`，48 KB，只用标准库 |
| 速度（285 KB 文件改一个值） | 未测（见下） | 11.8 ms |
| 学习成本 | 一门表达式语言 | 一个路径写法 |

**关于速度那一行，我得说清楚。** 这台机器到 github.com 的 HTTPS 通道不稳定，`curl` 拉 yq 的 Windows 二进制直接 `exit 56`（接收失败），换了几次也没下来。所以**没有 yq 的实测数字，我不编**。

表里「平均 118.7 行」那个数据来自 ruamel.yaml 和 PyYAML——它们是「解析再序列化」这条路上 Python 生态的两个标准做法。拿它们做参照是公平的（同样的思路、同样的架构），但它们不是 yq，yq 的具体表现请以它自己的 issue 为准。

不过有件事比 diff 行数更能说明差别：**ruamel.yaml 保住了 658 行注释，但改一个值仍然平均动 118.7 行。** 说明「保留注释」和「保留格式」是两件事。yq 的 issue 里那一大堆抱怨，大部分也不是在说注释丢了，是在说缩进、空行、引号、`|-` 这些**排版**变了。把注释存下来相对容易，把排版存下来就得放弃重新序列化。

## 六、怎么用

把 `yamlsplice.py` 拷到任何地方，不需要安装：

```bash
$ python yamlsplice.py --help
yamlsplice —— 按路径读写 YAML，不动其他字节
...
```

读一个嵌套值：

```bash
$ python yamlsplice.py get spec.template.spec.containers[0].image examples/deployment.yaml
nginx:1.25
```

改一个值，看 diff：

```bash
$ python yamlsplice.py set spec.template.spec.containers[0].image nginx:1.27 examples/deployment.yaml > /tmp/new.yaml
$ diff examples/deployment.yaml /tmp/new.yaml
22c22
<           image: nginx:1.25
---
>           image: nginx:1.27
```

一行。上面那行 `# 演示用配置...` 还在，中间那些空行还在，`containers:` 下面 `- name: web` 用两格缩进这种写法也还在。

读块标量，折行规则算好了给你：

```bash
$ python yamlsplice.py get ops.runbook examples/deployment.yaml
1. 先看 kubectl get pods
2. 再看 kubectl logs
```

删一整块：

```bash
$ python yamlsplice.py del spec.template.spec.containers[0].resources examples/deployment.yaml
```

给脚本读：

```bash
$ python yamlsplice.py get spec --json examples/deployment.yaml
{
  "replicas": 3,
  "selector": {
    "matchLabels": {
      "app": "demo"
    }
  },
  "template": {
    "metadata": {
      "annotations": {
        "prometheus.io/scrape": "true"
      }
    },
    ...
```

路径写错的时候，它不会只说「找不到」：

```bash
$ python yamlsplice.py get spec.template.metadata.annotations.prometheus.io/scrape examples/deployment.yaml
yamlsplice: 这一层没有键 'prometheus'。有的键是：prometheus.io/scrape
提示：名字里带点或方括号的键要用方括号包起来，例如 ["prometheus.io/scrape"]
```

按提示写就能读到：

```bash
$ python yamlsplice.py get 'spec.template.metadata.annotations["prometheus.io/scrape"]' examples/deployment.yaml
true
```

### 实测数字

拿 11 个真实文件（GitHub Actions 工作流、docker compose、istio bookinfo、helm 的 1270 行 values.yaml、traefik 的 5311 行 CRD），每个文件改第一个标量的值，数改了行数：

| 做法 | 平均改动行数 | 11 个文件里只改 1 行的 |
|---|---|---|
| **yamlsplice** | **1.0** | **11 / 11** |
| ruamel.yaml（round-trip 模式） | 118.7 | 1 / 11 |
| PyYAML（读完再 dump） | 333.5 | 1 / 11 |

注释：语料一共 658 行注释。yamlsplice 剩 658 行，ruamel 剩 658 行，**PyYAML 剩 1 行**。

空行更逗：语料一共 204 行空行，PyYAML 输出之后变成 **706 行**——它把 657 行注释删了，又自己加了 502 个空行。

速度（同一台机器，Windows，Python 3.13）：

| 文件 | yamlsplice 改一个值 | ruamel 读+写 | PyYAML 读+写 |
|---|---|---|---|
| traefik CRD，5,311 行 / 285 KB | 11.8 ms | 911 ms | 559 ms |
| 合成 compose，18,002 行 / 319 KB | 112.7 ms | 3,222 ms | 1,514 ms |

它快不是因为代码写得巧，是因为**它不需要构造那棵树**。扫一遍，记下位置，扔掉。

另外做了一轮「改完复核」：在 11 个真实文件上挑了 286 个不同的路径，每个都改一次，然后用 PyYAML 把改前改后都读一遍，逐条对比展开后的结构，确认**只有目标路径那一个值变了，别处一个都没有**。286 次里 285 次完全干净，剩下 1 次是 PyYAML 自己的怪癖（把 GitHub Actions 的键名 `on:` 读成布尔 `True`），跟改动无关——顺带说一句，我们这个工具把 `on` 当字符串，对工作流文件才是对的。

## 七、开发中踩的坑

**1. 块标量的右边界把后面的空行吃掉了。**

最开始的写法是：扫描块内容时，遇到空行就更新「块结束位置」，因为空行理论上也是块的一部分（`|+` 的 keep chomping 会保留它）。结果在一个 fixture 上翻车：

```yaml
script: |
  echo one
  echo two

folded: >-
```

替换 `script` 的时候，那个空行被算进了替换范围，输出变成 `script: |\n  ...\necho two\nfolded: >-`——空行没了，两段粘在一起。

发现方式不是靠肉眼：我写了个脚本，把文件里**每个**标量都设成它原本的值，然后比对输出和输入是不是逐字节相同。15 个标量里 10 个不一致，一眼就看到是这个块的问题。

修法：空行只推进扫描位置，不更新结束位置。也就是说「空行可能是块内容，但如果后面没有更多块内容，它就不是」。回归测试：`test_block_scalar_does_not_swallow_the_blank_line_after_it`。

**2. `"" in "[{"` 在 Python 里是 `True`。**

类型推断里我写了这么一句：

```python
if s[:1] in "[{" and s[-1:] in "]}":
```

判断「用户输入的是不是 flow 集合」。跑测试的时候报 `IndexError: string index out of range`——因为空字符串是任何字符串的子串，`"" in "[{"` 返回 `True`，于是空串被当成 flow 集合送进了括号配对函数。

用户怎么触发？`set a ""`，把一个值设成空字符串。一点都不罕见。

修法：改成 `s[:1] in ("[", "{")`。回归测试：`test_empty_string_value`。

**3. `get` 子树丢了缩进，`get .` 丢了首行注释。**

`get metadata` 输出的是 `name: demo` 而不是 `  name: demo`——因为节点的起始位置记的是第一个键的字符，不是那一行的行首。`get .` 更明显，文件开头那行 `# 演示用配置` 直接没了，因为根节点从第一个键开始算。

这个坑是我自己敲命令看出来的，测试没覆盖到——测试里我用 `assertEqual` 比对的是「值」，不是「缩进」。修法：打印块级子树时，从 `node.start` 所在那一行的**行首**开始切。回归测试：`test_subtree_keeps_indentation`、`test_root_path_returns_whole_document`。

教训是：**输出给人看的东西，测试得断言整段文本，不能只断言值。**

**4. 多行裸标量读出来带着原始换行。**

YAML 允许裸标量折行续写：

```yaml
k: first part
  second part
```

读出来应该是 `first part second part`（折行规则：并成一行），我一开始直接返回了原文 `first part\n  second part`。修法是加一个 `fold_plain`，按规范把续行折起来，空行才变成真换行。回归测试：`test_multiline_plain_scalar_is_read_as_one_value`。

**5. 带点的键寻址不了——不是 bug，是可用性问题。**

跑 286 次语料复核的时候，有一个路径一直失败：`metadata.annotations.controller-gen.kubebuilder.io/version`。键名本身带点，被路径语法按点切碎了。

这不是解析器的错，是使用者的错——但**使用者不该被罚站**。所以我在「键找不到」的报错里加了一句：如果这一层有带点的键，直接把方括号写法打出来。回归测试：`test_dotted_key_gets_a_bracket_hint`。

**6. 18,000 行要 164 毫秒，慢在切行上。**

第一版跑 18k 行的合成文件要 164 ms，profile 一看，`split_lines` 那个逐字符循环占了 62 ms——我在里面一个字符一个字符地找 `\r` 和 `\n`。

改成 `re.finditer(r"\r\n|\r|\n")` 一次扫完，再把「这行是不是空行/注释」提前算好存在 `Line` 上（原来每次 `skip_trivia` 都要 `strip()` 一次，18k 行触发了十万次）。两处改完，164 ms → 108.7 ms。

**7. 环境坑：yq 的二进制下载不下来。**

想拿 yq 做横向对比，`curl` 拉 release 二进制直接 `exit 56`（接收失败），重试也不行。所以对比表里用的是 ruamel.yaml 和 PyYAML，并且在报告和 README 里都写明了「这不是 yq 的实测数字」。宁可数字少一行，也不能拿别的东西冒充。

## 八、一句话总结

yq 的 issue 列表里躺着七个跨五年没解决的抱怨，全都指向同一件事——**它必须把你的文件重写一遍才能改一个值**；我把那一步砍了，于是那些抱怨在这个工具里连发生的机会都没有。
