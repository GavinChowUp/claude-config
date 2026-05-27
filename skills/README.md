# Skills 架构

基于脚本的 agent 工作流，共享同一套编排框架。

## 文件组织：「书本」模式

Skill 文件从上到下像读书一样阅读。依赖项在使用前定义。核心原则：**读者永远不需要向上翻滚才能理解当前内容。**

### 区段顺序

文件使用固定的区段序列。按类型分组，而非按步骤。在每个类型组内，按工作流步骤排序。无内容的区段可省略。

```python
# ============================================================================
# SHARED PROMPTS
# ============================================================================
# 被 2 个及以上工作流步骤使用的 prompt。如果较大，提取到 prompts/shared.py

# ============================================================================
# CONFIGURATION
# ============================================================================
# 常量、温度参数、阈值

# ============================================================================
# SYSTEM PROMPTS
# ============================================================================

# ============================================================================
# MESSAGE TEMPLATES
# ============================================================================
# 按步骤划分的子区段（见下文）

# ============================================================================
# PARSING FUNCTIONS
# ============================================================================

# ============================================================================
# MESSAGE BUILDERS
# ============================================================================
# 将模板组合成完整消息的函数

# ============================================================================
# [DOMAIN] LOGIC
# ============================================================================
# 领域相关的（工具性）函数

# ============================================================================
# STEP DEFINITIONS
# ============================================================================
# STATIC_STEPS 和 DYNAMIC_STEPS 字典（表驱动派发）

# ============================================================================
# OUTPUT FORMATTING
# ============================================================================
# format_output() 入口点

# ============================================================================
# ENTRY POINT
# ============================================================================
# main() 函数
```

原因：函数经常引用来自多个步骤的 prompt。按类型分组可避免函数区段内的前向引用。

### 按步骤划分的 MESSAGE TEMPLATES

在 MESSAGE TEMPLATES 内，使用步骤分隔符按时间顺序组织：

```python
# ============================================================================
# MESSAGE TEMPLATES
# ============================================================================

# --- STEP 1: SCOPE -----------------------------------------------------------

SCOPE_INSTRUCTIONS = """..."""

# --- STEP 2: SURVEY ----------------------------------------------------------

SURVEY_DISPATCH_CONTEXT = """\
Analysis goals from SCOPE step:
- User intent and what they want to understand
- Identified focus areas"""

SURVEY_DISPATCH_AGENTS = [
    "[Exploration focus 1: e.g., 'Explore authentication flow']",
    "[Exploration focus 2: e.g., 'Explore database schema']",
]

SURVEY_DISPATCH_GUIDANCE = """\
DISPATCH GUIDANCE:
...
ADVANCE: After results received, re-invoke with --confidence low."""

SURVEY_LOW_INSTRUCTIONS = """..."""
SURVEY_MEDIUM_INSTRUCTIONS = """..."""

# --- STEP 3: DEEPEN ----------------------------------------------------------

DEEPEN_DISPATCH_CONTEXT = SURVEY_DISPATCH_CONTEXT  # reuse if identical
DEEPEN_LOW_INSTRUCTIONS = """..."""

# --- STEP 4: SYNTHESIZE ------------------------------------------------------

SYNTHESIZE_EXPLORING_INSTRUCTIONS = """..."""
```

步骤分隔符格式：`# --- STEP N: PHASE_NAME ` 后跟破折号至第 76 列。

在步骤区段内，按执行流程顺序排列常量。dispatch 相关常量（context、agents、guidance）放在 instruction 常量之前。

### Dispatch Prompt：模板 vs 构建函数

Dispatch prompt 将静态模板与动态组合结合。将其拆分：

**静态部分 -> MESSAGE TEMPLATES**（常量）：

```python
# --- STEP 2: SURVEY ----------------------------------------------------------

SURVEY_DISPATCH_CONTEXT = """\
Analysis goals from SCOPE step:
- User intent and what they want to understand
- Identified focus areas (architecture, components, flows, etc.)"""

SURVEY_DISPATCH_AGENTS = [
    "[Exploration focus 1: e.g., 'Explore authentication flow']",
    "[Exploration focus 2: e.g., 'Explore database schema']",
]

SURVEY_DISPATCH_GUIDANCE = """\
DISPATCH GUIDANCE:

Single codebase, focused scope:
  - One Explore agent with specific focus

Large/broad scope:
  - Multiple parallel Explore agents by boundary

WAIT for Explore results before re-invoking this step.

ADVANCE: After results received, re-invoke with --confidence low."""
```

**组合部分 -> MESSAGE BUILDERS**（调用 `roster_dispatch()` 等的函数）：

```python
# ============================================================================
# MESSAGE BUILDERS
# ============================================================================

def build_survey_exploring_body() -> str:
    """Build SURVEY exploring instructions with dispatch."""
    dispatch_text = roster_dispatch(
        agent_type="Explore",
        agents=SURVEY_DISPATCH_AGENTS,
        command="Use Task tool with subagent_type='Explore'",
        shared_context=SURVEY_DISPATCH_CONTEXT,
        model="haiku",
    )
    return f"DISPATCH Explore agent(s):\n\n{dispatch_text}\n\n{SURVEY_DISPATCH_GUIDANCE}"
```

这种分离确保：

1. Prompt 文本在常量定义处即可见（无需追踪函数）
2. 构建函数只引用其上方定义的常量（按时间顺序）
3. 修改 dispatch 参数时不需要改动 prompt 文本

### 命名规范

```
[PHASE]_[TYPE]
```

PHASE 是工作流阶段：`SCOPE`、`SURVEY`、`DEEPEN`、`SYNTHESIZE`、`DISCOVERY`、`IDEATION`。
TYPE 是其角色：`INSTRUCTIONS`、`DISPATCH_CONTEXT`、`DISPATCH_AGENTS`、`DISPATCH_GUIDANCE`、`FORMAT`、`FEEDBACK`。

置信度变体使用后缀：`_LOW`、`_MEDIUM`、`_HIGH`、`_EXPLORING`、`_CERTAIN`。

示例：

```python
EVALUATION_CRITERIA              # 共享（被 2 个及以上步骤使用）
SCOPE_INSTRUCTIONS               # 步骤 1
SURVEY_DISPATCH_CONTEXT          # 步骤 2，dispatch 上下文
SURVEY_DISPATCH_AGENTS           # 步骤 2，dispatch agent 列表
SURVEY_LOW_INSTRUCTIONS          # 步骤 2，低置信度变体
DEEPEN_HIGH_INSTRUCTIONS         # 步骤 3，高置信度变体
SYNTHESIZE_FORMAT                # 步骤 4，输出格式
```

### 归属规则

> Prompt 归属于最早使用它的区段。

- 在步骤 2、4、6 中使用？-> SHARED PROMPTS 区段
- 只在步骤 3 中使用？-> MESSAGE TEMPLATES 中步骤 3 的位置
- 只在步骤 3 和 4 中使用？-> 步骤 3 的位置（连续使用不需要 SHARED）

### 视觉格式

区段标题（76 个等号）：

```python
# ============================================================================
# SECTION NAME
# ============================================================================
```

MESSAGE TEMPLATES 内的步骤分隔符（共 76 个字符）：

```python
# --- STEP N: PHASE_NAME ------------------------------------------------------
```

区段标题前后各留一个空行。步骤分隔符前后不强制要求空行。

### 字符串格式规范

多行字符串使用括号拼接，而非三引号字符串：

```python
# GOOD - 每行在其缩进级别清晰可见
SCOPE_INSTRUCTIONS = (
    "PARSE user intent:\n"
    "  - What codebase(s) are we analyzing?\n"
    "  - What is the user trying to understand?\n"
    "\n"
    "DEFINE goals (1-3 specific objectives):\n"
    "  - 'Understand how [system X] processes [Y]'\n"
    "  - 'Map dependencies between [A] and [B]'"
)

# GOOD - 与变量组合
EXECUTE_INSTRUCTIONS = (
    "Apply each approved change.\n"
    "\n"
    "INTEGRATION CHECKS:\n"
    "  - Cross-section references correct?\n"
    "  - Terminology consistent?\n"
    "\n"
    + ANTI_PATTERN_AUDIT + "\n"
    "\n"
    + CHANGE_PRESENTATION
)

# BAD - 三引号字符串强制内容从第 0 列开始
SCOPE_INSTRUCTIONS = """\
PARSE user intent:
  - What codebase(s) are we analyzing?
  - What is the user trying to understand?"""
```

规则：

- 每行都有自己的字符串字面量，并显式写 `\n`
- 空行单独写成 `"\n"`
- 最后一行没有尾随 `\n`
- 变量组合使用 `+ VARIABLE + "\n"`（如果是最后一个则不加 `"\n"`）

## Skill 如何构建步骤体

没有「action factories」，没有控制反转。只有字符串。

### 模式 1：静态步骤（deepthink 子 agent）

全静态工作流使用独立的 `STEP_TITLES` 和 `STEP_INSTRUCTIONS` 字典：

```python
STEP_TITLES = {
    1: "Context Grounding",
    2: "Analogical Generation",
    3: "Planning",
    # ...
}

STEP_INSTRUCTIONS = {
    1: CONTEXT_GROUNDING_INSTRUCTIONS,
    2: ANALOGICAL_GENERATION_INSTRUCTIONS,
    3: PLANNING_INSTRUCTIONS,
    # ...
}

def format_output(step: int) -> str:
    if step not in STEP_TITLES:
        return f"ERROR: Invalid step {step}"
    title = STEP_TITLES[step]
    instructions = STEP_INSTRUCTIONS[step]
    next_cmd = build_next_command(step)
    return format_step(instructions, next_cmd or "", title=f"WORKFLOW - {title}")
```

### 模式 2：参数化步骤（codebase-analysis）

模板和构建函数分离。模板是 MESSAGE TEMPLATES 中定义的常量（按步骤划分）。构建函数将模板组合成完整消息。

```python
# ============================================================================
# MESSAGE TEMPLATES
# ============================================================================

# --- STEP 2: SURVEY ----------------------------------------------------------

SURVEY_DISPATCH_CONTEXT = (
    "Analysis goals from SCOPE step:\n"
    "- User intent and what they want to understand"
)

SURVEY_DISPATCH_AGENTS = [
    "[Exploration focus 1: e.g., 'Explore authentication flow']",
    "[Exploration focus 2: e.g., 'Explore database schema']",
]

SURVEY_DISPATCH_GUIDANCE = (
    "DISPATCH GUIDANCE:\n"
    "...\n"
    "ADVANCE: After results received, re-invoke with --confidence low."
)

SURVEY_LOW_INSTRUCTIONS = (
    "EXTRACT findings from Explore output:\n"
    "..."
)

# ============================================================================
# MESSAGE BUILDERS
# ============================================================================

def build_survey_exploring_body() -> str:
    dispatch_text = roster_dispatch(
        agent_type="Explore",
        agents=SURVEY_DISPATCH_AGENTS,
        command="Use Task tool with subagent_type='Explore'",
        shared_context=SURVEY_DISPATCH_CONTEXT,
        model="haiku",
    )
    return (
        "DISPATCH Explore agent(s):\n"
        "\n"
        + dispatch_text + "\n"
        "\n"
        + SURVEY_DISPATCH_GUIDANCE
    )

def get_survey_body(confidence: str) -> str:
    if confidence == "exploring":
        return build_survey_exploring_body()
    elif confidence == "low":
        return "SURVEY - Low Confidence\n\n" + SURVEY_LOW_INSTRUCTIONS
    # ...

def format_output(step: int, confidence: str) -> str:
    bodies = {1: get_scope_body, 2: get_survey_body, ...}
    body = bodies[step](confidence)
    next_cmd = build_next_command(step, confidence)
    return format_step(body, next_cmd)
```

### 模式 3：Dispatch 步骤（planner 编排器）

```python
from skills.lib.workflow.prompts import format_step, subagent_dispatch
from skills.planner.prompts.constants import ORCHESTRATOR_CONSTRAINT

def format_dispatch_step(agent_type: str, invoke_cmd: str, state_dir: str) -> str:
    dispatch = subagent_dispatch(agent_type=agent_type, command=invoke_cmd)

    body = (
        ORCHESTRATOR_CONSTRAINT + "\n"
        "\n"
        + dispatch
    )

    next_step_cmd = "python3 -m skills.planner.orchestrator.planner --step N --state-dir " + state_dir
    return format_step(body, next_step_cmd)
```

### 模式 4：文件注入（prompt-engineer）

```python
from skills.lib.workflow.prompts import format_step, format_file_content

def format_technique_step(categories: list[str]) -> str:
    file_blocks = []
    for cat in categories:
        path = CATEGORY_TO_FILE[cat]
        content = (REFS_DIR / path).read_text()
        file_blocks.append(format_file_content(f"references/{path}", content))

    body = (
        "TECHNIQUE REFERENCES\n"
        "The following files have been loaded based on your category selection:\n"
        "\n"
        + "\n".join(file_blocks) + "\n"
        "\n"
        "TASK: Apply techniques from these references to the target prompt.\n"
        "..."
    )

    return format_step(body, "python3 -m skills.prompt_engineer.optimize --step 5")
```

### 模式 5：混合静态/动态步骤（deepthink）

以静态步骤为主、少量参数化步骤的工作流适合使用混合方式：

```python
# ============================================================================
# MESSAGE BUILDERS
# ============================================================================

def build_dispatch_body() -> str:
    """动态格式化器可能调用的构建函数。"""
    # ... implementation
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


# 动态格式化函数——必须在 DYNAMIC_STEPS 字典之前定义
def _format_step_9(mode: str, confidence: str, iteration: int) -> tuple[str, str]:
    """调用构建函数的动态步骤。"""
    return ("Dispatch", build_dispatch_body())


def _format_step_13(mode: str, confidence: str, iteration: int) -> tuple[str, str]:
    """带参数化标题和体的动态步骤。"""
    suffix = " -> Complete" if confidence == "certain" else ""
    title = f"Iterative Refinement (Iteration {iteration}){suffix}"
    body = INSTRUCTIONS.format(iteration=iteration, max_iter=MAX_ITERATIONS)
    return (title, body)


# 动态步骤字典——引用上方定义的函数
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

- 大多数步骤共享相同结构（title + 固定体）
- 少数步骤需要参数来构建 title 或体
- 参数在所有动态步骤中是统一的

优势：

- 静态步骤的紧凑表示（每个步骤一行）
- 动态步骤的清晰可读函数
- `format_output()` 中统一的派发点
- 遵循「书本模式」（所有引用均解析到上方的定义）

### 反模式：Action Factories

```python
# BAD - 不必要的间接层
def technique_review_actions(for_ecosystem=False):
    base = ["For each technique...", "1. QUOTE the trigger", ...]
    if for_ecosystem:
        base.append("Note techniques across prompts")
    return base
```

替换为：

```python
# GOOD - 文本在调用处直接可见
TECHNIQUE_REVIEW = (
    "For each technique in the Technique Selection Guide:\n"
    "1. QUOTE the trigger condition from the table\n"
    "2. QUOTE text from the target prompt that matches\n"
    "3. Verdict: APPLICABLE or NOT APPLICABLE"
)

TECHNIQUE_REVIEW_ECOSYSTEM = (
    TECHNIQUE_REVIEW + "\n"
    "Note techniques that apply to multiple prompts."
)

# 用法：用 + 组合
body = (
    "...\n"
    + TECHNIQUE_REVIEW_ECOSYSTEM + "\n"
    "..."
)
```

只有当存在复杂的条件逻辑（多个 if/else 分支）时，返回 prompt 片段的函数才是合理的。即便如此，它们也应存在于 skill 中，而非共享库中。

## 共享库

位置：`skills/lib/workflow/prompts/`

只有被 3 个及以上 skill 以相同语义使用的抽象才放这里：

```
prompts/
    __init__.py         # 重新导出
    subagent.py         # dispatch 模板
    step.py             # format_step()
    file.py             # format_file_content()
```

### subagent.py

通过 Task 工具派发子 agent 的三种 dispatch 模式：

- `subagent_dispatch(agent_type, command, prompt="", model=None)` —— 单个顺序 dispatch
- `template_dispatch(agent_type, template, targets, command, ...)` —— 并行 SIMD（相同模板，N 个目标，用 $var 替换）
- `roster_dispatch(agent_type, agents, command, shared_context="", ...)` —— 并行 MIMD（共享上下文 + 独立任务）

构建块（同样导出）：

- `task_tool_instruction(agent_type, model)` —— 如何使用 Task 工具
- `sub_agent_invoke(cmd)` —— 派发的 agent 运行的命令
- `parallel_constraint(count)` —— MANDATORY_PARALLEL 强制

### step.py

```python
def format_step(body: str, next_cmd: str = "", title: str = "") -> str:
    """组装完整的工作流步骤。

    Args:
        body: Prompt 内容（自由格式文本）
        next_cmd: 下一步运行的命令（最终步骤传空字符串）
        title: 可选标题，渲染为 "TITLE\\n======\\n\\n" 头部

    Returns:
        完整步骤输出的纯文本
    """
```

### file_content.py

```python
def format_file_content(path: str, content: str) -> str:
    """将文件内容嵌入 prompt。

    使用 4 个反引号围栏，以处理包含三重反引号的内容。
    """
```

## 两种调用概念

代码库中存在两种不同的「invoke」场景：

**子 agent invoke**（subagent.py 中的 `sub_agent_invoke()`）：出现在 dispatch prompt 内部。告知被派发的 agent 创建后应运行哪条命令。

**父级 invoke_after**（`format_step()` 中）：出现在体之后，作为步骤的终止指令。告知当前 agent 下一步运行什么。

一个 dispatch 步骤同时具备两者：

- 体包含一个带子 agent 调用命令的 dispatch prompt
- 步骤以父级的 invoke_after 结束，指明子 agent 返回后的后续操作

## 核心抽象：步骤

每个工作流步骤都有相同的基本结构：

```
[body]

[invoke_after]
```

就这样。两个部分：

1. **body**：实际的 prompt 内容，自由格式文本。
2. **invoke_after**：LLM 接下来应运行的命令。可选（最终步骤为空）。
