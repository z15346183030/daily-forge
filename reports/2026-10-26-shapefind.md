# shapefind · 2026-10-26

## 一、我拆的是哪个项目

[ast-grep/ast-grep](https://github.com/ast-grep/ast-grep)

- 星标：16,350
- 许可证：MIT
- 语言：Rust
- 一句话定位：用「写代码」的方式搜索代码——你写一段模式，它找所有结构相同的代码，不管空格、换行、注释怎么变。

为什么选它。理由有三个，按重要性排：

第一，它解决的是一个真问题。用 `grep` 找 `print(` 会被注释、字符串、`pprint` 全部命中；用正则找函数调用，嵌套括号一深就写不出来。按语法树匹配是唯一正确的解法，而这个解法在 16,350 个星标里被验证过了。

第二，它自己变复杂了。一个「让搜索变简单」的工具，仓库里 `schemas/` 目录下光 `rule.json`（描述它规则语言的 schema）就有 26,110 字节，`python_rule.json` 34,252 字节，`languages.json` 89,898 字节。这不是配置文件的体积问题，这是它已经长成了一门语言。

第三，痛点有据可查。它有一个叫 `strictness` 的旋钮，四个档位，而 issue 里有人明确说「我不知道该用哪个」，维护者的回应是「那再加一个配置项让你全局设」。这种「用配置解决配置问题」的循环，正是可以做减法的地方。

今天 GitHub Trending 上全是 AI Agent 和 skill 仓库——`cindy`、`hallmark`、`security-audit-skill` 之类。这些项目没有可复刻的核心算法，拆出来的东西只能是提示词，所以直接跳过，改用搜索 API 找真正的工具类项目。

## 二、它怎么做的

ast-grep 的核心不是「解析代码」，而是「让两棵树长得一样」。理解这一句，整个项目就通了。

它的代码分成十个 crate：`core`（匹配引擎）、`config`（规则加载）、`language`（tree-sitter 绑定）、`cli`、`lsp`、`napi`/`pyo3`/`wasm`（三种语言绑定）、`dynamic`（动态加载语言）、`outline`、`xtask`。真正干活的是 `core`，里面又分成 `matcher/`、`meta_var.rs`、`replacer/`、`match_tree/`、`ops.rs`、`pinned.rs`。

链路是这样的：

```
用户写的模式串（一段代码）
        │
        ▼
  按目标语言的 tree-sitter 语法解析
        │  ← 这一步是全部魔法所在
        ▼
  CST（具体语法树，带注释和空白）
        │
        ▼
  把 $A / $$$A 识别成「元变量节点」
        │
        ▼
  遍历目标文件的 CST，逐节点尝试 match_node()
        │
        ├── 命中 → 收集捕获 → 输出 / 重写
        └── 未命中 → 下一个节点
```

关键设计在于它用的是 **CST 而不是 AST**。tree-sitter 保留所有 token，包括注释和空白，好处是重写的时候能原样拼回去，坏处是——注释也进了树，于是注释能影响匹配。这就是后面那些 issue 的根源。

第二个关键设计是「模式就是代码」。用户写 `console.log($ARG)`，工具把它当作一段真实代码解析，解析出来的树里 `$ARG` 是一个标识符节点，匹配时特殊对待。这个设计让学习成本几乎为零，是它最成功的地方，也是我们唯一全盘保留的东西。

第三个是 `strictness`：`cst` / `smart` / `ast` / `relaxed` 四档，控制「哪些差异算不一样」。`cst` 最严格（连注释位置都比），`relaxed` 最宽松。这个旋钮存在的原因，就是 CST 保留了太多东西，必须让用户自己决定忽略多少。

## 三、它哪里让人难受

### 一、`strictness` 把语义决策推给了每一个用户

> "the default matching algorithm for patterns doesn't ignore comments, which is surprising for AST matching. If it cannot be changed now due to backward compatibility, it would be nice to have an option in config or CLI, so that we don't have to add `strictness` to every `pattern`."
>
> — [#2824 Configuration of the default strictness](https://github.com/ast-grep/ast-grep/issues/2824)

这段话值得逐句读。用户先陈述事实：**默认不忽略注释**。然后给出评价：**这对 AST 匹配来说很意外**。最后提出诉求：既然不能改默认值，那就给我一个全局开关，别让我在每条 pattern 上重复写。

注意维护者的选择：不改默认行为（向后兼容），加一个配置项。于是「注释算不算代码」这个问题，从「工具该替你决定」变成了「你每次都要决定一遍」。

这在我看来是设计上的错位。注释不是代码，这不是一个偏好问题，是一个事实问题。Python 的 `ast` 模块根本不保留注释——不是因为它偷懒，是因为注释在语法层面就不存在。一个需要用户反复声明「注释不算代码」的工具，等于把解析器的实现细节（用了 CST）泄漏成了用户的日常负担。

### 二、注释真的能改变匹配结果

> "[bug] Comments influence matching of AST node"
>
> — [#2212](https://github.com/ast-grep/ast-grep/issues/2212)

标题本身就是控诉。一个自称按语法树匹配的工具，被一段注释改变了匹配结果。这不是小概率边缘情况：`// TODO: 这里应该改成 func(a, b)` 这样的注释，恰好长得像一个调用，就会干扰匹配。

在 Python 里这个问题根本不存在——`ast.parse("f(a)  # f(b)")` 得到的树里没有任何注释痕迹。所以我们的实现天然免疫，不需要写一行代码去处理它。

### 三、片段不是合法程序时，得先学会 tree-sitter 的节点名

`return $X` 单独拿出来不是合法 Python；`except: pass` 也不是。ast-grep 的解法是让用户声明上下文：

```yaml
rule:
  pattern: $A
  inside:
    kind: function_definition
```

或者用 `context` + `selector` 指定「先按这段上下文解析，再从里面挑这个节点」。

问题是，`function_definition` 是 **tree-sitter 给 Python 函数定义起的节点名**。一个想搜 `return` 语句的 Python 程序员，被迫去查一个 C 库的语法节点表。这是把内部数据结构当成了用户接口。

真正需要的判断很简单：这段代码在哪几种上下文里是合法的？试一遍就知道了。`def` 里、`class` 里、`while True` 里、`try` 里，四次 `ast.parse` 的成本是微秒级，比让用户翻文档便宜得多。

### 四、重写之后缩进会错，而这是 Python

> "[bug] Python replacement indentation wrong in indented class"
>
> — [#1405](https://github.com/ast-grep/ast-grep/issues/1405)

> "[bug] Uneeded indentation is added during transform"
>
> — [#1935](https://github.com/ast-grep/ast-grep/issues/1935)

两个独立 issue，同一个症状：替换文本进入一个已有缩进的位置时，续行缩进不对——要么少缩进（语法错误），要么多缩进（逻辑跑到别的块里）。

对 C 或 JS，缩进错了只是难看。对 Python，缩进错了代码直接跑不起来，或者更糟：跑起来了但逻辑变了。所以「重写时缩进算错」对 Python 来说不是排版 bug，是**功能不可用**。

### 五、命令名和 semgrep 撞车

> "The semgrep program now includes a shortcut called 'sg' ... so it would be better to rename your 'sg' to 'ag'."
>
> — [#778](https://github.com/ast-grep/ast-grep/issues/778)

`sg` 这个名字被 semgrep 占了。同时装了 semgrep 和 ast-grep 的人，敲 `sg` 得靠 `which` 猜今天调用的是谁。工具本身没问题，但这类摩擦会累积。

## 四、我们做了什么

一句话定位：**一个 811 行的 Python 文件，按结构搜索和重写 Python 代码，零配置、零依赖、零旋钮。**

砍了什么、留了什么、加了什么：

| 动作 | 内容 | 理由 |
|---|---|---|
| 砍 | YAML 规则语言（`rule`/`kind`/`has`/`inside`/`not`/`all`/`any`） | 90% 的搜索一个模式就够，剩下 10% 用 `--check` 接管道更直接 |
| 砍 | `strictness` 四档旋钮 | 只保留一种语义：注释和空白永远不算代码。不给选择就不用解释 |
| 砍 | `context` + `selector` 手工声明 | 自动在四种上下文里重试解析 |
| 砍 | 20 多种语言 | 只做 Python，用标准库 `ast`，不引 tree-sitter |
| 砍 | 插件系统、LSP、三种语言绑定 | 命令行工具不必成为平台 |
| 留 | `$NAME` / `$$NAME` 元变量 | 原版最成功的设计，换掉只会更差 |
| 留 | 「模式就是代码」 | 这是整个工具存在的理由 |
| 留 | 搜索 + 重写双模式 | 找得到就得改得动 |
| 加 | 自动上下文重试 | `return $X`、`except: pass` 直接能写 |
| 加 | 标识符位置的元变量 | `def $F`、`class $C`、`self.$ATTR` 都能捕获 |
| 加 | 重写缩进按落点列对齐 | 修掉 #1405 / #1935 同类问题 |
| 加 | `--check` 退出码 | 一行命令当 CI 用，不写规则文件 |

### 核心算法

匹配是**结构同构判定**，不是文本比较。五步：

**第一步，模式预处理。** 把 `$X` 换成合法标识符 `__sf_s_X`，`$$X` 换成 `__sf_m_X`。为什么不能直接正则替换？因为模式里可能有字符串字面量包含 `$`——`re.match('$X', s)` 里的 `$X` 是字面量。所以用一个扫描器跟踪引号状态（单引号、双引号、三引号、转义、注释），只替换代码区域的 `$`。

**第二步，模式解析。** 先按模块解析。失败就依次在 `def` / `async def` / `class` / `while True` / `try` 里重试，每次把模式缩进四格塞进去。这就是「不用写 context」的全部秘密——猜五次比让用户查节点表便宜。解析成功后，如果顶层只有一个表达式语句，就取里面的表达式节点（这样 `foo()` 匹配的是 Call 而不是 Expr 包装）。

**第三步，递归比对。** 类型必须相同（`Call` 对 `Call`，`Name` 对 `Attribute` 不行），然后逐字段比。跳过两个字段：`ctx`（Load/Store 记账）和 `type_comment`（来自注释）。列表字段用带回溯的序列匹配——因为 `$$X` 可以吃掉 0 到 N 个元素，必须枚举所有切分点。

**第四步，捕获一致性。** 同名元变量第二次出现时，比较的是**结构指纹**（递归展开成元组，去掉 `ctx`）而不是源码文本。所以：

```python
a == a        # 匹配 $L == $L：两个捕获都是 Name('a')
b == c        # 不匹配：捕获值不同
x == (x)      # 匹配：括号在 AST 里根本不存在
```

这里有个坑我踩过：`True == 1` 在 Python 里是 `True`，但 `f(True)` 和 `f(1)` 结构上不是一回事。所以指纹里带上了类型名，`Constant` 比较时先比 `type(value)`。

**第五步，重写。** 捕获的源码片段按落点列重新缩进：先剥掉捕获自带的缩进，再按替换模板里的插入列补齐。然后从后往前替换，保证前面的偏移量不失效。

举个真实例子。这段代码，把 `$$BODY` 往一个已经缩进 4 格的位置塞：

```python
if something:
    class B(A):
        def replace(self, *args, **kwargs):
            return None
```

用 `class $C($$BASES): $$BODY` 重写成加一行 docstring 和一个类属性，结果是：

```python
if something:
    class B(A):
        """Generated wrapper."""
        batch_mode = True
        def replace(self, *args, **kwargs):
            return None
```

`return None` 保持了 12 格缩进——因为它被剥到相对 4 格，再按落点补 8 格。原版在这里出过两次 issue。

## 五、和原版差在哪

只列真正有差异的维度：

| 维度 | ast-grep | shapefind |
|---|---|---|
| 安装方式 | cargo / npm / pip / brew，二进制 10MB+ | 复制一个 811 行的 `.py` |
| 支持语言 | 20+，靠 tree-sitter | 1 种（Python），靠标准库 `ast` |
| 注释是否影响匹配 | 默认影响，要 `strictness` 调 | 永远不影响 |
| 片段模式 | 需 `context` + `selector` 或 `inside` | 自动重试五种上下文 |
| 规则复用 | YAML 规则 + `sgconfig.yml` | 无规则文件，模式即规则 |
| 重写缩进 | #1405 / #1935 报过错 | 按落点列对齐，有回归测试 |
| CI 检查 | `sg scan` + 一套规则 | `--check` 一个参数 |
| 配置文件数量 | 至少 2 个（项目配置 + 规则目录） | 0 |
| 扫描速度 | 更快（Rust + 增量解析） | 3.7 秒扫 386,762 行 |
| 语言覆盖面 | 全 | 只有 Python |

最后两行是诚实的代价。Rust 比 Python 快，这是事实；支持二十种语言比支持一种有用，这也是事实。shapefind 不是 ast-grep 的替代品，它是「我只写 Python，我只想要那 20% 功能」这个场景的答案。

## 六、怎么用

不需要安装。clone 下来就能跑。

```bash
git clone https://github.com/z15346183030/shapefind
cd shapefind
python shapefind.py 'print($X)' .
```

真实输出（在 shapefind 自己的仓库里搜）：

```
.\shapefind.py:769:17: print(json.dumps(format_json(path, source, item), ensure_ascii=False))
.\shapefind.py:772:17: print(format_text(path, source, item))
.\shapefind.py:779:13: print("%s:%d" % (path, count))
```

找裸 `except: pass`——`except` 单独不是合法模块，工具自己重试上下文，你不用声明：

```bash
python shapefind.py 'except: pass' src/
```

抓函数名、参数、函数体：

```bash
python shapefind.py 'def $F($$ARGS): $$BODY' --json .
```

```json
{"path": "demo.py", "line": 4, "column": 1,
 "text": "def f(a, b=2):\n    print(a)\n    return a + b",
 "captures": {"ARGS": "a, b", "BODY": "print(a)\nreturn a + b", "F": "f"}}
```

批量改调用，先看 diff：

```bash
python shapefind.py 'print($$ARGS)' --rewrite 'debug($$ARGS)' --diff .
```

```diff
--- app.py
+++ app.py
@@ -1,3 +1,3 @@
 def main():
-    print("start", 1)
+    debug("start", 1)
```

确认没问题，`--write` 落盘。CI 里拦危险调用：

```bash
python shapefind.py 'eval($X)' --check .   # 命中返回 1，干净返回 0
```

在标准库上的实测（920 个文件、386,762 行、排除 `site-packages`）：

| 查询 | 结果 |
|---|---|
| `grep -rn 'print('` | 970 行 |
| `shapefind 'print($X)'` | 246 处 |
| `shapefind 'print($$ARGS)'` | 368 处 |
| 全量扫描 | 3.7 秒 |

970 和 368 之间的差距就是这个工具的价值：注释里的 `print(`、`pprint`、docstring 里的示例、字符串里的 `print(`，全都不算调用。

368 而不是 611，是因为标准库里 611 个 `print` 调用中有 176 个带 `sep=` / `end=` 关键字参数，而模式 `print($$ARGS)` 明确说了「没有关键字参数」。想连关键字一起匹配，就写 `print($$ARGS)` 的扩展形式——工具不会替你想当然。

## 七、开发中踩的坑

三个，都是真实的，都有回归测试。

**坑一：`class $C` 匹配不上任何类。**

写完匹配引擎，`print($X)` 一切正常，但 `class $C: $$BODY` 一个都匹配不到。原因很反直觉：Python 的 AST 里，`ClassDef.name` 不是一个节点，是**一个纯字符串**。`FunctionDef.name`、`arg.arg`、`Attribute.attr`、`alias.name` 全都一样。我按「元变量一定是 `ast.Name` 节点」写的匹配逻辑，遇到这些位置直接走到字符串比较分支，`'__sf_s_C' != 'B'`，于是永远失败。

修法是加一条分支：字符串字段也检查元变量前缀，命中就把目标字符串本身存进捕获。捕获类型因此变成「AST 节点或字符串」两种，渲染时要分别处理。回归测试：`test_identifier_position_class_name`、`test_identifier_position_function_name`、`test_attribute_name`。

**坑二：`def $F($$ARGS): $$BODY` 一个都匹配不到。**

同一个根因的变体，但更隐蔽：`$$ARGS` 在参数列表里被解析成 `ast.arg(arg='__sf_m_ARGS')`，不是 `ast.Name`。我的 `meta_of()` 只认 `Name`，所以完全没识别出这是元变量。

修完这个还有第二层：即使识别出来了，`def f(a, b=2)` 也匹配不上，因为 Python 把默认值拆到 `arguments.defaults` 这个平行列表里，模式的 `defaults` 是空列表，目标的不空，字段比较失败。这个不能靠「空列表就放过」来糊——那会让 `f($X)` 匹配到 `f(a, b=1)`。最终规则是：**只有当签名里出现 `$$` 元变量时**，才跳过 defaults / vararg / kwarg 的比较，因为 `$$ARGS` 在语义上已经吞掉了整个参数区。回归测试：`test_multi_capture_in_body`、`test_multi_capture_in_call`。

**坑三：重写把缩进吃掉一半。**

第一个版本的重写，嵌套类里的 `return None` 从 12 格缩进变成了 8 格。原因是捕获的源码片段自带缩进：`$$BODY` 抓到一个 `def` 语句，它的第一行从列 8 开始（片段本身不含缩进），但续行是完整的 12 格。我把它拼进模板后再统一加匹配缩进，续行就多了一层。

修法分两半：捕获多行语句时，先剥掉它自己那层的缩进，变成相对缩进；替换时，按元变量在模板里所在的**列**给续行补空格。这样「剥掉绝对缩进 → 补上落点列 → 外层再统一加」三步走完，深度不变。回归测试：`test_nested_indentation_survives`（直接对应 #1405 的场景）、`test_multiline_body_keeps_relative_depth`。

顺带一个不算坑但值得记的：`ast` 的 `col_offset` 是 **UTF-8 字节偏移**，不是字符偏移。一行里有中文的时候，用 `col_offset` 去切 Python 字符串会切错位置。所以 `SourceText.offset()` 先把该行的字节前缀解码回字符再算列。回归测试：`test_non_ascii_column_offsets`，用 `s = "中文"; print(1)` 改成 `s = "中文"; log(1)`，结果正确。

## 八、一句话总结

ast-grep 教会了所有人「用代码搜代码」，然后用一个叫 `strictness` 的旋钮、一套 26KB 的 YAML schema 和一份 tree-sitter 节点表，把这份优雅还了回去——而 Python 自己的 `ast` 早就在那儿了，它连注释都不给你留，所以你根本不需要决定要不要忽略它。
