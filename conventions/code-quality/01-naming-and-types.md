<!-- applicable_phases: design_review, diff_review, codebase_review, refactor_design, refactor_code -->

# Naming & Types

评估名称和类型是否准确传达意图。

**核心问题**：如果读者只看名称或类型，其心理模型是否与实际行为一致？名称是微型文档。类型是契约。任何一方出现谎言，读者就会建立错误的心理模型并引入 bug。

**关注点**：

- 描述 HOW 而非 WHAT 的名称
- 撒谎的动词（get 却有变更操作，validate 却做解析）
- 缺失的领域类型（本该有概念的地方用了原始类型）
- 基于类型的分支（isinstance 链，说明缺少多态性）
- 同一文件中对同一概念使用多个名称

**门槛**：仅在名称/类型存在误导性，或领域概念隐藏在跨越边界的原始类型中时标记。不完美但准确的名称是风格偏好，不是质量问题。

<design-mode>
评估代码意图时（Design Review 阶段）：

- 提出的函数/类名称是否能预测其行为？
- 意图是否使用领域类型而非原始类型？
- 类型选择是否适合领域概念？

证据格式：引用代码意图描述，指出命名/类型问题。
</design-mode>

<code-mode>
评估实际代码时（Diff Review、Codebase Review、Refactor）：

- 实现的名称是否与实际行为一致？
- 领域概念是否隐藏在原始类型比较中？
- isinstance 链是否说明缺少多态性？

证据格式：引用代码（含 file:line），指出问题所在。
</code-mode>

---

## 1. Naming Precision

<principle>
名称是微型文档。它应当足够准确地预测行为，使读者阅读实现时得到印证而非惊讶。
</principle>

Detect: 名称是否准确描述了所做的事？仅凭名称建立的心理模型，是否与实际行为一致？

<grep-hints>
可能表示命名问题的词项（起点，非定论）：
`Manager`, `Handler`, `Utils`, `Helper`, `Data`, `Info`, `process`, `handle`, `do`
</grep-hints>

<violations>
说明性模式（非穷举——类似违规也存在）：

[high] Name-behavior mismatch

- 描述 HOW 而非 WHAT 的名称（如 loopOverItems → processOrders）
- 撒谎的动词（如 get 却有变更操作，validate 却做解析）
- 任何读到实现时令人惊讶的名称

[medium] Abstraction leakage

- 公共 API 名称中暴露实现细节
- 模糊的大伞词（如 Manager, Handler, Utils, Helper, Data, Info）

[low] Cognitive friction

- 否定式布尔值（如 isNotValid → isInvalid，disableFeature → featureEnabled）
  </violations>

<exceptions>
真正通用上下文中的通用名称（如泛型集合中的 item，类型参数中的 T）。测试：具体名称是否增加了信息量，或只是噪音？
</exceptions>

<threshold>
仅当名称存在主动误导时标记。不完美但仍然准确的名称是风格偏好。
</threshold>

## 2. Missing Domain Modeling

<principle>
领域概念应在代码中明确表达，而非隐藏在原始类型比较中。当同一概念以多种方式检查时，它属于领域对象。
</principle>

Detect: 领域概念是否隐藏在原始条件中？同一业务概念是否在多处通过原始类型比较来检查？

<grep-hints>
模式指示词（起点，非定论）：
`== 'admin'`, `== "admin"`, `status ==`, `role ==`, `type ==`, 魔法数字
</grep-hints>

<violations>
说明性模式（非穷举——类似违规也存在）：

[high] Hidden domain logic

- 原始条件中的领域谓词（如 user.role == 'admin' → user.can_edit()）
- 魔法值比较（如 status == 3 → Status.APPROVED）
- 任何只通过原始类型比较表达的业务概念

[medium] Implicit modeling

- 字符串比较表示状态（如 mode == 'active' → 枚举）
- 业务规则埋藏在条件中（提取到领域对象方法）
  </violations>

<exceptions>
领域层实现本身中的显式比较。启动时只比较一次的配置值。
</exceptions>

<threshold>
当同一领域概念在 2 处以上通过原始类型比较来检查时标记。
</threshold>

## 3. Type-Based Branching

<principle>
分散在代码中的类型派发表明缺少多态性。当在多处基于类型分支时，类型本身应承载行为。
</principle>

Detect: 是否在用类型检查做本应由多态性处理的事？同一类型派发是否在多处出现？

<grep-hints>
模式指示词（起点，非定论）：
`isinstance`, `typeof`, `instanceof`, `hasattr`, `in dict`, `.type ==`
</grep-hints>

<violations>
说明性模式（非穷举——类似违规也存在）：

[high] Scattered dispatch

- isinstance/typeof 链（3+ 个分支 → 多态性候选）
- 同一类型派发在多处出现

[medium] Implicit dispatch

- 属性存在性检查（如 hasattr/in dict 作为类型派发）

[low] Missing abstraction

- 应成为 protocol/interface 的鸭子类型条件
  </violations>

<exceptions>
用于输入验证的单个 isinstance 检查。用于类型安全的类型收窄。
</exceptions>

<threshold>
当同一类型派发出现在 2 处以上时标记。单次类型检查通常是合适的。
</threshold>

## 4. Type Design

<principle>
领域概念值得拥有自己的类型。未经验证跨越 API 边界的原始类型会引入 bug；带验证的值对象则能防止 bug。
</principle>

Detect: 哪些领域概念被表示为原始类型？原始类型是否在未经验证的情况下跨越 API 边界？

<grep-hints>
模式指示词（起点，非定论）：
ID 用 `str`，金额用 `float`，`dict` 在调用链中传递，`Any`，`object`
</grep-hints>

<violations>
说明性模式（非穷举——类似违规也存在）：

[high] Missing domain types

- 原始类型滥用（如 userId 为 string → 带验证的 UserId 类型）
- 缺少值对象（如 money 为 float → Money(amount, currency)）
- 任何以原始类型跨越 API 边界的领域概念

[medium] Weak typing

- 字符串化数据（JSON 字符串 → 有类型的对象）
- 泄漏的抽象（调用方必须了解实现细节）

[low] Type proliferation

- Optional 爆炸（大量可空字段 → 考虑为不同状态建立独立类型）
  </violations>

<exceptions>
内部实现中的原始类型。序列化边界。性能关键路径。
</exceptions>

<threshold>
当原始类型在未经验证的情况下跨越 API 边界时标记。内部使用原始类型是可以接受的。
</threshold>

## 5. Naming Consistency (File Scope)

<principle>
一个概念在文件内应只有一个名称。同一事物有多个名称会让人困惑：它们究竟是不是同一个东西？
</principle>

Detect: 同一文件内是否有多个名称指向同一概念？读者是否会疑惑 user 和 account 是否为同一实体？

<grep-hints>
模式指示词（起点，非定论）：
作为变量前缀的同义词（user/account/customer，config/settings/options，id/uid/identifier）
</grep-hints>

<violations>
说明性模式（非穷举——类似违规也存在）：

[high] Semantic confusion

- 同一实体在同一文件中有不同名称（如 user vs account vs customer）
- 任何在单个文件内对身份产生疑问的命名不一致

[medium] Inconsistent conventions

- 文件内缩写不一致（如 id vs identifier）

[low] Style drift

- 无语义混乱的风格不一致
  </violations>

<exceptions>
真正不同概念的不同名称。外部 API 命名约定。在特定范围内为清晰而使用的别名。
</exceptions>

<threshold>
当同一语义概念在文件内有多个名称**且**导致对其是否指同一事物产生困惑时标记。
</threshold>
