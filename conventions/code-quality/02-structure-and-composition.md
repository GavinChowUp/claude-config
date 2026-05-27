<!-- applicable_phases: design_review, diff_review, codebase_review, refactor_design, refactor_code -->

# Structure & Composition

评估代码结构是否有助于理解和变更。

**核心问题**：我能独立理解这个单元吗？我能在不理解其依赖方的情况下修改它吗？结构应当揭示意图并隔离关注点。

**关注点**：

- 做多件事的函数（描述时需要「and」）
- 深层嵌套模糊控制流
- 隐藏在布尔标志中的隐式状态机
- 使代码不可测试的硬编码依赖
- 组件定义分散在多处
- 丢失信息的错误处理

**门槛**：当结构模糊意图，或变更会不必要地扩散时标记。长度本身不是问题；职责不清晰才是。

<design-mode>
评估代码意图时（Design Review 阶段）：

- 提出的函数是做一件事还是多件事？
- 意图是否描述了清晰的职责边界？
- 设计是注入依赖还是硬编码依赖？
- 组件定义是在一处完整，还是分散在多处？

证据格式：引用代码意图描述，指出结构问题。
</design-mode>

<code-mode>
评估实际代码时（Diff Review、Codebase Review、Refactor）：

- 函数是否过长或嵌套过深？
- 布尔标志是否创建了隐式状态机？
- 错误处理是否保留了上下文？
- 组件定义是否分散（需求在一处，验证在另一处）？

证据格式：引用代码（含 file:line），指出问题所在。
</code-mode>

---

## 1. Function Composition

<principle>
函数应只做一件可以用单句描述的事。当描述需要「and」时，函数很可能需要拆分。
</principle>

Detect: 我能用一句话（不含「and」）描述这个函数的目的吗？

<grep-hints>
结构指示词（起点，非定论）：
函数超过 50 行，参数数量超过 4
</grep-hints>

<violations>
说明性模式（非穷举——类似违规也存在）：

[high] Responsibility diffusion

- 上帝函数（多个无关职责）
- 长参数列表（4+ 个参数说明缺少概念）
- 任何需要多句话来描述目的的函数

[medium] Structural complexity

- 深层嵌套（3+ 层条件）
- 混杂抽象层次（高层编排与低层细节混在一起）

[low] Interface friction

- 分叉行为的布尔参数（考虑拆分为两个函数）
  </violations>

<exceptions>
线性做一件事的长函数（如状态机、解析器）。源自错误处理的嵌套深度。
</exceptions>

<threshold>
当函数有多个无关职责时标记。长度本身不是问题。
</threshold>

## 2. Control Flow Smells

<principle>
控制流应揭示意图，而非隐藏它。当跟踪执行需要显著的脑力时，结构需要简化。
</principle>

Detect: 控制流是否比必要的更难跟踪？读者是否需要追踪多个分支才能理解行为？

<grep-hints>
模式指示词（起点，非定论）：
`elif.*elif.*elif`, `switch`, `case`, `? :.*? :`, 三元链
</grep-hints>

<violations>
说明性模式（非穷举——类似违规也存在）：

[high] Excessive branching

- 长 if/elif 链（5+ 个分支 → 查找表或策略模式）
- 任何需要追踪才能理解的分支结构

[medium] Obscured flow

- 嵌套三元（2+ 层 → 提取为命名变量）
- 早返回候选埋藏在嵌套 else 分支中

[low] Hidden complexity

- 条件赋值级联
- 隐藏边界情形的隐式 else 分支
  </violations>

<exceptions>
穷举模式匹配。带显式状态的状态机。
</exceptions>

<threshold>
当控制流模糊意图时标记。对已记录情况的显式分支是可以接受的。
</threshold>

## 3. State and Flags

<principle>
相互作用的布尔标志创建隐式状态机。当理解状态需要追踪多个标志时，应将状态机显式化。
</principle>

Detect: 布尔标志是否在创建隐式状态机？标志是否以需要精神追踪的方式相互作用？

<grep-hints>
模式指示词（起点，非定论）：
`is_.*=`, `has_.*=`, `_flag`, `_state`, 多个布尔赋值
</grep-hints>

<violations>
说明性模式（非穷举——类似违规也存在）：

[high] Implicit state machines

- 布尔标志缠结（3+ 个标志相互作用 = 隐式状态机）
- 任何标志交互需要精神状态追踪的情况

[medium] Order dependencies

- 依赖变更顺序的有状态条件

[low] Defensive complexity

- 防御性空链（如 x and x.y and x.y.z → 可选链或空对象）
  </violations>

<exceptions>
用于简单开/关状态的单个布尔值。Builder 模式标志。
</exceptions>

<threshold>
当标志以需要精神状态追踪的方式相互作用时标记。独立标志是可以的。
</threshold>

## 4. Dependency Injection

<principle>
业务逻辑应在不需要网络、磁盘或数据库的情况下可测试。硬编码依赖使代码不可测试且紧耦合。
</principle>

Detect: 我能在不 mock 基础设施的情况下独立测试这个函数吗？依赖是注入的还是硬编码的？

<grep-hints>
模式指示词（起点，非定论）：
`datetime.now`, `time.time`, `os.environ`, `open(`, `requests.`, `http.`
</grep-hints>

<violations>
说明性模式（非穷举——类似违规也存在）：

[high] Untestable coupling

- 硬编码依赖（如 new Date() 内联 → 注入时钟）
- 全局状态访问（避免或注入）
- 任何需要基础设施才能测试的业务逻辑

[medium] Mixed concerns

- 副作用与计算混合（将纯逻辑与副作用分离）
- 具体类依赖（依赖接口而非实现）

[low] Configuration coupling

- 环境耦合（直接读取环境变量 → 注入配置）
- 时间依赖逻辑（注入时钟以便测试）
  </violations>

<exceptions>
组装依赖的入口点。测试工具。直接运行的脚本。
</exceptions>

<threshold>
当不可测试的代码在业务逻辑中时标记。边界处的基础设施代码有依赖是正常的。
</threshold>

## 5. Definition Locality

<principle>
组件的定义应在单一位置完整呈现。当理解组件**是什么**——其标识、需求、约束和行为——需要阅读多处时，定义就是分散的。
</principle>

Detect: 要理解这个组件**是什么**，我需要阅读多少处？如果我修改该组件的需求，需要编辑多少文件？

<grep-hints>
结构指示词（起点，非定论）：
同一需求在 2 处以上检查，组件标识跨文件分散，带默认值的提取模式（args.get, kwargs.get, getattr with default）
</grep-hints>

<violations>
说明性模式（非穷举——类似违规也存在）：

[high] Scattered specification

- 同一需求在 2 处以上声明（如解析器标记为必填 AND 处理器检查是否缺失）
- 组件标识跨文件分散且无明确所有权
- 需要从 3+ 个来源「精神重组」的定义

[medium] Split declaration/enforcement

- 接口在一处声明，在另一处验证，且无共享引用
- 默认值与 schema 分开定义（如类型在 schema 中，默认值在代码中）
- 同一约束在多处检查
  </violations>

<exceptions>
依赖注入（注入协作者的定义在协作者处，不在这里——这是运行时组装，不是分散）。组合（A 使用 B；B 的定义是 B 的事）。继承（有意的分解）。插件架构（明确的所有权边界）。注册表 + 引用模式（定义一次，多处引用——这是修复方案，不是问题）。
</exceptions>

<threshold>
当组件定义在 2 处以上分散且无明确所有权时标记。关键测试：谁拥有这个事实？若所有权不清或重复，即为分散。LLM 生成的代码中常见。
</threshold>

## 6. Error Handling

<principle>
错误应保留上下文并到达适当的处理者。吞掉错误或使用通用 catch 会丢失信息；错误在错误抽象层出现会使调用方困惑。
</principle>

Detect: 如果这个操作失败会怎样？错误信息是否被保留并正确路由？

<grep-hints>
模式指示词（起点，非定论）：
`except:`, `catch (`, `catch(`, `pass`, `# TODO`, `raise Error(`
</grep-hints>

<violations>
说明性模式（非穷举——类似违规也存在）：

[high] Information loss

- 吞掉异常（空 catch 块）
- 通用 catch（如 catch Exception → 捕获特定错误）
- 任何丢失诊断信息的错误处理

[medium] Wrong abstraction

- 错误在错误的抽象层（低层错误泄露给调用方）

[low] Missing context

- raise Error('failed') → raise Error(f'order {id}: {reason}')
  </violations>

<exceptions>
顶层带日志记录的通用 catch。有注释说明的、故意吞掉的预期错误。
</exceptions>

<threshold>
当错误处理掩盖或丢失信息时标记。带日志记录的文档化 catch-all 是可以接受的。
</threshold>
