# 我的 Claude Code 工作流

我的大部分工作都用 Claude Code 完成。经过几个月的迭代，我发现一个规律：LLM
辅助写出的代码，腐烂得比手写代码更快。技术债不断累积，因为 LLM 不知道自己不知道
什么，而你也要等到为时已晚才会察觉。

这个 repo 就是我的解法：一套 skill 和工作流，强制「先规划后执行」、让上下文保持
聚焦、并在错误滚雪球之前就拦住它们。

## 为什么需要它

LLM 辅助编程在长期是失败的。技术债不断累积，因为 LLM 看不见它，而你又跑得太快、
来不及注意。我把这当作一个工程问题来对待，而不是工具问题。

LLM 是工具，不是协作者。当一个工程师说「加上重试逻辑」，另一个工程师会自动推断出
指数退避、抖动和幂等性。而 LLM 不会推断任何你没有明确说出的东西。它读不懂言外之
意，没有制度记忆，会满怀信心地实现错误的东西，还称之为「生产就绪」。

更大的上下文窗口也帮不上忙。给 LLM 更多文字，就像给人更厚的一摞文件；注意力会漂
向开头和结尾，中间的细节被漏掉。上下文越多，情况越糟。只给 LLM 当前任务恰好需要
的东西——别多给。

## 设计原则

这套工作流建立在四条原则之上。

### 上下文卫生

每个任务只拿到它恰好需要的信息——不多给。子 agent 从全新的上下文起步，因此架构知
识必须被编码到某个持久的地方。

我在每个目录里都用一种「双文件」模式：

**CLAUDE.md** —— Claude 进入目录时会自动加载。正因为它不管需不需要都会加载，内容
必须极简：一份表格式索引，配上简短描述和「何时打开该文件」的触发条件。当 Claude
打开 `app/web/controller.py` 时，它只会沿这条路径取回相关索引——而不是那些它可能
永远用不到的正文。

**README.md** —— 隐性知识：架构决策、从代码看不出来的不变量。判断标准是：如果开发
者读源码就能学到，那它就不该写在这里。Claude 只在 CLAUDE.md 的触发条件指示时才读
它。

核心原则是「即时上下文」（just-in-time context）。索引自动加载但保持精简，详细知识
只在相关时才加载。

technical-writer agent 会强制执行 token 预算：CLAUDE.md 约 200 token，README.md
约 500，函数文档 100，模块文档 150。这些上限强制纪律——如果你超了，多半是在记录代
码本身已经说明的东西。函数文档带有「use when…」触发条件，好让 LLM 知道何时该用到
它们。

planner 工作流会自动维护这套层级。如果你绕过 planner，就得自己维护。

### 先规划后执行

LLM 总会犯「第一枪」错误，无一例外。这套工作流把规划和执行分开，强制让歧义在「修
正成本还很低」时就浮现出来。

计划会记录：决策为什么这么做、哪些备选方案被否决、接受了哪些风险。计划写入文件。
当你清空上下文、重新开始时，这些推理依然留存。

### 复审循环

执行被拆成一个个里程碑——更小的单元，便于管理、可单独验证。这确保进度持续推进且
经过验证。否则执行就退化成瀑布式：早期一个小疏漏，agent 会层层放大每个错误，直到
结果无法使用。

每个阶段都有质量门把关。technical-writer agent 检查清晰度，quality-reviewer 检查完
整性。循环一直跑到两者都通过为止。

计划在执行开始前先过复审。执行期间，每个里程碑也要过复审，才轮到下一个。

### 高性价比委派

编排器（orchestrator）把任务委派给更小的 agent——简单任务用 Haiku，中等复杂度用
Sonnet。prompt 即时注入，恰好给小模型在每一步所需的引导。

当质量复审失败或问题反复出现时，编排器会升级到更高质量的模型。昂贵的模型只留给真
正的歧义，而非例行工作。

## 这玩意儿真的有用吗？

我没跑过正式的基准测试。我只能告诉你，我用这套工作流、完全靠 Claude Code 来构建和
维护非平凡应用时观察到了什么——后端系统、数据管道，以及用 C++、Python 和 Go 写的
流式应用。

那些我以前不断踩的坑，现在没有了：

**歧义消解。** 你让 LLM「给我做个三明治」，它端回来一份烤奶酪。技术上没错，但不是
你想要的。规划阶段强制这些误解在你把错的东西造出来之前就暴露。

**代码卫生。** 没有复审循环，同一个工具函数会在代码库里被重新实现十五遍。
quality-reviewer 会抓到这一点。technical-writer 则确保文档保持一致。

**LLM 可导航的文档。** 函数文档带有「use when…」触发条件。CLAUDE.md 文件告诉 LLM
哪些文件对某个任务重要。LLM 不再瞎猜哪些代码是相关的。

它比手写代码更好吗？我认为是，但我无法替所有人下结论。这套工作流很有主见。我是一
名后端工程师——这些模式应该也适用于前端工作，但我没测试过。如果你软件工程经验尚
浅，我很想知道它对你是帮助还是负担。

如果你认真对待 LLM 辅助编程、想试试一种结构化的方式，那就上手试试。我很想听听哪
些有用、哪些没用。

## 快速开始

克隆到你的 Claude Code 配置目录：

```bash
# 按项目
git clone https://github.com/solatis/claude-config .claude

# 全局（全新安装）
git clone https://github.com/solatis/claude-config ~/.claude

# 全局（已有 ~/.claude）
cd ~/.claude
git remote add workflow https://github.com/solatis/claude-config
git fetch workflow
git merge workflow/main --allow-unrelated-histories
```

## 使用方式

非平凡改动的工作流：探索 -> 规划 -> 执行。

**1. 探索问题。** 搞清楚你面对的是什么，找出解法。

这一步相对自由。如果项目和/或涉及面特别大，先用 `codebase-analysis` skill 好好探
索项目代码，再提出解法。

**2.（可选）想透彻。** 我非常频繁地用 `deepthink`，比任何其他 skill 都多。它处理
那些你还不知道答案该是什么形状的分析性问题——分类法设计、权衡取舍、定义性问题、
评价性判断、探索性调查。

它会自动判断复杂度。快速模式直接推理。完整模式启动多个带不同分析视角的并行子
agent，再通过「一致性模式」综合。两种模式都会自我验证。

所以，对大多数分析性问题，deepthink 就够了。当缺少上下文时，它会探索你的代码库。
只在问题边界清晰时才动用专门的 skill：

- `problem-analysis`：专门做根因分析
- `decision-critic`：对某个具体决策做压力测试

**3. 写计划。** 「用你的 planner skill 把计划写到 plans/my-feature.md」

planner 会让你的计划过一遍复审循环——technical-writer 把关清晰度、
quality-reviewer 把关完整性——直到通过。

planner 会记录所有决策、权衡，以及从代码里看不出来的信息，好让这些上下文不致丢失。

**4. 清空上下文。** `/clear`——从头开始。你需要的一切都已经写进计划里了。

**5. 执行。** 「用你的 planner skill 执行 plans/my-feature.md」

planner 会把工作委派给子 agent，它自己从不直接写代码。每个里程碑先过 developer，再
过 technical-writer 和 quality-reviewer。上一个里程碑没通过复审，下一个就不会开始。

只要有可能，它会并行执行多个任务。

各个 skill 的详细介绍见它们各自的 README：

- [DeepThink](skills/deepthink/README.md)
- [Codebase Analysis](skills/codebase-analysis/README.md)
- [Problem Analysis](skills/problem-analysis/README.md)
- [Decision Critic](skills/decision-critic/README.md)
- [Planner](skills/planner/README.md)

### 实战示例

我需要把一个老旧的 C# Windows 服务从「print 式日志」迁移到能真正轮转文件的方案。

这套代码用一个自制的 Log() 方法、靠 File.AppendAllText 往单个文件里写。没有轮转、
没有日志级别、同步 I/O 阻塞线程。散落各处的六个 Console.WriteLine 调用，在以服务方
式运行时根本没有输出去向。

我用一个 prompt 同时启动了探索和分析：

```
用你的 codebase analysis skill 简要探索这个 C# 项目，
重点关注当前所有发出 debug 日志的位置。

然后用你的 problem analysis skill 想清楚一个合适的
日志框架：
 * 必须兼容 .NET Framework 4.8.1
 * 必须开箱即用地支持日志轮转
 * 我们在同一台机器上跑多个进程，所以需要结构化的
   多进程支持
```

codebase analysis 找到了 31 处调用点和 Console.WriteLine 泄漏。problem analysis 针
对我的约束评估了 NLog、Serilog、log4net 和 Microsoft.Extensions.Logging。

结论是 NLog。它开箱即用地处理轮转和异步。多进程支持来自布局变量（layout
variables）。Serilog 也能用，但实现同样功能需要三个包。

我认同这个结论。这不是个复杂决策，所以我跳过了 `decision-critic`，直接进入规划：

```
用你的 planner skill 把实现计划写到：plan-logging.md
```

planner 浮现出两处歧义：

1. 替换所有 Log() 调用点，还是只换实现？对人来说显而易见，但值得一开始就澄清。
2. 日志轮转的默认值。planner 假设按 1 天轮转，但我还需要按 1GB 大小轮转。

计划过了复审。technical-writer 标记出那些「解释 what 而非 why」的注释——NLog.config
里有诸如「configures file target（配置文件目标）」这样的注释，而不是解释轮转策略。
quality-reviewer 抓到了两个我会漏掉的问题：服务的 OnStop() 处理器里没有显式调用
LogManager.Shutdown()，以及文件路径少了 src/ 前缀、不正确。

这些正是你跳过复审循环时会带进生产环境的 bug。shutdown 那个问题会导致服务重启时丢
日志。路径那个问题则会静默失败。

修完之后，我清空上下文并执行：

```
用你的 planner skill 执行：@plan-logging.md
```

developer、debugger、technical-writer 和 quality-reviewer 一起跑完实现。每个里程碑
先过复审，才轮到下一个。如果实现偏离了计划，我会知道。

## 其他 skill

不是每个任务都需要完整的规划工作流。下面这些 skill 处理特定的关注点。

### DeepThink

这个 skill 我一天用好几次——只要我还不知道答案该是什么形状，就会用它。

和 `problem-analysis` 或 `decision-critic` 不同，deepthink 没有固定结构。它处理权
衡取舍、分类法问题、评价性判断——你抛给它什么都行。

那么，我什么时候会动用它？

元认知调试。我老犯同一个错。LLM 老是误解任务。为什么？有什么东西坏了，我得先看见
它才能修。

策略评估。存在多个有效的方案（而且光凭直觉不够）。PDF 转换：下载 TeX 源码、直接解
析 PDF、还是让 LLM 用视觉方式渲染。S3 制品版本管理：时间戳路径、指针文件、校验和。
系统性比较胜过直觉。

最佳实践调研。规范的做法是什么？成熟的 CI/CD 系统如何处理制品版本管理？行业里多半
已有现成模式——只是我还不知道。

架构与设计。这些组件应该如何交互？委派边界划在哪里？我会先想清楚，再投入写代码。

合并决策。这两个 skill 是否应该合并？它们服务于不同目的，还是我在维护不必要的复杂
度？

两种模式，自动判断。快速模式直接推理。完整模式启动多个带不同分析视角的并行子
agent，再通过「一致性模式」综合。

```
用你的 deepthink skill 想清楚 [问题]
```

要显式指定模式：

```
用你的 deepthink skill（quick）来 [问题]
用你的 deepthink skill（full）来 [问题]
```

### Refactor

LLM 生成的代码会积累技术债。LLM 看不到跨文件的重复，也注意不到巨型函数正在膨胀。

refactor skill 在多个维度上并行探索——命名、提取、类型、错误处理、模块、架构、抽象
——根据证据验证发现，再输出按优先级排序的建议。它不生成代码；它告诉你该修什么、为
什么。

适用场景：

- LLM 生成的功能能跑、但感觉乱的时候
- 重大改动之前，识别摩擦点
- code review 暴露出结构性问题时
- 简单改动却要动很多文件时

```
用你的 refactor skill 处理 src/services/
```

带聚焦方向：

```
用你的 refactor skill 处理 src/ —— 聚焦于重构渲染引擎，使它能在多个组件中复用。
```

### Prompt Engineer

这套工作流完全由 prompt 构成。每一个都可以单独优化。

这个 skill 分析 prompt、提出改动并明确标注所依据的模式，在应用任何改动前等待你批
准。

适用场景：

- 某个子 agent 定义表现不如预期
- 优化某个 skill 的 Python 脚本 prompt
- 审查多 prompt 工作流的一致性

```
用你的 prompt engineer skill 优化 agents/developer.md 的系统 prompt
```

这个 skill 是用它自己优化出来的。

### Doc Sync

CLAUDE.md/README.md 这套层级需要维护。结构会随时间变化，文档会漂移。

doc-sync skill 审计并同步整个 repo 的文档。

适用场景：

- 在已有 repo 上初次引入这套工作流
- 重大重构或目录重组之后
- 定期审计，检查文档漂移

如果你一直在用规划工作流，technical-writer agent 会把文档作为执行的一部分处理掉。
doc-sync 主要用于初次引入或事后补救。

```
用你的 doc-sync skill 同步整个 repo 的文档
```

要做定向更新：

```
用你的 doc-sync skill 更新 src/validators/ 里的文档
```
