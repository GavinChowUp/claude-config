<!-- applicable_phases: diff_review, codebase_review, refactor_code -->

# Repetition & Consistency

评估代码是否遵循 DRY 原则并保持一致性。

**核心问题**：这是否 DRY 且一致？当相同的逻辑、验证或模式出现在多处时，bug 必须在所有地方修复——但实际上不会。当类似的操作使用不同模式时，读者会质疑差异是否有意义。

**关注点**：

- 需要多处修改才能修复的重复代码块
- 多次实现的验证规则
- 分散在各处的业务规则
- 重复的布尔表达式
- 同一文件或类中不一致的错误处理

**门槛**：当重复是无意的且需要协调变更时标记。当不一致导致对差异是否有意义产生困惑时标记。为模块化或有界上下文隔离而有意重复是可以接受的。

<design-mode>
不适用——此组需要实际代码才能评估。
</design-mode>

<code-mode>
评估实际代码时（Diff Review、Codebase Review、Refactor）：

- 修复 bug 是否需要修改多处？
- 验证/业务规则是否重复？
- 类似操作的处理是否不一致？
- 重复的模式是否需要提取？

证据格式：引用代码（含 file:line），指出重复/不一致所在。
</code-mode>

---

## 1. Duplication

<principle>
代码应有单一事实来源。当相同逻辑存在于多处时，bug 必须在所有地方修复——但实际上不会。
</principle>

Detect: 如果我在这里修复一个 bug，还需要在哪里修复？

<grep-hints>
结构指示词（起点，非定论）：
相同的多行代码块，相似的函数体，跨模块中名称暗示相似目的的函数
</grep-hints>

<violations>
说明性模式（非穷举——类似违规也存在）：

[high] Direct duplication

- 重复的相同代码块（3 行以上，逻辑性内容而非样板代码）
- 任何需要多处修复才能修复 bug 的逻辑

[medium] Near-duplication

- 带细微差异的复制粘贴

[low] Missed abstraction

- 未提取到共享位置的通用模式
  </violations>

<exceptions>
有意服务于不同目的的不同逻辑。测试 setup 代码。生成/vendor 代码。为模块化故意隔离的代码。不同有界上下文中的相似代码。
</exceptions>

<threshold>
当修复 bug 需要修改多处**且**重复是无意的时标记。
</threshold>

## 2. Validation Scattering

<principle>
验证规则应在一处存在。当相同的验证被多次实现时，实现会出现分歧——其中一些会出错。
</principle>

Detect: 这个验证是否重复了？更改验证规则是否需要修改多处？

<grep-hints>
模式指示词（起点，非定论）：
重复的正则模式，重复的边界检查，跨处的邮件/电话/格式验证
</grep-hints>

<violations>
说明性模式（非穷举——类似违规也存在）：

[high] Diverged validation

- 实现间的验证规则已出现分歧
- 任何需要多处更新的验证

[medium] Repeated validation

- 相同验证重复而无共享实现

[low] Defensive re-validation

- 在调用链深处重复防御性验证
  </violations>

<exceptions>
信任边界处的验证。设计上的纵深防御。特定于上下文的验证规则。服务边界验证。
</exceptions>

<threshold>
当相同验证出现 3 次以上（文件范围）或 5 个以上文件（代码库范围）**且**实现已出现或将出现分歧时标记。
</threshold>

## 3. Business Rule Scattering

<principle>
业务规则应有单一事实来源。当相同决策在多处做出时，它们最终会产生分歧。
</principle>

Detect: 这条规则的单一事实来源在哪里？如果规则改变，需要更新多少处？

<grep-hints>
模式指示词（起点，非定论）：
重复的条件模式，多处的魔法数字，定价/权限/资格逻辑
</grep-hints>

<violations>
说明性模式（非穷举——类似违规也存在）：

[high] Scattered decisions

- 相同业务决策在可能出现分歧的多处存在
- 任何没有明确单一事实来源的业务规则

[medium] Mixed concerns

- 业务逻辑与基础设施代码混合

[low] Implicit rules

- 规则嵌入在原始条件中而非命名谓词
  </violations>

<exceptions>
调用多个规则检查的编排代码。为服务隔离故意重复的规则。每租户/区域的规则变体。计算规则的缓存。
</exceptions>

<threshold>
当相同业务决策在 2 处以上（文件范围）或 3 个以上文件（代码库范围）做出**且**它们已出现或可能独立出现分歧时标记。
</threshold>

## 4. Condition Pattern Repetition

<principle>
重复的布尔表达式应成为命名谓词。当相同条件到处出现时，修改它需要找到所有出现处。
</principle>

Detect: 这个条件是否应该成为命名谓词？提取它是否能减少 bug 面？

<grep-hints>
模式指示词（起点，非定论）：
相同的布尔表达式，重复的保护子句，权限/功能标志检查模式
</grep-hints>

<violations>
说明性模式（非穷举——类似违规也存在）：

[high] High-frequency repetition

- 相同条件在 3 处以上（文件）或 5 个以上文件（代码库）出现（提取可减少 bug 面）
- 任何条件逻辑改变时需要多处更新的情况

[medium] Pattern repetition

- 重复的功能标志条件

[low] Guard repetition

- 相关函数中相同的保护子句模式
  </violations>

<exceptions>
标准保护子句（空检查、边界检查）。框架要求的模式。内联清晰易读的简单条件。
</exceptions>

<threshold>
当相同条件出现 3 次以上（文件范围）或 5 个以上文件（代码库范围）**且**提取为命名谓词会减少 bug 面时标记。
</threshold>

## 5. Error Pattern Consistency (File Scope)

<principle>
同一抽象层内的错误处理应当一致。混用模式会导致调用方对错误如何传播和应如何处理产生困惑。
</principle>

Detect: 这个文件或类中的错误处理是否一致？调用方能否预期类似操作的行为？

<grep-hints>
模式指示词（起点，非定论）：
混用异常/返回码模式，不一致的错误消息格式，变化的错误上下文
</grep-hints>

<violations>
说明性模式（非穷举——类似违规也存在）：

[high] Incompatible patterns

- 同一类中类似操作使用不兼容的错误模式
- 任何造成调用方困惑的错误处理

[medium] Inconsistent hierarchy

- 同一抽象层内不一致的异常层级

[low] Missing convention

- 文件内错误上下文/包装没有标准
  </violations>

<exceptions>
不同抽象层使用不同模式（领域 vs API vs 基础设施）。在错误风格间转换的包装函数。处于积极迁移中的遗留代码。
</exceptions>

<threshold>
当同一类对类似操作使用 2 种以上不兼容的错误模式**且**没有迁移计划时标记。
</threshold>
