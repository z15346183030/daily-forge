# benchloop · 2026-10-27

## 一、我拆的是哪个项目

[sharkdp/hyperfine](https://github.com/sharkdp/hyperfine) — 29,156 星，Apache-2.0，Rust 写的命令行基准测试工具，最近一次提交是 2026-10-12。

一句话定位：**把「A 和 B 哪个快」变成一个可重复、有统计支撑的命令行问题**。跑 N 次，算均值和标准差，顺手告诉你谁快几倍。

为什么选它，有三个理由。

第一，**它解决的问题是被反复验证过的真需求**。二十九千星不是刷出来的，写性能对比、写 README 里的 benchmark 表格、CI 里防止性能回退，都要用。这类工具的替代品不多，而且大多更重。

第二，**它有一个开了八年的功能请求，而且作者明确说过「我倾向于不加」**。这一点很关键——说明这个缺口不是没人想到，是被权衡掉了。被权衡掉的东西，往往就是别人真正的痛点。

第三，**它的核心可以收敛成一个单一职责**：按某种顺序执行命令、计时、算统计。周边的参数扫描、五种导出格式、生命周期钩子，全都可以砍掉而不影响核心价值。这符合「一天之内做出有价值的精简版」的标准。

顺手说一句筛选过程：今天没从 Trending 里挑。过去两周的记录显示，Trending 榜上长期被 agent 框架和 skill 仓库占满，那些项目要么没有可拆的核心算法，要么依赖外部模型服务才能跑。所以改成按「有明确复杂度抱怨」去找——在搜索 API 里按星标筛，再翻 issue 的评论数，找那种「提了三年还在讨论」的条目。hyperfine 的 #21 就是典型。

## 二、它怎么做的

先看骨架。`src/` 下 41 个文件、233 KB Rust，主干是这样一条链：

```
              ┌──────────────┐
  argv  ───►  │   cli.rs     │  解析参数（22 KB）
              └──────┬───────┘
                     ▼
              ┌──────────────┐
              │  options.rs  │  校验 + 归一化成执行配置（18 KB）
              └──────┬───────┘
                     ▼
              ┌──────────────┐        ┌──────────────────┐
              │ scheduler.rs │───────►│  export/         │  json / csv /
              └──────┬───────┘        │  markdown / ...  │  markdown / ...
                     ▼                └──────────────────┘
              ┌──────────────┐
              │ benchmark/   │  一条命令跑 N 次，收集 Measurement
              │  mod.rs      │
              │  executor.rs │  真正 fork 子进程的地方
              └──────┬───────┘
                     ▼
              ┌──────────────┐
              │  timer/      │  五个平台各一套计时器
              └──────────────┘
```

核心链路拆成五步：

1. **解析**。`cli.rs` 拿到 argv，`options.rs` 把它变成一份执行配置。光是这两件事就占了 39,658 字节——比我们整个工具还大一倍。这不是作者啰嗦，是功能多到必须这么写：参数扫描、生命周期命令、输出策略、失败策略、导出策略，每个都有自己的选项。

2. **调度**。`scheduler.rs` 做的事非常朴素：遍历命令列表，每条命令交给 `Benchmark::run()`，跑完把结果追加到 `results`，顺便导出一次（防止后面崩了丢数据）。

3. **执行**。`executor.rs` 里区分 `RawExecutor`（直接 exec）和 `ShellExecutor`（走 shell）。后者有个 `calibrate()` 方法，注释写得很直白：走 shell 时，必须先量出 shell 的启动开销，之后从每次结果里减掉。

4. **计时**。`timer/` 下五个实现：Linux 读 `wait4` 的 rusage，macOS 用 `getrusage`，Windows 用 `GetProcessTimes`，另外还有一套纯墙钟的兜底。为了拿到 user / system 时间，每个平台都得单独写一遍。

5. **统计与输出**。均值、标准差、min/max，离群点用修改 Z 分数（中位数 + MAD）判定，然后渲染成文本或五种格式之一。

用大白话讲，它的工作原理是：**把命令扔进子进程，掐表，重复 N 次，然后把 N 个数字算成一个故事**。所有复杂度都在「怎么掐表才准」和「故事怎么讲才不误导」这两件事上。

## 三、它哪里让人难受

### 一、交错执行：2018 年提的，到今天还开着

> "This would be useful when comparing two similar commands, as interleaving them makes it less likely that e.g. a load spike will unfavorably affect only one of them, or due to e.g. thermal throttling negatively affecting the last command."
>
> — [#21 Interleaved benchmark execution (multiple commands)](https://github.com/sharkdp/hyperfine/issues/21)，42 条评论，2018-01-21 提出

作者当年的回应是：

> "Also, simplicity is a desirable goal for me. If there are command line options that we don't necessarily need, I would tend to leave them out."

这条理由本身站得住——工具确实不该什么都加。问题在于它把成本转嫁给了用户：**默认行为就是有偏的**。

证据在 `src/benchmark/scheduler.rs`：

```rust
for (number, cmd) in self.commands.iter().enumerate() {
    self.results.push(
        Benchmark::new(number, cmd, self.options, &*executor).run(self.results.first())?,
    );
}
```

一条命令的全部轮次跑完，再跑下一条。2020 年 comex 在同一个 issue 里把后果讲透了：

> "Instantaneous load spikes are one thing, since they can be removed by outlier detection or at least detected by a high variance. But there are also many ways in which the state of the system can vary *slowly* over time. For example: Thermal throttling..."

这句话点到了要害：**离群检测治的是尖峰，治不了漂移**。一台笔记本从凉跑到热是一个连续过程，中间不会出现任何一个「离群点」，但后测的那条命令会整体慢一截。这不是统计能修的，是实验设计的问题。

我在自己机器上复现了这个偏差，用的是同一个命令跟它自己比（细节在第四节）：

```
顺序模式：#1 median 2.56 s，#2 median 658 ms → 报告「#2 快 3.90 倍」
```

两条命令一模一样，工具却给出了 3.90 倍的差距。这个数字不是 bug，是默认执行顺序的必然产物。

### 二、想要置信区间和 p 值，请自己去 scripts 目录

> "I use this in academic research, where I'll be computing p-values to describe the confidence with which the sample data suggests that one program is faster than another."
>
> — [#523 Confidence intervals, p-values](https://github.com/sharkdp/hyperfine/issues/523)

作者的答复是把 `scripts/advanced_statistics.py` 和 `scripts/welch_t_test.py` 指给用户。脚本能用，但它们是旁路：你得先把结果导出成 JSON，再自己喂进去。而且脚本只认识「两组独立样本」——交错执行本来会产生的配对结构，在导出那一刻就丢了。

同一个 issue 里还有人追问 `1.40 ± 0.03 times faster` 到底该怎么读。答案是「一个标准差」，不是置信区间。一个在学术场景里被使用的工具，输出的核心数字含义需要用户去翻源码才能确定，这件事本身就是设计上的欠账。

### 三、`--subtract`：讨论了两年，最后靠写假 shell 绕过

> "My usecase for this is that I want to benchmark page loads, for which I'm using headless chromium, but want to subtract the time that chromium takes to start up."
>
> — [#265 Add --subtract option](https://github.com/sharkdp/hyperfine/issues/265)

四年后，有人在 issue 里贴了自己的解法：写一个假 shell，通过 `-S` 传给工具，让它在执行除目标程序外的每条命令时先垫一段固定开销，把差异抹平。这已经不是「用工具」，是在跟工具搏斗。

hyperfine 其实已经对 shell 启动时间做了扣减（`calibrate()`），但只覆盖 shell 本身，不覆盖「你的程序自己的启动开销」。而后者恰恰是很多真实场景里的主要成本。

### 四、一条关不掉的警告

> "I keep consistently getting that warning when absolutely nothing else on my computer is running... I've already tried `--shell=none` and as many as 1000 warmup runs and it still shows up. ... could there please be a way to suppress it?"
>
> — [#528 Suppress "statistical outliers detected" warning](https://github.com/sharkdp/hyperfine/issues/528)

警告本身没错，错在它只说「检测到离群点」，不说「哪几次、差多少、要不要紧」。用户看到这句话，唯一能做的就是去翻 JSON 自己找。而 `src/outlier_detection.rs` 的阈值是 `1.4826 * 10.0`，也就是 10 个修改 Z 分数——一个已经很宽松的门槛，还能持续触发，说明数据里确实有东西，只是工具没说清是什么。

### 五、`--prepare` 会改变测量结果

> "When using `hyperfine` I noticed that it always reported worse performance when using the `--prepare` option to execute a command in between runs even if the command should not have any effect on the system or the benchmark command."
>
> — [#715 Using --prepare increases benchmark results](https://github.com/sharkdp/hyperfine/issues/715)

生命周期钩子是功能，也是污染源。`--prepare` 在两次测量之间插入一段执行，这段执行的缓存效应、调度影响，全都算进了下一次测量。工具给了你这个旋钮，却没有告诉你它会影响结果。

## 四、我们做了什么

**定位一句话：只做「按轮交替跑、用配对数据算比值」这一件事，并把「这次测量到底有没有意义」直接说出来。**

### 砍了什么

| 砍掉 | 为什么 |
|---|---|
| `--prepare` / `--setup` / `--cleanup` | 三条生命周期命令各自还要处理失败策略、输出策略、是否计时。而且它本身就是污染源（#715） |
| 参数扫描 `-P` | 好用，但它是另一个功能：批量实验。不该和「比较两条命令」混在同一个命令里 |
| 五种导出格式 | JSON 一种够用。Markdown 表格三行代码就能从 JSON 生成 |
| CPU / 内存指标 | 标准库拿不到跨平台的子进程 CPU 时间。宁可只给一个准的墙钟时间 |
| 直方图、进度条、配色主题 | 输出是给人看的一段文字，不需要终端 UI |
| shell 启动时间扣减 | 见下面「为什么不扣」 |

### 留了什么

| 留下 | 为什么 |
|---|---|
| 墙钟时间 + 预热 | 所有基准的地基 |
| 中位数、均值、标准差、min/max | 少一个都看不懂分布 |
| 修改 Z 分数标离群 | 抗尖峰，二十行就能写完 |
| JSON 输出 | CI 要用 |

### 加了什么

| 加 | 为什么 |
|---|---|
| 默认交错，ABBA 顺序 | 直接对着 #21 去。单数轮反序，让「排在前面的那次」不总是同一条命令 |
| 配对 bootstrap 区间 + 符号检验 p 值 | 交错跑出来的数据天生配对，就用配对的方法算。对应 #523 |
| 进程创建地板 | 不扣减，而是量出来告诉你 |
| 离群点标出具体轮次和耗时 | 对应 #528：不说「有离群点」，说「第 4 次 1.36 s」 |

### 核心算法怎么工作的

假设比较 A 和 B，跑 10 轮。执行顺序不是「A×10 再 B×10」，而是：

```
第 1 轮：A B
第 2 轮：B A      ← 反序
第 3 轮：A B
第 4 轮：B A
...
```

每一轮产出一个比值 `r_i = t_A(i) / t_B(i)`，共 10 个。`r_i > 1` 表示这一轮 B 更快。

然后做两件事：

**第一，给这个比值一个区间。** 对 10 个比值做 bootstrap：有放回地重采样 2000 次，每次算一次几何平均，得到 2000 个估计值，取 2.5% 和 97.5% 分位。用几何平均而不是算术平均，因为比值是乘性的——2 倍和 0.5 倍应该互相抵消。随机种子固定，同一组数据每次跑出同一个区间。

**第二，给这个区间一个「是不是巧合」的判断。** 数一下 10 轮里有几轮 B 领先，做精确二项检验（符号检验）。10 轮全胜，双侧 p = 2/1024 ≈ 0.0020；8 胜 2 负，p ≈ 0.11；6 胜 4 负，p = 0.75，等于什么都没说。

真实例子（list 查重 vs set 查重，各 10 轮）：

```
#1  median 1.71 s
#2  median 511 ms
#2 vs #1   3.34× 更快   95% CI [3.21, 3.46]   10/10 轮领先 (p=0.0020)
```

区间很窄，10 轮全胜，所以这个 3.34 倍是可信的——至少「B 比 A 快」这个方向可信。至于 3.34 是不是真实差距，看下面那行地板提示。

### 为什么不扣 shell 启动时间

hyperfine 的做法是先量、再减。减法的前提是「开销是常数」，可它不是。我在自己机器上量了 5 次空命令：

```
155 ms / 159 ms / 169 ms / 234 ms / 323 ms
```

拿中位数 169 ms 去减，每次测量都会引入一个新的、大小未知的误差项，而且减完之后你无法知道误差有多大。极端情况下还能减出负数。

benchloop 改成：量一次地板，如果被测命令离地板太近，直接说清楚：

```
spawn floor 181 ms（空命令的中位数）—— 最快的命令只有它的 2.8 倍，量到的主要是进程创建，不是你的程序
```

数字保持原样，判断权交回用户。这比一个看起来更「精确」的减法结果诚实。

## 五、和原版差在哪

| 维度 | hyperfine | benchloop |
|---|---|---|
| 执行顺序 | 一条跑完再跑下一条 | 默认交错，单数轮反序 |
| 比较统计 | 均值比值 ± 一个标准差 | 每轮比值的几何平均 + bootstrap 95% 区间 + 符号检验 p 值 |
| 进程开销 | 量出来并从结果里扣掉 | 量出来作为地板提示，不扣 |
| 离群点 | 只报数量，警告关不掉 | 报具体轮次和耗时，不改变任何计算结果 |
| 计时指标 | 墙钟 + user/system + 内存（部分平台） | 只有墙钟 |
| 输出 | 文本 + 5 种导出 + 直方图 | 文本 + JSON |
| 生命周期钩子 | `--prepare` / `--setup` / `--cleanup` | 无 |
| 参数扫描 | `-P 1 8 'make -j {n}'` | 无 |
| 代码规模 | 41 个 Rust 文件，`src/` 共 233 KB | 1 个 Python 文件，625 行 / 21 KB |
| 依赖 | Rust 工具链 + 若干 crate | Python 3.9+ 标准库 |

规模差 11 倍，但我不会说「功能是它的 1/11」。砍掉的都是它做得对、只是我不需要的东西：参数扫描它做得比我好，导出格式它比我全，跨平台 CPU 计时我根本不做。**差异不在「谁功能多」，而在「默认值假设了什么」**——它假设你会一条一条跑，我假设机器会变。

## 六、怎么用

不需要安装，一个文件：

```bash
curl -O https://raw.githubusercontent.com/z15346183030/benchloop/main/benchloop.py
python benchloop.py --version
```

**量一条命令，只看分布：**

```bash
$ python benchloop.py -n 5 "python -c pass"
benchloop  runs=5  warmup=2  顺序  shell=auto

#1  python -c pass
    median 539 ms   mean 531 ms ± 44.6 ms (8.4%)   min 481 ms   max 597 ms

spawn floor 167 ms（空命令的中位数）—— 最快的命令只有它的 3.2 倍，量到的主要是进程创建，不是你的程序
```

这台机器上 `python -c pass` 要 539 ms，其中 167 ms 是 `cmd.exe` 的启动，剩下 370 ms 是 Python 解释器自己的启动。看到这一行你就知道：拿它去比两个 Python 脚本的启动速度，误差会比差异还大。

**比较两条命令：**

```bash
$ python benchloop.py -n 10 "python examples/dedupe_list.py 20000" "python examples/dedupe_set.py 20000"
benchloop  runs=10  warmup=2  交错  shell=auto

#1  python examples/dedupe_list.py 20000
    median 1.71 s   mean 1.73 s ± 92.8 ms (5.4%)   min 1.62 s   max 1.92 s

#2  python examples/dedupe_set.py 20000
    median 511 ms   mean 519 ms ± 34.3 ms (6.6%)   min 479 ms   max 601 ms
    outliers: 第 1 次 601 ms（只标记，不剔除）

comparison  (baseline #1, 10 轮配对)
    #2 vs #1   3.34× 更快   95% CI [3.21, 3.46]   10/10 轮领先 (p=0.0020)

spawn floor 181 ms（空命令的中位数）—— 最快的命令只有它的 2.8 倍，量到的主要是进程创建，不是你的程序
```

**塞进 CI：**

```bash
$ python benchloop.py -n 3 -q "python a.py" "python b.py"
#2 vs #1: 1.30× 更快

$ python benchloop.py -n 5 --json "python a.py" "python b.py" > result.json
```

**同一命令对同一命令，验证方法本身没偏差：**

```bash
$ python benchloop.py -n 10 "python examples/dedupe_set.py 20000" "python examples/dedupe_set.py 20000"
comparison  (baseline #1, 10 轮配对)
    #2 vs #1   1.01× 更慢   95% CI [0.92, 1.07]   6/10 轮领先 (p=0.7539)
```

1.01×，区间跨过 1.0，p=0.75。这是我最看重的一次测量：**工具没有系统性地偏向任何一边**。

## 七、开发中踩的坑

**坑一：顺序模式写成了轮转，测试抓到了。**
`plan_order` 一开始是这样写的：外层遍历轮次，内层遍历命令，`interleave=False` 时只是不做反序。结果「顺序模式」变成了 A B A B A B——它还是交替跑，只是少了一次倒序。这是自己给自己挖的坑：变量名叫 sequential，行为叫 round-robin。
写完测试才发现：`test_sequential_groups_by_command` 断言 `[0,0,0,1,1,1]`，实际得到 `[0,1,0,1,0,1]`。改成 `if not interleave: return [(r, idx) for idx in ... for r in ...]` 才对。**如果没有这条测试，这个 bug 会一直活着**——因为它在单条命令下完全看不出来。

**坑二：Windows 上 `shlex.split(posix=False)` 会把引号留在 token 里。**
`--shell none` 时要把命令行拆成 argv，Windows 上必须用 `posix=False`（否则反斜杠路径被吃掉），但代价是 `"C:\Program Files\python.exe"` 拆出来的第一个 token 带着引号。直接传给 `subprocess` 会被转义成字面引号，命令就找不到了。
修法是拆完再手动剥一层引号，回归用例叫 `test_windows_quoted_path_loses_its_quotes`。这个坑只有在路径带空格时才出现，而我的开发机上恰好没有空格路径——所以这条用例是照着 Windows 的行为规范写的，不是照着 bug 写的。

**坑三：这台机器的进程创建开销大到让我怀疑代码。**
第一次跑 `python benchloop.py -n 5 "python -c pass"`，得到 median 542 ms，地板 165 ms。我第一反应是计时写错了。单独测了三次才确认：`cmd /c exit 0` 在这台机器上真的要 155–323 ms，`python -c pass` 要 400–575 ms。这是沙箱环境的真实开销，不是 bug。
这个意外直接催生了 `spawn floor` 这个功能——如果我自己都会被自己的数字骗，用户一定也会。

**坑四：负载注入实验前两次没效果。**
想证明「交错能消除漂移」，得先制造漂移。`examples/load_spike.py` 第一版只开 4 个进程，这台机器有 8 个核，被测进程总能找到一个空闲核跑，结果两条命令的差异只有 18%，还在噪声范围内。
改成 `multiprocessing.cpu_count()` 全核压满，效果立刻出来了：

```
顺序模式（两条命令完全相同）：
#1  median 2.56 s
#2  median 658 ms
comparison  #2 vs #1  3.90× 更快
```

同一组条件，交错模式：

```
#2 vs #1   1.17× 更慢   95% CI [0.42, 1.39]   3/8 轮领先 (p=0.7266)
```

偏差从 3.90× 降到 1.17×，区间跨过 1.0，p=0.73——工具给出的结论是「这点差异说明不了什么」，而这就是正确答案。**这个实验反过来验证了第一节那个 issue 不是理论担忧。**

## 八、一句话总结

基准测试工具真正要防的不是测不准，是**测出一个错的数字还让你相信它**——所以 benchloop 把 A 和 B 揉在一起跑，再把「这次测量到底有没有意义」写在最后一行。
