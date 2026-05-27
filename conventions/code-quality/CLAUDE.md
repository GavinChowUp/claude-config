# code-quality/

面向 LLM 辅助开发的代码质量检查，按认知模式分类组织。

## 文件

| 文件                               | 内容                                               | 何时阅读                                                   |
| ---------------------------------- | -------------------------------------------------- | ---------------------------------------------------------- |
| `README.md`                        | 格式设计原理、集成方式、隐性知识说明               | 理解文档格式、修改分类时                                   |
| `01-naming-and-types.md`           | 名称与类型的意图表达                               | 理解命名、领域建模、类型设计时                             |
| `02-structure-and-composition.md`  | 代码结构与组合                                     | 理解函数组合、控制流、错误处理时                           |
| `03-patterns-and-idioms.md`        | 惯用模式                                           | 理解表达式模式、现代语言惯用法、死代码时                   |
| `04-repetition-and-consistency.md` | DRY 原则与一致性                                   | 理解代码重复、验证逻辑、业务规则时                         |
| `05-documentation-and-tests.md`    | 文档与测试                                         | 理解文档规范、测试规范、schema 一致性时                    |
| `06-module-and-dependencies.md`    | 模块边界                                           | 理解模块结构、架构设计时                                   |
| `07-cross-file-consistency.md`     | 跨文件一致性                                       | 理解接口、命名、错误处理的跨文件一致性时                   |
| `08-codebase-patterns.md`          | 全代码库模式                                       | 理解可理解性、抽象机会时                                   |

## 适用性速查表

| 文档                          | Design Review | Diff Review | Codebase Review | Refactor Design | Refactor Code |
| ----------------------------- | :-----------: | :---------: | :-------------: | :-------------: | :-----------: |
| 01-naming-and-types           |      Yes      |     Yes     |       Yes       |       Yes       |      Yes      |
| 02-structure-and-composition  |      Yes      |     Yes     |       Yes       |       Yes       |      Yes      |
| 03-patterns-and-idioms        |      No       |     Yes     |       Yes       |       No        |      Yes      |
| 04-repetition-and-consistency |      No       |     Yes     |       Yes       |       No        |      Yes      |
| 05-documentation-and-tests    |      No       |     Yes     |       Yes       |       No        |      Yes      |
| 06-module-and-dependencies    |      Yes      |     No      |       Yes       |       Yes       |      Yes      |
| 07-cross-file-consistency     |      Yes      |     No      |       Yes       |       Yes       |      Yes      |
| 08-codebase-patterns          |      No       |     No      |       Yes       |       No        |      Yes      |
