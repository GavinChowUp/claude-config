# Planner

带 QR（Quality Review）gate、TW（Technical Writer）阶段与 Dev（Developer）执行阶段的规划与执行工作流。

本文档是 planner skill 架构的权威说明。

## 架构：Python 脚本 vs LLM

Python 脚本输出工作流 prompt 和路由指令。LLM 在两次脚本调用之间运行：

1. 脚本为当前步骤输出 prompt/指引
2. LLM 读取 prompt，执行推理/评估
3. LLM 决定结果（如 QR PASS/FAIL）
4. LLM 根据结果调用下一个脚本

QR PASS/FAIL 由读取 QR 输出的 LLM 决定，而非 Python。Gate 路由是 LLM 基于 QR 结果的决策。Python 脚本提供结构，LLM 提供智能。

## 状态文件

除初始 context.json 外，所有状态修改均通过 Python CLI 命令完成。状态目录通过 `tempfile.mkdtemp()` 在 `/tmp` 下创建。

| 文件              | Schema         | 创建者     | 修改者     | 生命周期              |
| ----------------- | -------------- | ----------- | -------------- | ---------------------- |
| `plan.json`       | Pydantic v2    | 步骤 1 初始化 | CLI 命令   | 可变 -> 冻结      |
| `context.json`    | 宽松 JSON     | 步骤 2      | LLM Write 工具 | 步骤 2 后冻结    |
| `qr-{phase}.json` | QA 条目 schema | QR 派发 | QR 期间的 LLM  | 每个 QR 周期临时存在 |

### plan.json Schema

```
Plan
  schema_version: 2
  plan_id: UUID
  created_at: timestamp
  frozen_at: Optional[timestamp]

  overview:
    title, problem, approach

  planning_context:
    decision_log[]: id (DL-XXX), decision, reasoning_chain, timestamp
    rejected_alternatives[]: id (RA-XXX), alternative, rejection_reason, decision_ref
    constraints[]: id (C-XXX), type, description, source
    known_risks[]: id (R-XXX), risk, mitigation, anchor?, decision_ref?

  invisible_knowledge:
    architecture: {diagram_ascii, description}
    data_flow: {diagram_ascii, description}
    structure_rationale, invariants[], tradeoffs[]

  milestones[]:
    id (M-XXX), number, name, files[], flags[], requirements[], acceptance_criteria[]
    tests: files[], type?, backing?, scenarios{normal[], edge[], error[]}, skip_reason?
    code_intents[]: id (CI-XXX), file, function?, behavior, decision_refs[], params{}
    code_changes[]: id (CC-XXX), intent_ref, file, diff, context_lines, why_comments[]
    documentation: module_comment?, docstrings[], algorithm_blocks[], inline_comments[]
    is_documentation_only, delegated_to?

  milestone_dependencies:
    diagram_ascii
    waves[]: wave number, milestones[]
```

引用完整性：code_change.intent_ref -> code_intent.id，decision_refs -> decision_log.id

### context.json Schema

规划期间捕获的用户提供上下文：

```json
{
  "task_spec": ["goal", "scope", "out-of-scope"],
  "constraints": ["MUST: X", "SHOULD: Y"],
  "entry_points": ["file:function - why"],
  "rejected_alternatives": ["alternative - why dismissed"],
  "current_understanding": ["how system works"],
  "assumptions": ["inference (confidence)"],
  "invisible_knowledge": ["design rationale", "invariants"],
  "user_quotes": ["verbatim quote"]
}
```

### qr-{phase}.json Schema

阶段：`qr-plan-design`、`qr-plan-code`、`qr-plan-docs`、`qr-impl-code`、`qr-impl-docs`

```json
{
  "schema_version": "1.0",
  "phase": "plan-design",
  "items": [
    {
      "id": "qa-001",
      "scope": "*",
      "check": "...",
      "status": "TODO|PASS|FAIL",
      "finding": null
    }
  ]
}
```

## 工作流阶段与修改

### Planner 工作流（11 步）

| 步骤 | 名称                | 模式函数          | 修改内容              | Agent        |
| ---- | ------------------- | ------------------------- | -------------------- | ------------ |
| 1    | plan-init           | `init_step()`             | 创建 plan.json    | Orchestrator |
| 2    | context-verify      | `verify_step()`           | 创建 context.json | Orchestrator |
| 3    | plan-design-execute | `execute_dispatch_step()` | plan.json            | Architect    |
| 4    | plan-design-qr      | `qr_dispatch_step()`      | qr-plan-design.json  | QR           |
| 5    | plan-design-qr-gate | `qr_gate_step()`          | -                    | Orchestrator |
| 6    | plan-code-execute   | `execute_dispatch_step()` | plan.json            | Developer    |
| 7    | plan-code-qr        | `qr_dispatch_step()`      | qr-plan-code.json    | QR           |
| 8    | plan-code-qr-gate   | `qr_gate_step()`          | -                    | Orchestrator |
| 9    | plan-docs-execute   | `execute_dispatch_step()` | plan.json            | TW           |
| 10   | plan-docs-qr        | `qr_dispatch_step()`      | qr-plan-docs.json    | QR           |
| 11   | plan-docs-qr-gate   | `qr_gate_step()`          | 设置 frozen_at       | Orchestrator |

**修改详情**：

- 步骤 3（Architect）：填充 planning_context、milestones[]、code_intents[]、invisible_knowledge
- 步骤 6（Developer）：按 milestone 填充 code_changes[]
- 步骤 9（TW）：按 milestone 填充 documentation[]，创建 plan.md

### Executor 工作流（9 步）

| 步骤 | 名称              | 修改内容           | Agent        |
| ---- | ----------------- | ----------------- | ------------ |
| 1    | init              | -                 | Orchestrator |
| 2    | load-verify       | -                 | Orchestrator |
| 3    | impl-execute      | 代码库文件    | Developer    |
| 4    | impl-code-qr      | qr-impl-code.json | QR           |
| 5    | impl-code-qr-gate | -                 | Orchestrator |
| 6    | impl-docs-execute | 代码库文档     | TW           |
| 7    | impl-docs-qr      | qr-impl-docs.json | QR           |
| 8    | impl-docs-qr-gate | -                 | Orchestrator |
| 9    | reconcile         | -                 | QR           |

## 组件

```
orchestrator/
  planner.py      11 步规划工作流
  executor.py     9 步执行工作流

architect/
  plan_design.py  计划创建（探索、milestones、code_intents）

developer/
  plan_code.py    Code Intent -> Code Changes（unified diff）
  exec_implement.py  波次感知的实现

technical_writer/
  plan_docs.py    文档规划（WHY 注释、时态清理）
  exec_docs.py    实现后文档（CLAUDE.md、README.md）

quality_reviewer/
  plan_design_qr.py   计划完整性验证
  plan_code_qr.py     代码 diff 验证
  plan_docs_qr.py     文档质量
  impl_code_qr.py     实现后代码审查
  impl_docs_qr.py     实现后文档审查
  exec_reconcile.py   计划与实现对账

shared/
  resources.py    路径推导、上下文加载
  builders.py     XML 输出构建器
  constraints.py  编排器约束 AST 构建器
  qr/             QR 工具（类型、常量、工具函数、schema）

state/
  models.py       plan.json 的 Pydantic v2 schema
  validator.py    验证函数
  decisions.py    决策生命周期枚举（保留供未来使用）

cli/
  plan.py         plan.json 操作命令
```

## QR Gate 机制

QR gate 使用 LoopState 枚举追踪迭代进度：INITIAL -> RETRY -> COMPLETE

```
INITIAL -> PASS -> COMPLETE（终止）
INITIAL -> FAIL -> RETRY（iteration++）
RETRY   -> FAIL -> RETRY（iteration++）
RETRY   -> PASS -> COMPLETE（终止）
```

按迭代次数的阻塞严重性：

| 迭代次数  | 阻塞              |
| --------- | ------------------- |
| 1–2       | MUST、SHOULD、COULD |
| 3–4       | MUST、SHOULD        |
| 5+        | 仅 MUST           |

## 步骤 Handler 架构

闭包捕获静态配置，handler 接收动态状态：

```python
def execute_dispatch_step(title, agent, script, ...):
    def handler(ctx):  # 接收 state_dir、qr、qr_fail
        return {"title": ..., "actions": ..., "next": ...}
    return handler

STEPS = {
    1: init_step("plan-init", ...),
    3: execute_dispatch_step("plan-design-execute", agent="architect", ...),
    4: qr_dispatch_step("plan-design-qr", ...),
    5: qr_gate_step("plan-design-qr-gate", ...),
}
```

## 设计决策

**基于闭包的步骤派发**：STEPS dict 将步骤编号映射到 handler 闭包。模式函数捕获静态配置（title、agent、script），handler 通过 ctx 接收动态状态。用显式模式替代魔法键。

**基于约定的路径**：子 agent 接收 --state-dir，通过 get_context_path() 推导文件路径。修改 context.json 位置只需更新 resources.py。

**LLM 管理状态**：状态文件由读取步骤指引的 LLM agent 写入，而非 Python 脚本。利用 LLM 的上下文理解和格式遵循能力。

**JSON-IR 优先**：plan.json 是权威来源；plan.md 从其派生。

**QR 迭代阻塞**：严重性阈值随迭代次数变化。早期迭代阻塞所有严重性。后期迭代仅阻塞 MUST，防止无限循环。

**不清理临时目录**：由操作系统在重启时处理 /tmp 清理。

## 不变量

1. 每个 skill 入口点定义恰好一个 Workflow
2. discover_workflows() 能找到所有 Workflow 且无导入错误
3. plan.json 自包含，可独立执行
4. 冻结的 plan.json 不可变（frozen_at 时间戳意味着不再写入）
5. qr-{phase}.json 文件是临时的（仅在 QR 周期内存在）
6. QR 迭代阻塞：迭代 1–2 全部；迭代 3–4 MUST/SHOULD；迭代 5+ 仅 MUST
