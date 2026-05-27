<!-- applicable_phases: design_review, codebase_review, refactor_design, refactor_code -->

# Cross-File Consistency

评估文件间模式是否一致。

**核心问题**：跨文件是否一致？相似的 API 应当行为相似。同一概念在整个代码库中应只有一个名称。错误处理在每个抽象层应当是可预期的。功能标志应当以一致的逻辑判断。

**关注点**：

- 跨模块命名漂移（同一概念用 userId/uid/id）
- 跨模块相似操作的不兼容签名
- 跨抽象层的错误模式不一致
- 同一功能标志在不同地方以不同逻辑判断

**门槛**：当不一致对使用者造成困惑或不可预测时标记。当同一概念在跨模块有多个名称**且**导致集成困惑时标记。此组需要同时查看多个文件才能检测到模式。

<design-mode>
评估代码意图时（Design Review 阶段）：

- 提出的 API 是否与现有类似 API 一致？
- 是否为已有概念引入了新名称？
- 错误处理是否与该层级的其他组件一致？

证据格式：引用代码意图描述，指出不一致之处。
</design-mode>

<code-mode>
评估实际代码时（Codebase Review、Refactor）：

- 类似操作是否使用不同约定？
- 同一概念是否在不同模块有不同名称？
- 类似错误是否在同一层级以不同方式处理？

证据格式：引用来自多个文件的代码，指出不一致之处。
</code-mode>

---

## 1. Interface Consistency

<principle>
相似的 API 应有一致的签名。当类似函数以不同约定使用者，会导致 bug。
</principle>

Detect: 这些 API 的使用者是否会因不一致而感到惊讶？类似操作是否有不兼容的签名？

<grep-hints>
模式指示词（起点，非定论）：
参数顺序不同的相似函数签名，CRUD 操作模式，服务方法签名
</grep-hints>

<violations>
说明性模式（非穷举——类似违规也存在）：

[high] Signature inconsistency

- 目的相似的 API 有不兼容的签名**且**共享使用者
- 任何导致调用方困惑的 API 不一致

[medium] Naming inconsistency

- 相关函数间不一致的命名约定

[low] Pattern inconsistency

- 类似操作混用同步/异步而无明确原因
  </violations>

<exceptions>
有意的 API 差异。领域特定约定。版本化 API。目的明确不同的重载。
</exceptions>

<threshold>
当 2 个以上相似函数有不同参数顺序（文件范围）或 3 个以上 API 有不兼容签名（代码库范围）**且**困惑影响使用者时标记。
</threshold>

## 2. Naming Consistency (Cross-File Scope)

<principle>
一个概念在整个代码库中应只有一个名称。同一事物有多个名称会让人困惑：它们究竟是不是同一个东西？
</principle>

Detect: 是否有多个名称在跨模块指向同一概念？读者是否会疑惑 userId 和 uid 是否指同一实体？

<grep-hints>
模式指示词（起点，非定论）：
跨模块作为变量前缀的同义词（user/account/customer，config/settings/options，id/uid/identifier）
</grep-hints>

<violations>
说明性模式（非穷举——类似违规也存在）：

[high] Semantic confusion

- 同义词漂移导致集成点产生困惑
- 任何导致跨模块对身份产生疑问的命名不一致

[medium] Inconsistent conventions

- 跨模块缩写不一致（如 userId vs uid vs id）

[low] Style drift

- 无语义混乱的风格不一致
  </violations>

<exceptions>
真正不同概念的不同名称。外部 API 命名约定。领域特定术语。有界迁移中的遗留兼容别名。
</exceptions>

<threshold>
当同一语义概念在 3 个以上模块有不同名称**且**导致对其是否指同一事物产生困惑时标记。
</threshold>

## 3. Error Pattern Consistency (Cross-File Scope)

<principle>
同一抽象层内的错误处理应当一致。混用模式会导致调用方对错误如何传播和应如何处理产生困惑。
</principle>

Detect: 同一抽象层的组件间错误处理是否一致？调用方能否预期类似操作的行为？

<grep-hints>
模式指示词（起点，非定论）：
混用异常/返回码模式，跨模块不一致的错误消息格式，变化的错误上下文
</grep-hints>

<violations>
说明性模式（非穷举——类似违规也存在）：

[high] Incompatible patterns

- 跨组件类似操作使用不兼容的错误模式
- 任何在集成边界造成调用方困惑的错误处理

[medium] Inconsistent hierarchy

- 同一抽象层不一致的异常层级

[low] Missing convention

- 跨模块错误上下文/包装没有标准
  </violations>

<exceptions>
不同抽象层使用不同模式（领域 vs API vs 基础设施）。在错误风格间转换的包装函数。处于积极迁移中的遗留代码。
</exceptions>

<threshold>
当同一抽象层在文件间对类似操作使用 3 种以上不兼容的错误模式**且**没有迁移计划时标记。
</threshold>

## 4. Feature Flag Sprawl

<principle>
功能标志应当以一致的方式判断。当同一标志在不同地方以不同逻辑评估时，行为变得不可预测。
</principle>

Detect: 功能标志在整个代码库中如何判断？同一标志在所有地方的评估是否一致？

<grep-hints>
结构指示词（起点，非定论）：
功能标志检查，toggle 模式，条件特性代码
</grep-hints>

<violations>
说明性模式（非穷举——类似违规也存在）：

[high] Inconsistent evaluation

- 功能标志判断不一致（同一标志使用不同条件）
- 任何跨位置评估逻辑出现分歧的标志

[medium] Undocumented dependencies

- 标志依赖未记录（标志 A 需要标志 B）
  </violations>

<exceptions>
有意在不同上下文中行为不同的标志。A/B 测试变体。逐步上线逻辑。
</exceptions>

<threshold>
当同一功能标志在不同地方以不同逻辑判断**且**差异是无意的时标记。
</threshold>

注意：死标志（特性已上线但标志从未移除）在 08-codebase-patterns.md Zombie Code（Codebase Scope）中处理。
