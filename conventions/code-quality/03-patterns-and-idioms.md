<!-- applicable_phases: diff_review, codebase_review, refactor_code -->

# Patterns & Idioms

评估代码是否使用了语言的惯用模式。

**核心问题**：这符合惯用法吗？现代语言提供了简化常见模式的特性。当代码使用过时模式、冗长反模式或不必要的复杂表达式时，会在没有任何收益的情况下增加认知负担。

**关注点**：

- 需要精神推演的复杂布尔表达式
- 有更简单等价形式的冗长条件模式
- 过时的迭代/回调模式
- 注释掉的代码块和不可达分支（文件范围内）
- 本可简化代码但未使用的语言特性

**门槛**：标记机械反模式和掩盖意图的表达式级复杂性。有充分注释的复杂逻辑是可以接受的；不必要的复杂逻辑则不行。仅当项目语言版本中存在明显更好的现代惯用法时才标记过时模式。

<design-mode>
不适用——此组需要实际代码才能评估。
</design-mode>

<code-mode>
评估实际代码时（Diff Review、Codebase Review、Refactor）：

- 布尔表达式是否一目了然？
- 条件是否使用了更简单的等价形式？
- 是否利用了现代语言特性？
- 注释掉的代码是否在占用文件空间？

证据格式：引用代码（含 file:line），指出问题所在。
</code-mode>

---

## 1. Boolean Expression Complexity

<principle>
布尔表达式应当一目了然。如果需要精神推演才能理解，就需要简化或命名。
</principle>

Detect: 我能不经过脑力追踪就理解这个布尔表达式吗？

<grep-hints>
模式指示词（起点，非定论）：
`and.*and`, `or.*or`, `&&.*&&`, `||.*||`, `not.*not`, `!.*!`
</grep-hints>

<violations>
说明性模式（非穷举——类似违规也存在）：

[medium] Cognitive overload

- 多子句表达式（3+ 个 AND/OR 项 → 提取命名谓词）
- 否定复合条件（如 not (a and b) → 更清晰的正向形式）
- 任何需要纸笔/精神推演才能求值的表达式

[low] Ambiguity

- 混用 AND/OR 而无括号澄清优先级
- 双重/三重否定（如 if not disabled，if not is_invalid）
  </violations>

<exceptions>
结构清晰且有注释解释逻辑的复杂条件。
</exceptions>

<threshold>
当表达式需要精神推演才能理解时标记。有充分注释的复杂条件是可以接受的。
</threshold>

## 2. Conditional Anti-Patterns

<principle>
条件应当直接表达意图。当存在保留相同含义的更简单形式时，复杂形式就是反模式。
</principle>

Detect: 是否存在保留相同含义的更简单方式来表达这个条件？

<grep-hints>
模式指示词（起点，非定论）：
`if.*return True.*else.*return False`, `try:.*except:.*pass`, `and do_`
</grep-hints>

<violations>
说明性模式（非穷举——类似违规也存在）：

[medium] Verbose patterns

- if cond: return True else: return False（直接 return cond）
- 基于异常的控制流（try/except 当作 if/else）
- 任何有更简单等价形式的条件

[low] Subtle complexity

- 短路副作用（如 cond and do_thing()）
- 没有明确收益的 Yoda 条件（如 if 5 == x）
  </violations>

<exceptions>
用于真正异常情况的异常处理。用于延迟求值的短路。
</exceptions>

<threshold>
仅标记机械反模式。保留意图的变体是风格偏好。
</threshold>

## 3. Modern Idioms

<principle>
现代语言特性的存在就是为了简化常见模式。当旧模式不必要地持续存在时，会在没有收益的情况下增加认知负担。
</principle>

Detect: 是否存在能简化此代码的新语言特性？项目语言版本是否被充分利用？

<grep-hints>
模式指示词（起点，非定论）：
`for i in range(len(`, `+ str(`, `.format(`, 回调模式, `null` 检查
</grep-hints>

<violations>
说明性模式（非穷举——类似违规也存在）：

[medium] Outdated patterns

- 旧式迭代模式（如手动索引循环 → for-each, enumerate）
- 已废弃的 API 用法
- 任何有更简单现代等价形式的模式

[low] Missing features

- 缺少语言特性（如无解构、无模式匹配）
- 遗留模式（如回调 → async/await）
- 过时惯用法（如字符串拼接 → f-string/模板）
- 手动空值检查（→ 可选链、空值合并）
  </violations>

<exceptions>
出于兼容性故意使用旧模式。避免分配的性能关键代码。
</exceptions>

<threshold>
仅当现代惯用法明显更好**且**在项目语言版本中可用时标记。不标记风格偏好。
</threshold>

## 4. Readability

<principle>
代码应当能独立理解。当理解需要查阅外部资料或依赖部落知识时，代码需要澄清。
</principle>

Detect: 我能不读其他文件或询问他人就理解这段代码吗？意图是否从代码本身就清晰可见？

<grep-hints>
模式指示词（起点，非定论）：
函数调用中的布尔字面量，魔法数字，无解释的常量
</grep-hints>

<violations>
说明性模式（非穷举——类似违规也存在）：

[high] Obscured intent

- 布尔陷阱（如 fn(True, False) → fn(enabled=True, debug=False)）
- 任何需要查阅函数签名才能理解参数含义的调用

[medium] Magic values

- 魔法数字/字符串（如 42 → MAX_RETRIES = 42）
- 使用命名参数更清晰意图时却用了位置参数

[low] Dense expressions

- 密集表达式（如嵌套三元 → 命名中间变量）
- 非显然决策缺少 WHY 注释
- 调用间的隐式顺序依赖（记录或使其显式）
  </violations>

<exceptions>
众所周知的常量（0、1、-1、100）。名称明显的函数中的布尔值（如 setEnabled(true)）。
</exceptions>

<threshold>
当含义需要查阅外部资料时标记。不言自明的代码无需注释。
</threshold>

## 5. Zombie Code (File Scope)

<principle>
死代码是误导读者的噪音。无法执行或从未被调用的代码应该删除，而非留下来迷惑未来的维护者。
</principle>

Detect: 如果我删除这段代码，任何测试会失败或行为会改变吗？

<grep-hints>
模式指示词（起点，非定论）：
注释掉的代码块，`#if 0`，不可达分支，未使用的变量
</grep-hints>

<violations>
说明性模式（非穷举——类似违规也存在）：

[high] Dead code blocks

- 注释掉的代码块（>5 行代码，不是文档）
- 不可达分支（如无条件 return 后的 else，死 switch case）
- 任何无法执行的代码

[medium] Unused declarations

- 未使用的局部变量或参数

[low] Orphaned functions

- 文件内定义但从未调用的函数
  </violations>

<exceptions>
带说明的注释代码（调试辅助）。接口契约要求的未使用参数。公共 API 入口点。插件接口。
</exceptions>

<threshold>
当代码明显不可达/未使用**且**不是公共 API 入口点、插件接口或有文档记录的调试辅助时标记。
</threshold>
