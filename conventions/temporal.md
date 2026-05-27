# Temporal Contamination in Code Comments

本文档定义了识别泄露代码历史、变更过程或规划制品信息的注释所用术语。@agent-technical-writer 和 @agent-quality-reviewer 均参考本规范。

## The Core Principle

> **永恒现在时规则**：注释必须从一个首次接触代码、对其前身或演变一无所知的读者角度来编写。代码就是**存在**的。

**为什么这很重要**：变更叙事注释是 LLM 的产物——一种范畴错误，不只是风格问题。变更过程是短暂的，与代码的持续存在无关。人类自然描述代码**是什么**，而非他们**做了什么**来创建它。引用创建注释的变更，从根本上就混淆了文档应该记录什么。

这样想：小说的叙述者从不描述作者的打字过程。同理，代码注释也不应描述开发者的编辑过程。代码就存在在那里；它如何到达那里是不可见的。

在计划中，这意味着注释**如同计划已经执行完毕**一样来编写。

## Detection Heuristic

对照以下五个问题评估每条注释。信号词只是示例——请举一反三，推断语义上类似的结构。

### 1. 描述的是采取的行动，而非当前存在的内容？

**Category**: Change-relative

| Contaminated                           | Timeless Present                                            |
| -------------------------------------- | ----------------------------------------------------------- |
| `// Added mutex to fix race condition` | `// Mutex serializes cache access from concurrent requests` |
| `// New validation for the edge case`  | `// Rejects negative values (downstream assumes unsigned)`  |
| `// Changed to use batch API`          | `// Batch API reduces round-trips from N to 1`              |

信号词（非穷举）：「Added」、「Replaced」、「Now uses」、「Changed to」、「New」、「Updated」、「Refactored」

### 2. 与代码中不存在的内容进行比较？

**Category**: Baseline reference

| Contaminated                                      | Timeless Present                                                    |
| ------------------------------------------------- | ------------------------------------------------------------------- |
| `// Replaces per-tag logging with summary`        | `// Single summary line; per-tag logging would produce 1500+ lines` |
| `// Unlike the old approach, this is thread-safe` | `// Thread-safe: each goroutine gets independent state`             |
| `// Previously handled in caller`                 | `// Encapsulated here; caller should not manage lifecycle`          |

信号词（非穷举）：「Instead of」、「Rather than」、「Previously」、「Replaces」、「Unlike the old」、「No longer」

### 3. 描述代码放在哪里，而非代码做什么？

**Category**: Location directive

| Contaminated                  | Timeless Present                              |
| ----------------------------- | --------------------------------------------- |
| `// After the SendAsync call` | _(删除——diff 结构已编码位置)_                 |
| `// Insert before validation` | _(删除——diff 结构已编码位置)_                 |
| `// Add this at line 425`     | _(删除——diff 结构已编码位置)_                 |

信号词（非穷举）：「After」、「Before」、「Insert」、「At line」、「Here:」、「Below」、「Above」

**处理**：始终删除。位置已编码在 diff 结构中，不在注释里。

### 4. 描述的是意图，而非行为？

**Category**: Planning artifact

| Contaminated                           | Timeless Present                                         |
| -------------------------------------- | -------------------------------------------------------- |
| `// TODO: add retry logic later`       | _(删除，或立即实现 retry)_                               |
| `// Will be extended for batch mode`   | _(删除——不记录假设性的未来)_                             |
| `// Temporary workaround until API v2` | `// API v1 lacks filtering; client-side filter required` |

信号词（非穷举）：「Will」、「TODO」、「Planned」、「Eventually」、「For future」、「Temporary」、「Workaround until」

**处理**：删除、实现该特性，或改写为当前约束。

### 5. 描述的是作者的选择，而非代码行为？

**Category**: Intent leakage

| Contaminated                               | Timeless Present                                     |
| ------------------------------------------ | ---------------------------------------------------- |
| `// Intentionally placed after validation` | `// Runs after validation completes`                 |
| `// Deliberately using mutex over channel` | `// Mutex serializes access (single-writer pattern)` |
| `// Chose polling for reliability`         | `// Polling: 30% webhook delivery failures observed` |
| `// We decided to cache at this layer`     | `// Cache here: reduces DB round-trips for hot path` |

信号词（非穷举）：「intentionally」、「deliberately」、「chose」、「decided」、「on purpose」、「by design」、「we opted」

**处理**：提取技术论证，丢弃决策叙事。读者不需要知道有人「决定了」——他们需要知道**为何**这种方案可行。

**测试**：删去意图词后注释是否仍有意义？若有，删去意图词。若无，围绕技术原因重写。

---

**兜底原则**：如果一条注释只对了解代码历史的人才有意义，它就是时间污染——即使不符合上述任何分类。

## Subtle Cases

相同词汇，不同判断——说明检测需要语义判断，而非关键词匹配。

| Comment                                | Verdict      | Reasoning                                          |
| -------------------------------------- | ------------ | -------------------------------------------------- |
| `// Now handles edge cases properly`   | Contaminated | 「properly」暗示之前是不正确的                     |
| `// Now blocks until connection ready` | Clean        | 「now」描述运行时时刻，而非代码历史                |
| `// Fixed the null pointer issue`      | Contaminated | 描述修复，而非行为                                 |
| `// Returns null when key not found`   | Clean        | 描述行为                                           |

## The Transformation Pattern

> **提取技术论证，丢弃变更叙事。**

1. 其中隐藏了什么有用信息？（问题、行为）
2. 改写为永恒现在时

示例：「Added mutex to fix race」→「Mutex serializes concurrent access」
