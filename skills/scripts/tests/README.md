# 工作流测试框架

## 概述

数据驱动的测试框架，用所有合法参数组合穷举测试所有基于工作流的 skill 的每个步骤。使用类型化领域抽象（BoundedInt、ChoiceSet、Constant）表示参数空间，从 Workflow AST 提取 schema，生成合法输入的笛卡尔积，并通过 parametrize 与 pytest 集成。

## 架构

```
Workflow AST          领域类型           测试生成
     |                     |                       |
     v                     v                       v
+----------+        +-------------+         +--------------+
| Workflow |  --->  | BoundedInt  |  --->   | generate_    |
| _params  |        | ChoiceSet   |         | test_inputs  |
| _step_   |        | Constant    |         +--------------+
|  order   |        +-------------+                |
+----------+                                       v
                                           +-------------+
                                           | pytest      |
                                           | parametrize |
                                           +-------------+
```

## 数据流

1. 导入 skill -> 注册 Workflow 对象
2. extract_schema(workflow) -> {step: {param: Domain}}
3. generate_inputs(workflow) -> Iterator[dict]（笛卡尔积）
4. pytest.parametrize -> 带 ID 的测试用例
5. run_skill_invocation(workflow, params) -> 子进程退出码

## 为何如此设计

领域类型与生成逻辑分离，原因如下：

- 领域类型可复用（可驱动模糊测试、文档生成等）
- 生成逻辑依赖工作流结构，而非领域语义
- 测试文件依赖两者，但额外引入 pytest 专属关注点

这种分离带来以下好处：

- 可对测试框架本身进行测试（领域类型可单元测试）
- 可将领域抽象复用于其他目的
- 关注点边界清晰（FP 可组合性）

## 设计决策

### 穷举 vs 采样

选择穷举枚举，因为领域规模很小。当前工作流共生成约 300–500 个测试用例（每个迭代步骤：5 次迭代 x 5 种置信度 x 2 种模式）。穷举测试可行，且能覆盖所有边界情况。采样会遗漏特定参数值相互作用导致失败的边界组合。

代价：更多测试用例需要运行
收益：合法输入空间的完整覆盖

### 硬编码 vs 内省模式门控

选择硬编码模式门控步骤（目前只有 deepthink 有跳过步骤 6–11 的 quick 模式）。通过内省 handler 字节码来检测模式门控，对于单一工作流而言复杂度不合算。

代价：若更多工作流添加模式门控，需手动更新
收益：代码清晰、易于维护

### 领域类型放在 types.py

领域类型（BoundedInt、ChoiceSet、Constant）与 Arg、QRStatus、Confidence 一起放在 workflow/types.py 中。这保持了内聚性——领域类型是类型系统的扩展。另一种方案会造成导入碎片化。

### 迭代上界硬编码为 5

iteration 领域的 BoundedInt(1, 5) 与 QR_ITERATION_LIMIT 常量一致。硬编码以避免对配置常量的导入耦合。当前值（5）是所有迭代工作流的标准。

## 不变量

- 每个测试用例有唯一 ID（workflow-step-params 组合）
- 条件参数仅适用于对应步骤（iteration 只在迭代步骤出现）
- 模式门控步骤在对应模式值下会被跳过
- step 参数始终存在（1 到 workflow.total_steps）
- Workflow.\_step_order 提供权威的步骤索引映射：len(\_step_order) == workflow.total_steps，索引对应 CLI --step 值

## 约束

- Python 3.10+（dataclass、match 语句）
- pytest 可用
- conftest.py 中的 run_skill_invocation() 负责子进程执行
- MAX_ITERATIONS = 5，所有工作流统一
- 排除的 skill：leon-writing-style、prompt-engineer-improver（不在 git 中）
