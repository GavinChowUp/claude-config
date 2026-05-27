# QR 严重度分类

## 严重度级别 (MoSCoW)

| 级别   | 含义                     | 渐进降级                  |
| ------ | ------------------------ | ------------------------- |
| MUST   | 遗漏后不可恢复           | 所有迭代                  |
| SHOULD | 可维护性债务             | 迭代 1-4                  |
| COULD  | 可自动修复，影响低       | 迭代 1-3                  |

## 按可恢复性分类

### KNOWLEDGE (MUST)

知识损失是永久性的，**始终**阻塞。

| 类别                   | 检测方式                                           |
| ---------------------- | -------------------------------------------------- |
| DECISION_LOG_MISSING   | 非平凡选择未记录原理                               |
| POLICY_UNJUSTIFIED     | 策略默认值缺乏 Tier 1 支持                         |
| IK_TRANSFER_FAILURE    | 隐性知识未在最佳位置                               |
| TEMPORAL_CONTAMINATION | 注释中含变更相对语言                               |
| BASELINE_REFERENCE     | 注释引用了已删除/已替换的代码                      |
| ASSUMPTION_UNVALIDATED | 架构假设无引用佐证                                 |
| LLM_COMPREHENSION_RISK | 会让未来 LLM 困惑的模式                            |
| MARKER_INVALID         | intent marker 缺少有效说明                         |

### STRUCTURE (SHOULD)

可维护性债务，会累积但事后可发现。

| 类别                        | 检测方式                                         |
| --------------------------- | ------------------------------------------------ |
| GOD_OBJECT                  | 方法 >15 OR 依赖 >10 OR 关注点混杂               |
| GOD_FUNCTION                | 代码行 >50 OR 抽象层混杂 OR 嵌套 >3              |
| DUPLICATE_LOGIC             | 复制粘贴块、并行函数                             |
| INCONSISTENT_ERROR_HANDLING | 同模块内混用异常/错误码                          |
| CONVENTION_VIOLATION        | 违反已记录的项目规范                             |
| TESTING_STRATEGY_VIOLATION  | 测试不符合已确认的策略                           |

### DIAGRAM (MUST for semantic, COULD for format)

图表图结构完整性。语义问题阻塞；格式问题警告。

| 类别                 | 严重度   | 检测方式                                       |
| -------------------- | -------- | ---------------------------------------------- |
| ORPHAN_NODE          | MUST     | 节点零边                                       |
| INVALID_EDGE_REF     | MUST     | 边的源/目标引用了不存在的节点                  |
| INVALID_SCOPE_REF    | MUST     | 范围引用了不存在的里程碑                       |
| DIAGRAM_WIDTH_EXCEED | COULD    | ASCII 渲染行超过 80 字符                       |
| UNCLOSED_BOX         | COULD    | ASCII 渲染中框角未对齐                         |

### COSMETIC (COULD)

可自动修复，影响极小。

| 类别                | 检测方式                                                    |
| ------------------- | ----------------------------------------------------------- |
| DEAD_CODE           | 未使用的函数、不可达分支                                    |
| FORMATTER_FIXABLE   | 可由格式化工具/linter 修复的风格问题                        |
| MINOR_INCONSISTENCY | 不符合已记录规则的情况                                      |
| TOOLCHAIN_CATCHABLE | 计划代码中编译器/linter/解释器会标记的错误，且正确代码从上下文明显可知（拼写错误、缺少导入、非穷举匹配）。不包括：揭示计划层面理解偏差的错误——那些是 ASSUMPTION_UNVALIDATED（MUST） |

## IK 邻近原则

隐性知识必须在**最佳位置**：「尽可能靠近相关位置，但不超过」

| 知识类型       | 最佳位置                                |
| -------------- | --------------------------------------- |
| 接受的风险     | 被标记代码位置的 :TODO: 注释            |
| 架构           | **同一目录**下的 README.md              |
| 取舍           | 决策体现处的代码注释                    |
| 不变量         | 执行点处的代码注释                      |

位置错误 = IK_TRANSFER_FAILURE（MUST）
