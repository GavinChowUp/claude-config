<!-- applicable_phases: codebase_review, refactor_code -->

# Codebase Patterns

评估仅在全代码库分析中才能浮现的模式。

**核心问题**：出现了哪些模式？理解不应需要阅读整个代码库。跨文件的重复模式表明存在缺失的抽象。死导出和死模块作为噪音不断积累。这些问题在本地审查中是不可见的——只有从整体视角才能看到。

**关注点**：

- 需要 5+ 个文件才能理解且没有文档的流程
- 在 3+ 个文件中应用的相同转换（缺失抽象）
- 在任何地方都没有调用者的导出函数
- 始终为 true/false 的功能标志（从未切换）
- 没有来自活跃代码导入的死模块

**门槛**：当可理解性受损（5+ 个文件，无指导）时标记。当模式出现在 3+ 个实现中**且**提取有帮助时标记。标记可证明不可达/未使用的代码（不是公共 API 或插件接口）。此组需要全代码库可见性。

<design-mode>
不适用——此组需要全代码库分析。
</design-mode>

<code-mode>
评估代码库时（Codebase Review、Refactor）：

- 我能在不阅读许多文件的情况下理解流程吗？
- 是否存在应该被抽象的重复模式？
- 导出/模块层是否存在死代码？

证据格式：描述跨多个文件的模式，或引用特定的死导出。
</code-mode>

---

## 1. Cross-File Comprehension

<principle>
理解一个流程不应需要阅读整个代码库。当理解一个操作需要 5+ 个文件且没有指导时，可理解性已经受损。
</principle>

Detect: 理解这个流程需要阅读多少文件？是否有文档或编排器解释整体情况？

<grep-hints>
结构指示词（起点，非定论）：
调用链，事件处理器，回调注册
</grep-hints>

<violations>
说明性模式（非穷举——类似违规也存在）：

[high] Implicit contracts

- 文件间的隐式契约（调用方必须了解被调用方内部）
- 任何需要未记录假设才能理解的流程

[medium] Hidden dependencies

- 隐藏的依赖（文件 A 假设文件 B 已先运行）

[low] Scattered flow

- 分散的控制流（一个操作跨 5+ 个文件且没有编排器）
  </violations>

<exceptions>
有良好文档的模块边界。插件架构。有明确事件契约的事件驱动设计。
</exceptions>

<threshold>
当理解单个操作需要阅读 5+ 个文件且没有流程文档时标记。
</threshold>

## 2. Abstraction Opportunities

<principle>
跨文件的重复模式表明存在缺失的抽象。当你在 3+ 个地方看到相同的转换时，一个概念正在尝试浮现。
</principle>

Detect: 隐藏在这些重复模式中的领域概念是什么？提取共享抽象是否能减少重复？

<grep-hints>
结构指示词（起点，非定论）：
并行实现，相似的转换链，重复的配置形状
</grep-hints>

<violations>
说明性模式（非穷举——类似违规也存在）：

[high] Missed abstractions

- 相同转换在多个文件中应用（3+ 次出现）
- 任何跨实现出现且应当共享的模式

[medium] Structural duplication

- 以不同方式做相似事情的并行类层级
- 复制粘贴继承（带细微变体的相似类）

[low] Configuration patterns

- 结构相同的数据转换管道
- 无抽象的重复配置模式
  </violations>

<exceptions>
有意独立的相似但独立的实现。领域特定变体。生成相似代码的模板/生成器。
</exceptions>

<threshold>
当模式出现在 3+ 个实现中**且**修复方案是提取共享抽象时标记。这些问题只有在看到多个实现后才变得可见。
</threshold>

## 3. Zombie Code (Codebase Scope)

<principle>
死代码是误导读者的噪音。无法执行或从未被调用的代码应该删除，而非留下来迷惑未来的维护者。
</principle>

Detect: 如果我删除这个导出或模块，任何测试会失败或行为会改变吗？

<grep-hints>
模式指示词（起点，非定论）：
0 调用者的导出符号，功能标志，配置选项，死模块
</grep-hints>

<violations>
说明性模式（非穷举——类似违规也存在）：

[high] Dead exports

- 在代码库中 0 调用者的导出函数
- 始终为 true/false 的功能标志（在任何环境中从未切换）
- 任何没有使用者的可公开访问的代码

[medium] Stale flags

- 死标志（特性已上线，标志从未移除）

[low] Orphaned configuration

- 从未读取的配置选项
- 死模块（任何活跃代码路径都没有导入）
  </violations>

<exceptions>
公共 API 入口点。插件接口。外部控制的功能标志。带弃用通知的向后兼容导出。
</exceptions>

<threshold>
当代码可证明不可达/未使用**且**不是公共 API 入口点、插件接口或有文档记录的兼容性垫片时标记。
</threshold>

注意：文件范围的 zombie 代码（注释块、不可达分支）在 03-patterns-and-idioms.md Zombie Code（File Scope）中处理。
