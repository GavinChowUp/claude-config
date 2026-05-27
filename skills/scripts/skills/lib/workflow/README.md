# 工作流框架

## 概述

用于 skill 注册与测试的框架。skill 使用 `Workflow` 类配合 `StepDef` 实例定义——这是一种数据驱动的方式，将转换关系表达为显式数据结构。

## 架构

```
Skills 层（12 个模块）
       |
       v
   Workflow API（Workflow/StepDef/Outcome）
       |
       v
Discovery 层（importlib 扫描）
       |
       v
核心框架（类型、注册表、ResourceProvider）
       |
       v
CLI / 测试脚手架
```

### 数据流

```
CLI 调用
      |
      v
discover_workflows() -> 扫描 skills/ -> 构建注册表
      |
      v
Workflow.run(step_id) -> STEPS[step_id].handler(context)
      |
      v
StepOutput（title、actions、next_command）
```

Discovery 使用 importlib 扫描来发现工作流，无需执行模块级代码。这种拉取式方式消除了导入时的副作用，并支持隔离测试。

## 核心类型

### Outcome 枚举

将「结果是什么？」与「下一步去哪？」分离，使转换图成为可内省的数据：

```python
class Outcome(str, Enum):
    OK = "ok"           # 成功，继续下一步
    FAIL = "fail"       # 失败，可能触发错误处理
    SKIP = "skip"       # 跳过分支（用于模式分支）
    ITERATE = "iterate" # 继续循环（用于置信度推进）
    DEFAULT = "_default" # 若无对应结果映射时的兜底
```

**为何不用布尔值？** 布尔值强迫转换逻辑写在代码里。Outcome 让转换图成为可验证、可可视化、可推理的数据。

**示例**：没有 Outcome 时，模式分支需要：

```python
if mode == "quick":
    return "step_11"  # 含义不明
else:
    return "step_5"   # 这是什么意思？成功？跳过？
```

有了 Outcome：

```python
def step_planning(ctx):
    if ctx.workflow_params["mode"] == "quick":
        return Outcome.SKIP, {}  # 明确：跳过此分支
    return Outcome.OK, {}        # 明确：正常推进

StepDef(id="planning", next={
    Outcome.OK: "subagent_design",      # full 模式路径
    Outcome.SKIP: "initial_synthesis",  # quick 模式路径
})
```

转换图现在在 StepDef 中一目了然，不再埋藏在 handler 逻辑里。

### StepContext

传递给 handler 的运行时状态容器，支持有状态迭代：

```python
@dataclass
class StepContext:
    step_id: str                         # 当前步骤标识符
    workflow_params: dict[str, Any]      # 不可变工作流参数（--mode、--decision）
    step_state: dict[str, Any]           # 可变状态（迭代计数、置信度等级）
```

**为何要区分 params 和 state？**

- `workflow_params`：在工作流启动时设置，之后不变（如 mode、输入路径）
- `step_state`：由 handler 更新，在步骤间传递迭代状态

**示例——置信度驱动的迭代**：

```python
def step_investigate(ctx: StepContext) -> tuple[Outcome, dict]:
    iteration = ctx.step_state.get("iteration", 1)
    confidence = ctx.workflow_params.get("confidence", "exploring")

    if confidence == "high":
        return Outcome.OK, {"confidence": confidence}
    elif iteration >= MAX_ITERATIONS:
        return Outcome.OK, {"confidence": "capped"}
    else:
        return Outcome.ITERATE, {"iteration": iteration + 1}

StepDef(id="investigate", handler=step_investigate,
        next={Outcome.OK: "formulate", Outcome.ITERATE: "investigate"})
```

### Handler 签名

Handler 处理步骤逻辑并返回下一个结果：

```python
def handler(ctx: StepContext) -> tuple[Outcome, dict]:
    # 访问工作流参数（不可变）
    mode = ctx.workflow_params.get("mode", "full")

    # 访问步骤状态（来自上一次迭代）
    iteration = ctx.step_state.get("iteration", 1)

    # 执行步骤逻辑...

    # 返回结果和更新后的状态
    return Outcome.OK, {"iteration": iteration + 1}
```

**为何返回状态 dict？** Handler 是纯函数。返回状态而非修改上下文，使流程显式且可测试。

**仅输出步骤**：只打印指引的步骤可以使用无操作 handler：

```python
def step_handler(ctx: StepContext) -> tuple[Outcome, dict]:
    return Outcome.OK, {}
```

### Arg（参数元数据）

为 handler 参数添加注解，用于测试：

```python
@dataclass(frozen=True)
class Arg:
    description: str = ""
    default: Any = inspect.Parameter.empty
    min: int | float | None = None
    max: int | float | None = None
    choices: tuple[str, ...] | None = None
    required: bool = False
```

**使用方式**：

```python
from typing import Annotated

def step_handler(
    ctx: StepContext,
    mode: Annotated[str, Arg(description="Workflow mode", choices=("quick", "full"))] = "full"
) -> tuple[Outcome, dict]:
    ...
```

`Arg` 元数据在工作流验证期间被提取，用于测试。

### Dispatch vs 可调用 Handler

**可调用 handler**：内联 Python 函数（最常见）

```python
def step_analyze(ctx: StepContext) -> tuple[Outcome, dict]:
    # 分析逻辑
    return Outcome.OK, {}

StepDef(id="analyze", handler=step_analyze, ...)
```

**Dispatch handler**：委托给子 agent 脚本（用于 QR gate、并行 agent）

```python
from skills.lib.workflow.types import Dispatch, AgentRole

StepDef(
    id="qr_completeness",
    handler=Dispatch(
        agent=AgentRole.QUALITY_REVIEWER,
        script="skills.planner.qr.plan_completeness",
    ),
    next={Outcome.OK: "implementation", Outcome.FAIL: "revise_plan"}
)
```

`Dispatch` handler 告知编排器：

1. 用指定脚本启动对应 agent
2. 等待完成
3. 将 agent 的结果映射为 Outcome

**何时使用 Dispatch？**

- QR gate（质量审查检查）
- 并行子 agent 执行
- 需要独立脚本的复杂子工作流

## 工作流验证

`Workflow.__init__` 执行 5 项验证检查：

1. **入口点存在**：`entry_point` 步骤 ID 必须在工作流中
2. **所有转换目标存在**：`next` dict 中的每个目标必须是合法步骤 ID 或 `None`（终止）
3. **至少一个终止步骤**：至少一个步骤的 `next` dict 中包含 `None`
4. **所有步骤可达**：每个步骤必须从入口点可达（检测孤立步骤）
5. **参数提取**：从 handler 签名提取 `Arg` 元数据，用于测试

这些检查在注册时运行，尽早捕获错误。

## 工作流示例

```python
from skills.lib.workflow import discover_workflows
from skills.lib.workflow.core import (
    Workflow, StepDef, StepContext, Outcome, Arg
)

def step_handler(ctx: StepContext) -> tuple[Outcome, dict]:
    return Outcome.OK, {}

WORKFLOW = Workflow(
    "decision-critic",
    StepDef(
        id="extract_structure",
        title="Extract Structure",
        actions=[...],
        handler=step_handler,
        next={Outcome.OK: "classify_verifiability"},
    ),
    StepDef(
        id="classify_verifiability",
        title="Classify Verifiability",
        actions=[...],
        handler=step_handler,
        next={Outcome.OK: "generate_questions"},
    ),
    # ... 其余步骤
    description="Structured decision criticism workflow",
)

# 工作流发现通过 discover_workflows('skills') 进行
# 无需手动注册——直接读取 WORKFLOW 常量
# 拉取式发现消除了导入时的副作用（里程碑 1）
```

此架构的优势：

- 步骤与转换共同存在于数据结构中
- 转换显式且可验证
- 工作流结构可内省且可验证
- 转换图可内省

## 不变量

- **不变量 1**：每个 skill 入口点定义恰好一个 Workflow
- **不变量 2**：discover_workflows() 能找到所有 Workflow 且无导入错误
- **不变量 3**：Dispatcher 路由产生的输出与旧 if-step 链相同
- **不变量 4**：ResourceProvider 协议支持全部 5 种访问模式（约定、文件 I/O、资源、Workflow 对象、步骤数据）
- **不变量 5**：QR 迭代阻塞严重性：迭代 1–2 阻塞所有；迭代 3–4 阻塞 MUST/SHOULD；迭代 5+ 仅阻塞 MUST

## 设计决策

**为何区分 Workflow 和 StepDef？** Workflow 是集合，步骤是原子单元。分离允许在 Workflow 层级进行验证（可达性、终止步骤），同时保持步骤定义聚焦。

**为何使用冻结 dataclass？** Workflow 和 StepDef 是不可变规范。冻结 dataclass 防止意外修改，并允许跨线程安全共享。

**为何使用可调用 handler 而非字符串？** 类型安全、IDE 支持和更易重构。Handler 是一等函数，而非魔法字符串。

**独立 CLI 入口点**：将模块作为 `__main__` 运行会导致模块身份问题（通过 `__init__.py` 导入 vs 作为 `__main__` 执行）。独立的 CLI 入口点可避免此问题。

## 权衡

**惯用 API vs 最小化**：为跨所有 skill 保持一致架构而扩大了重构范围。think.py 的模式证明了 Workflow/StepDef API 可行；扩展它能在不引入新抽象的前提下实现一致性。

**集中枚举 vs 本地**：多一个维护点，换来可发现性和共享理解。枚举（LoopState、DocumentAvailability）使状态机显式，并支持基于属性的测试。

**干净切断 vs 双路径**：实现更简单，代价是无迁移期。重构范围仅限内部（无外部调用者），因此干净切断减少了总工作量并消除了过渡期 bug。

## 常用模式

### 模式 1：线性工作流

```python
WORKFLOW = Workflow(
    "skill-name",
    StepDef(id="step1", title="...", actions=[...],
            handler=step_handler, next={Outcome.OK: "step2"}),
    StepDef(id="step2", title="...", actions=[...],
            handler=step_handler, next={Outcome.OK: "step3"}),
    StepDef(id="step3", title="...", actions=[...],
            next={Outcome.OK: None}),  # 终止步骤
)
```

### 模式 2：置信度驱动的迭代

```python
def step_investigate(ctx: StepContext) -> tuple[Outcome, dict]:
    iteration = ctx.step_state.get("iteration", 1)
    confidence = ctx.workflow_params.get("confidence", "exploring")

    if confidence == "high":
        return Outcome.OK, {"confidence": confidence}
    elif iteration >= MAX_ITERATIONS:
        return Outcome.OK, {"confidence": "capped"}
    else:
        return Outcome.ITERATE, {"iteration": iteration + 1}

StepDef(id="investigate", handler=step_investigate,
        next={
            Outcome.OK: "formulate",       # 退出循环
            Outcome.ITERATE: "investigate"  # 继续循环
        })
```

### 模式 3：模式分支

```python
def step_planning(ctx: StepContext) -> tuple[Outcome, dict]:
    mode = ctx.workflow_params.get("mode", "full")
    if mode == "quick":
        return Outcome.SKIP, {}
    return Outcome.OK, {}

StepDef(id="planning", handler=step_planning,
        next={
            Outcome.OK: "subagent_design",      # full 模式
            Outcome.SKIP: "initial_synthesis",  # quick 模式
        })
```

### 模式 4：QR Gate

```python
from skills.lib.workflow.types import Dispatch, AgentRole

StepDef(
    id="qr_completeness",
    title="QR: Plan Completeness",
    actions=[...],
    handler=Dispatch(
        agent=AgentRole.QUALITY_REVIEWER,
        script="skills.planner.qr.plan_completeness",
    ),
    next={
        Outcome.OK: "implementation",   # QRStatus.PASS -> Outcome.OK
        Outcome.FAIL: "revise_plan",    # QRStatus.FAIL -> Outcome.FAIL
    },
)
```

### 模式 5：混合静态/动态步骤（deepthink）

大多数步骤为静态、少数步骤需要参数化的工作流可采用混合方式：

```python
# ============================================================================
# MESSAGE BUILDERS
# ============================================================================

def build_dispatch_body() -> str:
    """动态格式化器可调用的构建函数。"""
    # ... 实现
    return dispatch_text


# ============================================================================
# STEP DEFINITIONS
# ============================================================================

# 静态步骤：(title, instructions) 元组
STATIC_STEPS = {
    1: ("Context Clarification", CONTEXT_CLARIFICATION_INSTRUCTIONS),
    2: ("Abstraction", ABSTRACTION_INSTRUCTIONS),
    # ... 更多静态步骤
}


# 动态格式化函数——必须定义在 DYNAMIC_STEPS dict 之前
def _format_step_9(mode: str, confidence: str, iteration: int) -> tuple[str, str]:
    """调用构建函数的动态步骤。"""
    return ("Dispatch", build_dispatch_body())


def _format_step_13(mode: str, confidence: str, iteration: int) -> tuple[str, str]:
    """带参数化标题和正文的动态步骤。"""
    suffix = " -> Complete" if confidence == "certain" else ""
    title = f"Iterative Refinement (Iteration {iteration}){suffix}"
    body = INSTRUCTIONS.format(iteration=iteration, max_iter=MAX_ITERATIONS)
    return (title, body)


# 动态步骤 dict——引用上面定义的函数
DYNAMIC_STEPS = {
    9: _format_step_9,
    13: _format_step_13,
}


# ============================================================================
# OUTPUT FORMATTING
# ============================================================================

def format_output(step: int, mode: str, confidence: str, iteration: int) -> str:
    """可调用派发：静态查找或动态函数调用。"""
    if step in STATIC_STEPS:
        title, instructions = STATIC_STEPS[step]
    elif step in DYNAMIC_STEPS:
        title, instructions = DYNAMIC_STEPS[step](mode, confidence, iteration)
    else:
        return f"ERROR: Invalid step {step}"

    next_cmd = build_next_command(step, mode, confidence, iteration)
    return format_step(instructions, next_cmd or "", title=f"WORKFLOW - {title}")
```

**顺序约束（书本模式）**：调用 MESSAGE BUILDERS 的动态格式化函数必须出现在 MESSAGE BUILDERS 之后。DYNAMIC_STEPS 字典必须出现在它引用的所有 `_format_step_*` 函数之后。

适用场景：

- 大多数步骤共享相同结构（标题 + 常量正文）
- 少数步骤需要参数来构造标题或正文
- 参数在所有动态步骤间统一

优势：

- 静态步骤表示紧凑（每步一行）
- 动态步骤函数清晰可读
- `format_output()` 中单一派发点
- 遵循「书本模式」（所有引用都能解析到上方的定义）

## 问题转发协议

子 agent 可通过主 agent 向用户请求澄清。该协议是纯 prompt 协调——无 Python 拦截。

### 设计决策

**任务重新调用（而非恢复）**：子 agent 带着问题让出后，编排器在获得用户回答后以全新方式重新调用它（新 Task，无 resume 参数）。子 agent 在让出前将状态保存到 plan.json，重新调用后再读回。选择此方案而非 resume，原因如下：

- resume 语义不可靠（0 tokens、0 tool uses 失败）
- 状态文件读取显式且可审计
- 全新上下文避免了过期上下文问题
- 子 agent 脚本可检测续跑状态（plan.json 是否存在）

**仅输出问题**：子 agent 需要澄清时，只输出 `<needs_user_input>` XML 块，其余什么都不输出。这使检测无歧义——无需对自然语言进行启发式解析。

**显式 XML 标记**：使用结构化 XML 标签，而非检测散文中的问号。这防止了分析输出中的反问句造成误判。

**最多 3 个问题，每题 2–3 个选项**：约束与 AskUserQuestion 工具 schema 匹配。批量提问减少往返次数。选项应明确且可操作。

**让出前保存状态**：子 agent 必须在发出 `<needs_user_input>` 之前将所有进度保存到 plan.json。重新调用的实例将读取此状态。

### 流程

1. 子 agent 将当前状态保存到 plan.json
2. 子 agent 以整个响应发出 `<needs_user_input>` XML
3. 主 agent 提取问题，调用 AskUserQuestion
4. 主 agent 以全新方式重新调用子 agent，传入回答和 STATE_DIR
5. 新子 agent 实例读取 plan.json，从已保存状态继续

### 常量

| 常量                        | 用途                                  |
| --------------------------- | ---------------------------------------- |
| `SUB_AGENT_QUESTION_FORMAT` | 告知子 agent 如何发出问题    |
| `QUESTION_RELAY_HANDLER`    | 告知主 agent 如何检测并转发 |

### 集成

对于支持问题转发的派发步骤：

```python
from skills.lib.workflow.constants import QUESTION_RELAY_HANDLER

# 在派发步骤的 format_output 或 step handler 中：
if step_info.get("supports_questions"):
    actions.append(QUESTION_RELAY_HANDLER)
```

对于可能提问的子 agent 脚本：

```python
from skills.lib.workflow.constants import SUB_AGENT_QUESTION_FORMAT

# 在第 1 步的指引中：
actions.append(SUB_AGENT_QUESTION_FORMAT)
```

## 不变量

- 每个 skill 模块都出现在 `tests/conftest.py` 的 `SKILL_MODULES` 中
- 工作流验证必须通过（入口点存在、所有转换合法、至少一个终止步骤、所有步骤可达）
- Handler 签名必须匹配 `(ctx: StepContext) -> tuple[Outcome, dict]`，或为 `Dispatch` 实例
- `next` dict 的键必须是 `Outcome` 枚举值
- `next` dict 的值必须是合法步骤 ID 或 `None`（终止）

## 穷举测试框架

穷举测试框架为工作流步骤生成所有合法参数组合，使用类型化领域抽象来表示参数空间。

### 架构

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

### 为何如此设计

领域类型与生成逻辑分离：

- 领域类型可复用（可驱动模糊测试、文档生成）
- 生成逻辑依赖工作流结构，而非领域语义
- 测试文件额外引入 pytest 专属关注点

### 数据流

1. 导入 skill -> 注册 Workflow 对象
2. extract_schema(workflow) -> {step: {param: Domain}}
3. generate_inputs(workflow) -> Iterator[dict]（笛卡尔积）
4. pytest.parametrize -> 带 ID 的测试用例
5. run_skill_invocation(workflow, params) -> 子进程退出码

### 关键设计决策

**穷举 vs 采样**：领域规模很小（5 次迭代 x 5 种置信度 x 2 种模式 = 约 300–500 个总用例）。穷举枚举可行，提供完整覆盖。采样会遗漏边界组合。

**硬编码模式门控**：只有 deepthink 有 mode 参数（quick 模式跳过步骤 6–11）。单一情况不值得引入内省复杂度。显式硬编码更清晰且易维护。

**迭代检测**：Step.next dict 中包含 Outcome.ITERATE 的自循环步骤。直接检查无需启发式，适用于所有当前和未来的迭代工作流。

**步骤索引映射**：\_params 以 step_id（字符串）而非步骤编号为键。\_step_order 提供用于 CLI 调用的权威索引。

### 不变量

- 每个测试用例有唯一 ID（workflow-step-params 组合）
- 条件参数仅适用于对应步骤（iteration 只在迭代步骤出现）
- 模式门控步骤在对应模式值下被跳过
- step 参数始终存在（1 到 total_steps）
- total_steps 始终与 workflow.total_steps 匹配
- Workflow.\_step_order 提供权威步骤索引映射：len(\_step_order) == total_steps，索引对应 CLI --step 值

### 领域类型

位于 types.py：

**BoundedInt**：含闭区间 [lo, hi] 的整数领域

```python
list(BoundedInt(1, 5))  # [1, 2, 3, 4, 5]
```

**ChoiceSet**：离散选择领域

```python
list(ChoiceSet(("full", "quick")))  # ["full", "quick"]
```

**Constant**：单值领域

```python
list(Constant(42))  # [42]
```

所有类型均实现 **iter**，可与 itertools.product 配合使用。frozen=True 支持 pytest 参数缓存的可哈希性。

## 测试

所有测试使用 pytest。从 `skills/scripts/` 运行：

```bash
# 运行所有测试
pytest tests/ -v

# 测试指定工作流
pytest tests/ -k deepthink -v

# 按类别测试
pytest tests/test_workflow_import.py -v     # 导入测试
pytest tests/test_workflow_structure.py -v  # 结构验证
pytest tests/test_workflow_steps.py -v      # 步骤可调用性（穷举）
pytest tests/test_domain_types.py -v        # 领域类型单元测试
```
