# planner/

带 QR gate、TW 阶段与 Dev 执行的规划与执行工作流。状态文件由 LLM agent 管理，用于会话连续性。

## 文件

| 文件        | 内容                                                | 何时阅读                                     |
| ----------- | --------------------------------------------------- | ------------------------------------------------ |
| `README.md` | 架构、数据流、QR gate、设计决策 | 理解 planner 架构、QR 工作流时 |

## 共享文件

| 文件                    | 内容                                           | 何时阅读                         |
| ----------------------- | ---------------------------------------------- | ------------------------------------ |
| `shared/schema.py`      | Pydantic schema（context、plan、qr）及       | 理解状态文件 schema、    |
|                         | validate_state() 函数                      | 修改 schema 定义时         |
| `shared/constraints.py` | 编排器约束 AST 构建器               | 构建 planner/executor prompt、   |
|                         | （`build_orchestrator_constraint`、                  | 组合可复用约束块时 |
|                         | `build_step_header`、`build_state_banner`）         |                                      |
| `shared/gates.py`       | 统一 gate 输出构建器                    | 理解 QR gate 逻辑、         |
|                         | （`build_gate_output`）                          | 修改 gate 行为时            |

## 子目录

| 目录                | 内容                                   | 何时阅读                               |
| ------------------- | -------------------------------------- | ------------------------------------------ |
| `orchestrator/`     | 主工作流（planner、executor）     | 创建/执行计划时                   |
| `architect/`        | 计划设计子 agent                  | 理解规划工作流时            |
| `developer/`        | 代码填充与实现        | Dev 执行、生成 diff 时           |
| `technical_writer/` | 文档清理与生成 | TW 阶段、时态清理时     |
| `quality_reviewer/` | 所有阶段的 QR 模块              | QR 逻辑、验证、理解 gate 时  |
| `shared/`           | 共享资源、schema、约定 | 访问约定、资源管理时 |

## 状态文件

所有计划状态存储在 plan.json 中，上下文单独存入 context.json。完整 schema 见 README.md。

| 文件              | 内容                                    | 可变性              | 何时阅读                   |
| ----------------- | --------------------------------------- | ------------------- | ------------------------------ |
| `plan.json`       | 完整计划状态（milestones、diffs、 | 可变 -> 冻结   | 所有规划阶段            |
|                   | code_intents、code_changes、docs）       |                     |                                |
| `context.json`    | 用户提供的规划上下文          | 步骤 2 后冻结 | 子 agent 上下文交接     |
| `qr-{phase}.json` | 特定阶段的 QA 条目（临时） | 临时性           | QA 拆解、验证时 |
