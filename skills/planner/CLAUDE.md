# planner/

带质量门的规划与执行 skill。

## 文件

| 文件        | 内容                               | 何时阅读                     |
| ----------- | ---------------------------------- | ---------------------------- |
| `SKILL.md`  | Skill 激活与调用方式               | 使用 planner skill 时        |
| `INTENT.md` | 权威设计规范                       | 理解系统设计时               |
| `README.md` | 架构、工作流、设计理由             | 理解 planner 设计时          |

## 子目录

| 目录         | 内容                   | 何时阅读                            |
| ------------ | ---------------------- | ----------------------------------- |
| `resources/` | 计划格式、diff 规范    | 编辑计划结构或 diff 格式时          |
| `architect/` | 计划设计子 agent       | 理解规划工作流时                    |

Python 代码:`scripts/skills/planner/`(planner.py、executor.py、explore.py、qr/、tw/、dev/)

## 通用约定

脚本从 `.claude/conventions/` 读取以下约定:

| 约定文件            | 何时阅读                                         |
| ------------------- | ------------------------------------------------ |
| `documentation.md`  | 理解 CLAUDE.md/README.md 格式时                  |
| `structural.md`     | 更新 QR RULE 2 或 planner 决策审计时             |
| `temporal.md`       | 更新 TW/QR 时态污染逻辑时                        |
| `severity.md`       | 理解 QR 严重级别时                               |
| `intent-markers.md` | 理解 `:PERF:`/`:UNSAFE:` 标记时                  |
