<!-- applicable_phases: design_review, codebase_review, refactor_design, refactor_code -->

# Module & Dependencies

评估模块边界是否清晰，以及架构是否与变更模式一致。

**核心问题**：边界是否清晰？模块应当有明确的边界且耦合最小。架构应与特性的实际变更方式保持一致。当变更扩散到无关模块或需要修改许多组件时，边界是错误的。

**关注点**：

- 循环依赖
- 层级违规（领域层导入基础设施）
- 错误的组件边界（特性被尴尬地拆分）
- 强迫跨切关注点变更的架构（即使是单领域特性）

**门槛**：当依赖导致编译问题或领域污染时标记。当添加一个特性需要修改许多无关组件时标记。这本质上是关于文件和模块之间的关系，而非本地代码模式。

<design-mode>
评估代码意图时（Design Review 阶段）：

- 提出的设计是否创建了循环依赖？
- 是否违反了层级边界？
- 实现这个特性是否需要修改许多组件？

证据格式：引用代码意图描述，指出边界问题。
</design-mode>

<code-mode>
评估实际代码时（Codebase Review、Refactor）：

- 导入图是否显示循环依赖？
- 实际导入中是否存在层级违规？
- 特性是否跨许多松散相关的组件拆分？

证据格式：引用 import 语句或描述依赖结构，指出问题所在。
</code-mode>

---

## 1. Module Structure

<principle>
模块应有明确的边界且耦合最小。当变更扩散到无关模块时，边界是错误的。
</principle>

Detect: 变更是否扩散到无关模块？模块能否在不理解其依赖方的情况下被修改？

<grep-hints>
结构指示词（起点，非定论）：
导入图，依赖声明，模块边界
</grep-hints>

<violations>
说明性模式（非穷举——类似违规也存在）：

[high] Structural violations

- 循环依赖（如 A 导入 B 导入 A）
- 层级违规（如领域层导入基础设施）
- 任何导致编译顺序问题或领域污染的依赖

[medium] Cohesion problems

- 错误的内聚（无关事物分组在同一模块）
- 缺少外观（模块内部直接暴露）

[low] Scope creep

- 上帝模块（一个模块承担太多职责）
  </violations>

<exceptions>
同一有界上下文内的循环依赖。导入领域的基础设施适配器。共享内核模式。
</exceptions>

<threshold>
当依赖导致编译顺序问题**或**层级违规允许基础设施污染领域时标记。
</threshold>

## 2. Architecture

<principle>
架构应与变更模式一致。当添加一个特性需要修改许多无关组件时，架构与领域对抗。
</principle>

Detect: 添加一个特性是否需要修改许多组件？跨切关注点变更是否表明边界不对齐？

<grep-hints>
结构指示词（起点，非定论）：
组件边界，服务接口，配置位置
</grep-hints>

<violations>
说明性模式（非穷举——类似违规也存在）：

[high] Boundary misalignment

- 错误的组件边界（特性被尴尬地拆分）
- 单点故障（无回退，无重试路径）
- 任何强迫单领域特性也需要跨切变更的架构

[medium] Scaling issues

- 扩展瓶颈（需要异步的地方用了同步）
- 分布式代码中使用单体模式（或反之）

[low] Missing structure

- 缺少抽象层（所有东西直接耦合）
- 配置分散（无中央策略，设置在多处）
  </violations>

<exceptions>
为简单性故意耦合。早期阶段单体。带共享内核的有界上下文。
</exceptions>

<threshold>
当架构强迫单领域特性也需要跨切变更时标记。
</threshold>
