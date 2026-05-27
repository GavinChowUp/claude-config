# conventions/

面向 agent 和 skill 的通用规范。

## 文件

| 文件                | 内容                                      | 何时阅读                                                  |
| ------------------- | ----------------------------------------- | --------------------------------------------------------- |
| `documentation.md`  | CLAUDE.md/README.md 格式规范              | 编写 CLAUDE.md、创建 README.md、使用 doc-sync skill 时   |
| `intent-markers.md` | :PERF:/:UNSAFE:/:SCHEMA: 标记规范        | 添加 intent marker、QR 验证标记时                         |
| `severity.md`       | MUST/SHOULD/COULD 严重度定义              | 理解 QR 严重度、编写 QR 脚本时                            |
| `structural.md`     | 代码质量规范、测试规则                    | QR 代码审查、planner 决策审计时                           |
| `temporal.md`       | 注释的永恒现在时规则                      | TW/QR 时间污染检查、编写注释时                            |
| `diff-format.md`    | 代码变更的统一 diff 规范                  | 编写代码 diff、Developer/QR diff 验证时                   |

## 子目录

| 目录            | 内容                                     | 何时阅读                                                         |
| --------------- | ---------------------------------------- | ---------------------------------------------------------------- |
| `code-quality/` | 基线/拆分/漂移质量检查                   | 设计、代码审查、重构、规划阶段质量验证时                         |
