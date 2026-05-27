# Planner Skill 设计意图

planner skill 的权威设计规范。本文档阐明系统「为什么」如此运作。实现必须严格遵循本规范。

## 设计哲学

三条原则贯穿始终:

**提前消除歧义**:业务决策发生在规划阶段,在写代码之前。执行是机械性的。关于需求、架构或方案的问题在规划期间就要给出答案,而不是留到实现时才发现。

**捕获隐性知识**:决策、设计理由和上下文写入状态文件,让任何 agent 都能理解「为什么」,而不只是「做什么」。子 agent 接手工作时读取状态文件,即可获得完整上下文。信息绝不只存在于对话历史中。

**质量优先于速度**:LLM 会出错。多道 QR 质量门配合迭代循环,在错误扩散前将其拦截。本 skill 明确以执行时间换取正确性。

## 状态文件

所有状态变更(初始上下文捕获除外)均通过 Python 脚本完成。编排器派发子 agent;子 agent 调用脚本;脚本生成 prompt;LLM 执行工作并写入状态。

### context.json

由编排器在步骤 2(context-verify)创建。持久化用户提供的规划上下文以供子 agent 交接。

```json
{
  "task_spec": ["goal sentence", "scope: dir/module", "out-of-scope: X"],
  "constraints": ["MUST: X", "SHOULD: Y"],
  "entry_points": ["file:function - why relevant"],
  "rejected_alternatives": ["alternative - why dismissed"],
  "current_understanding": ["how system works", "bug: symptom + repro"],
  "assumptions": ["inference (H/M/L confidence)"],
  "invisible_knowledge": ["design rationale", "invariants", "tradeoffs"],
  "user_quotes": ["verbatim quote with context"],
  "reference_docs": ["doc/spec.md - what it specifies"]
}
```

所有字段均为字符串数组。空数组可接受;字段缺失则不可接受。

**QR 工作流访问权**:context.json 对所有 QR 子 agent(decompose、verify、fix)以只读方式开放,作为针对原始用户需求进行语义校验的参考。这使 QR agent 不仅能验证结构正确性,还能验证与用户意图的一致性。

### plan.json

主状态文件。在步骤 1(plan-init)创建为骨架,在规划各阶段中持续变更。

**无 schema 版本控制**:状态文件(context.json、plan.json、qr-\*.json)是临时性的,在单次规划会话内创建并消费。schema 版本控制对短生命周期产物只增加复杂度而无收益。Pydantic v2 模型位于 `shared/schema.py`。

```
Plan
  overview
    problem: string       -- 我们在解决什么问题
    approach: string      -- 我们如何解决

  planning_context
    decisions: Decision[]
      id: "DL-001"
      decision: string
      reasoning: string   -- 用 -> 符号表示的逻辑推理链
                          -- 例如 "高调用量 -> bcrypt 太慢 -> 改用 HMAC-SHA256"

    rejected_alternatives: RejectedAlternative[]
      alternative: string
      reason: string
      decision_ref: "DL-XXX"

    constraints: string[] -- 自由格式，例如 "MUST: 支持 Python 3.9+（用户指定）"

    risks: Risk[]
      risk: string
      mitigation: string
      anchor: string | null       -- 若位置相关则填 "file:L###-L###"
      decision_ref: "DL-XXX" | null

  invisible_knowledge
    system: string        -- 架构、数据流、结构设计理由（散文形式）
    invariants: string[]  -- 必须保持的属性
    tradeoffs: string[]   -- 已知的权衡取舍

  diagram_graphs: DiagramGraph[]   -- 由 Architect 填充（IR），由 TW 渲染
    id: "DIAG-001"
    type: "architecture" | "state" | "sequence" | "dataflow"
    scope: string         -- "overview" | "invisible_knowledge" | "milestone:M-XXX"
    title: string
    nodes: DiagramNode[]
      id: string          -- "node-001"
      label: string       -- 自由格式，例如 "gRPC Server"
      type: string | null -- 自由格式，例如 "service", "database", "queue"
    edges: DiagramEdge[]
      source: string      -- node id（已验证：必须存在）
      target: string      -- node id（已验证：必须存在）
      label: string       -- 自由格式，例如 "validates", "sends", "reads"
      protocol: string | null  -- 自由格式，例如 "gRPC", "HTTP"
    ascii_render: string | null  -- 由 TW 填充，渲染前为 null

  milestones: Milestone[]
    id: "M-001"
    name: string
    files: string[]
    requirements: string[]
    acceptance_criteria: string[]
    tests: string[]       -- 自由格式条目，例如：
                          -- "file:tests/test_auth.py"
                          -- "scenario:EDGE empty token returns 401"
                          -- "skip:no integration environment"

    code_intents: CodeIntent[]  -- 由 Architect 填充
      id: "CI-001"
      file: string
      behavior: string    -- 代码应该做什么，包含函数/参数
      decision_refs: string[]

    code_changes: CodeChange[]  -- 由 Developer 填充，由 TW 修订
      intent_ref: "CI-XXX" | null  -- 跨模块文档（README.md）时为 null
      file: string
      diff: string                 -- unified diff，包含所有文档
      comments: string             -- 变更级别上下文，可引用决策

    is_documentation_only: bool
    delegated_to: string | null

  waves: Wave[]
    id: "W-001"
    milestones: string[]  -- M-XXX 引用
```

Wave 按数组顺序执行。W-001 中所有里程碑完成后,W-002 才开始。同一 wave 内的里程碑可并行执行。

交叉引用验证:`Plan.validate_refs()` 检查:

- `code_changes.intent_ref` -> `code_intents.id`(同一里程碑内,非 null 时)
- `code_intents.decision_refs` -> `decisions.id`
- `rejected_alternatives.decision_ref` -> `decisions.id`
- `risks.decision_ref` -> `decisions.id`
- `diagram_graphs.edges.source` -> `diagram_graphs.nodes.id`(同一图内)
- `diagram_graphs.edges.target` -> `diagram_graphs.nodes.id`(同一图内)
- `diagram_graphs.scope` -> `milestones.id`(当 scope 为 `milestone:M-XXX` 时)

### qr-{phase}.json

临时 QR 状态文件。在 QR 拆解阶段创建。阶段通过后删除。共五个阶段:plan-design、plan-code、plan-docs、impl-code、impl-docs。

```json
{
  "phase": "plan-design",
  "iteration": 1,
  "items": [
    {
      "id": "qa-001",
      "scope": "*",
      "check": "Description of what to verify",
      "status": "TODO",
      "finding": null
    }
  ]
}
```

**顶级字段:**

- phase:此文件追踪的 QR 阶段
- iteration:当前 QR 循环计数(1 = 首次尝试,2+ = 失败后重试)

**条目字段:**

- id:阶段内的唯一标识符(qa-001、qa-002 ...)
- scope:自由格式位置说明符(见下方 Scope 哲学)
- check:可操作的验证指令
- status:"TODO" | "PASS" | "FAIL"
- finding:null 或解释字符串(FAIL 时必填)

数组中的条目数量是自适应的——由内容复杂度决定,而非预设范围。简单阶段可能条目较少;架构关注点众多的复杂阶段可能条目较多。

#### 迭代作为唯一真实来源

`iteration` 字段在文件内部追踪 QR 循环计数。这是迭代状态的权威来源——CLI flag 不追踪迭代。

**拆解步骤行为:**

每个 QR 阶段拆解只运行一次。首次调用生成所有 QR 条目;后续迭代跳过拆解,直接重新验证已有条目。

1. 检查 qr-{phase}.json 是否存在
2. 若存在:跳过拆解,直接进入验证步骤
3. 若不存在:运行 8 步拆解,创建文件(iteration: 1)
4. 输出下一步命令

**为什么每阶段只拆解一次:**
拆解定义了验证目标。每次迭代重新生成条目会造成移动靶:新条目引入与原始问题无关的新失败,导致无法收敛。修复-验证循环需要稳定的条目集才能终止。

**为什么检查文件存在性而非迭代次数:**
存在性表示「拆解已完成」;迭代次数表示「验证循环计数」。检查迭代次数会将拆解与验证进度耦合(错误的抽象)。

**迭代语义:**

迭代计数器追踪验证循环,而非拆解循环:

- iteration=1:初始拆解后的首次验证
- iteration=2+:修复后的重新验证(由验证步骤在 RETRY 时递增)

手动重新拆解:删除 qr-{phase}.json 以强制全新拆解。

**为什么基于文件的迭代:**

- 拆解脚本从文件状态以编程方式确定迭代次数
- 质量门/路由步骤只需以 --state-dir 调用工作步骤(无需迭代参数)
- 单一真实来源消除了 CLI 参数与文件内容之间的状态漂移
- 符合「状态检测优先于 flag」的不变量

#### QR 文件路径可计算

qr-{phase}.json 的路径始终为 `{state_dir}/qr-{phase}.json`。脚本从 --state-dir 和阶段名称计算此路径;无需 CLI flag 显式传递路径。

**路由器检测逻辑:**

```python
def detect_fix_mode(state_dir: str, phase: str) -> tuple[bool, int]:
    """Check if QR file exists with failures. Return (is_fix_mode, iteration)."""
    qr_path = Path(state_dir) / f"qr-{phase}.json"
    if not qr_path.exists():
        return False, 1
    qr_state = json.loads(qr_path.read_text())
    has_failures = any(item.get("status") == "FAIL" for item in qr_state.get("items", []))
    iteration = qr_state.get("iteration", 1)
    return has_failures, iteration
```

**对质量门路由的影响:**

质量门步骤只用 --state-dir 循环回工作步骤。工作步骤的路由器检查 qr-{phase}.json 以判断是执行还是修复工作流。这消除了编排器追踪失败状态的责任。

#### Scope 哲学

QA 条目分为两类:

1. **有范围的检查**:适用于特定代码位置(文件、函数、行范围)
2. **全局检查**:适用于整个产物(质量方面、一致性规则)

无需为文件、行、组件、质量方面等分别设置字段,单一自由格式 `scope` 字段涵盖所有情况。LLM 按需填写适当粒度:

| Scope 值                   | 含义                               |
| -------------------------- | ---------------------------------- |
| `*`                        | 全局检查——适用于所有地方           |
| `file:src/auth.py`         | 整个文件                           |
| `file:src/auth.py:L10-L50` | 特定行范围                         |
| `function:validate_token`  | 具名函数(任意文件)                |
| `component:auth-flow`      | 跨文件的架构组件                   |

这信任 LLM 的散文理解能力。拆解 agent 写出符合人类描述位置习惯的 scope。验证 agent 读取 scope 就知道去哪里查。无需刚性分类体系。

**Prompt 生成**:脚本将 scope 值原样输出到验证 prompt。示例 prompt 片段:「在范围 `{scope}` 内验证以下内容:{check}」

#### QR 状态变更(cli/qr.py)

拆解步骤创建初始 qr-{phase}.json 文件后,所有后续变更均通过 QR CLI 脚本进行。Agent 不直接修改 JSON 文件——而是调用脚本更新条目状态。

**CLI 接口:**

```
python3 -m skills.planner.cli.qr --state-dir <dir> --qr-phase <phase> update-item <id> --status <status> [--finding <text>]

Arguments:
  --state-dir    State directory containing qr-{phase}.json (required)
  --qr-phase     One of: plan-design, plan-code, plan-docs, impl-code, impl-docs (required)
  --status       PASS or FAIL (required)
  --finding      Explanation text (required when FAIL, forbidden when PASS)
```

**调用示例:**

```bash
# 验证 agent 将条目标记为 PASS
python3 -m skills.planner.cli.qr --state-dir /tmp/state --qr-phase plan-design \
    update-item qa-001 --status PASS

# 验证 agent 将条目标记为 FAIL
python3 -m skills.planner.cli.qr --state-dir /tmp/state --qr-phase plan-design \
    update-item qa-003 --status FAIL --finding "Missing null check in validate_token()"
```

**状态语义:**

- 无明确状态的条目视为 TODO(拆解后的初始状态)
- TODO 表示「尚未验证」——拆解 agent 创建条目,验证 agent 评估条目
- PASS 表示「验证通过」——条目不可变,进一步更新会报错
- FAIL 表示「验证失败」——finding 说明原因,修复后条目可转为 PASS

**有效状态转换:**

```
TODO -> PASS         (首次尝试验证通过)
TODO -> FAIL         (验证失败)
FAIL -> PASS         (修复后重新验证通过)
FAIL -> FAIL         (重新验证仍失败，finding 可更新)
PASS -> *            (ERROR: 条目一旦通过即不可变)
```

**为什么通过脚本进行状态变更:**

并行验证 agent 同时更新同一个 qr-{phase}.json 文件。直接写 JSON 会导致竞态条件(无锁的读-改-写 = 更新丢失)。CLI 脚本使用文件锁(fcntl.flock)和原子写入(tmp + rename)来安全地串行化并发更新。

**实现复用:**

脚本复用 `shared/qr/utils.py` 中的辅助函数:

- `load_qr_state(state_dir, phase)` -- 加载并解析 qr-{phase}.json
- `get_qr_item(qr_state, item_id)` -- 按 ID 查找条目
- `get_qr_iteration(state_dir, phase)` -- 获取当前迭代次数(文件不存在时为 1)
- `has_qr_failures(state_dir, phase)` -- 检查文件是否存在且含有 FAIL 条目

CLI 脚本额外增加了带锁的原子保存(utils.py 中没有,因为只有 CLI 需要)。路由器脚本使用 `has_qr_failures()` 检测修复模式,无需加载完整状态。

#### 计划状态变更(cli/plan.py)

plan.json 实体通过 plan CLI 脚本配合比较并交换(CAS)版本控制进行变更。Agent 不直接修改 plan.json——而是调用 set-X 命令来强制版本一致性。

**CAS 版本控制模型:**

每个可版本化实体有一个 `version: int` 字段,起始值为 1。更新时需提供当前版本;版本不匹配时脚本拒绝操作。

- 创建:省略 `--id`,省略 `--version` -> 自动生成 ID,version=1
- 更新:提供 `--id`,必须提供 `--version` -> 验证版本匹配,成功后递增

**为什么使用 CAS:**

1. **防止竞态条件**:多个 agent 无法盲目覆盖彼此的变更
2. **强制写前读取**:Agent 必须读取当前状态以获得版本号
3. **冲突检测**:过期读取会立即以版本不匹配的形式暴露

**CLI 接口:**

```
python3 -m skills.planner.cli.plan --state-dir <dir> set-intent \
    --milestone M-001 --file path.py --behavior "description"    # 创建

python3 -m skills.planner.cli.plan --state-dir <dir> set-intent \
    --id CI-M-001-001 --version 1 --behavior "updated"           # 更新
```

**版本不匹配输出:**

版本不匹配时,CLI 打印完整的当前实体 JSON 和重试说明。确保 agent 在失败时始终拥有最新状态:

```xml
<version_mismatch_error>
  <entity_id>CI-M-001-001</entity_id>
  <provided_version>1</provided_version>
  <current_version>2</current_version>
  <current_entity>
    {"id": "CI-M-001-001", "version": 2, "file": "...", ...}
  </current_entity>
  <action>Integrate your changes into the current entity above and retry with --version 2</action>
</version_mismatch_error>
```

**成功输出:**

成功时,CLI 打印实体 ID 和新版本号:

```xml
<entity_result>
  <id>CI-M-001-001</id>
  <version>2</version>
  <operation>updated</operation>
</entity_result>
```

**统一的 set-X 命令:**

| 命令               | 实体          | 角色      |
| ------------------ | ------------- | --------- |
| set-milestone      | Milestone     | architect |
| set-intent         | CodeIntent    | architect |
| set-decision       | Decision      | architect |
| set-diagram        | DiagramGraph  | architect |
| add-diagram-node   | DiagramNode   | architect |
| add-diagram-edge   | DiagramEdge   | architect |
| set-change         | CodeChange    | developer |
| set-doc            | Documentation | tw        |
| set-diagram-render | DiagramGraph  | tw        |

已废弃的 add-X 和 update-X 命令已移除。所有变更均使用带 CAS 版本控制的 set-X。

## 文档模型

文档按范围分为两类:

### 代码本地文档

存在于源文件中的文档。由 Technical Writer 通过修订 diff 来交付。

| 层级           | 内容                                      | 位置        | 示例                                                   |
| -------------- | ----------------------------------------- | ----------- | ------------------------------------------------------ |
| 模块注释       | 文件级别:这里有什么                       | 文件顶部    | `# auth.py -- Token validation and session management` |
| Docstring      | 函数级别:做什么、何时使用                 | 函数上方    | `def validate(token): """Validate JWT..."""`           |
| 内联注释       | 逻辑说明:算法、决策                       | 代码上方    | `# xxhash for speed; collisions acceptable (DL-003)`   |

TW 修订 Developer 的 diff 来添加这些内容。diff 本身就是文档的交付机制。

### 跨模块文档

跨越多个文件/组件的文档。创建为独立的 code_changes,`intent_ref: null`。

| 类型      | 内容                                    | 处理方式                            |
| --------- | --------------------------------------- | ----------------------------------- |
| README.md | 设计决策、架构概述                      | 带 `intent_ref: null` 的 code_change |

示例:

```json
{
  "intent_ref": null,
  "file": "src/auth/README.md",
  "diff": "--- /dev/null\n+++ b/src/auth/README.md\n...",
  "comments": "Cross-cutting auth design decisions"
}
```

README.md 文件不实现代码行为,因此 `intent_ref` 为 null。它们仍然是 code_changes,因为它们是计划中追踪的文件变更。

## 图表模型

图表是人类理解计划实现内容的主要入口。一张设计精良的图表能在 10 秒内回答「这是做什么的?」。

### 图表类型

| 类型         | 使用时机                                   | 结构                            |
| ------------ | ------------------------------------------ | ------------------------------- |
| architecture | 服务、API、SDK、组件边界                   | 带方向箭头的方框                |
| state        | 显式状态机、协议生命周期                   | 带标记边的命名状态              |
| sequence     | 多方请求/响应、时间顺序                    | 垂直时间线、水平箭头            |
| dataflow     | ETL 管道、流处理、数据转换                 | 从左到右的分阶段流              |

默认使用 `architecture`。只有当计划明确涉及状态机、多方协议或数据管道时才使用其他类型。

### 图表范围

范围决定图表出现在渲染输出的哪个位置:

| 范围                  | 渲染位置               | 目的                                   |
| --------------------- | ---------------------- | -------------------------------------- |
| `overview`            | 概述区段之后           | 「主视图」——第一个视觉上下文          |
| `invisible_knowledge` | 隐性知识区段           | 供 LLM 建立架构心智模型               |
| `milestone:M-XXX`     | 里程碑顶部             | 这个特定里程碑新增了什么              |

每个范围允许有多张图表。列表中第一张匹配范围的图表为主图表。

### 两阶段工作流

图表使用图 IR(中间表示)将正确性与渲染分离:

**阶段 1 —— Architect(plan-design-work):**

- 创建带节点和边的 diagram_graphs
- 验证语义正确性:无孤立节点,边引用有效
- ascii_render 保持为 null

**阶段 2 —— Technical Writer(plan-docs-work):**

- 将每个 diagram_graph 渲染为 ASCII
- 填充 ascii_render 字段
- 验证格式:宽度、方框对齐

这种分离确保 Architect 专注于传达什么(图结构),而 TW 专注于如何传达(视觉渲染)。

### ASCII 约定

图表渲染为定宽 ASCII,保证通用可移植性(cat、vim、git diff、终端):

```
+------------------+     +------------------+
| Component A      | --> | Component B      |
| (description)    |     | (description)    |
+------------------+     +------------------+
        |
        v
+------------------+
| Component C      |
+------------------+
```

语法:

- 方框角:``+``
- 水平边:``-``
- 垂直边:``|``
- 箭头:``v``、``^``、``<``、``>``、``-->``、``<--``
- 边标签:内联在箭头上或括号形式

目标宽度:最多 80 字符。比终端宽的图表会换行并失去价值。

### 跳过标准

并非所有计划都需要图表。以下情况跳过图表生成:

- 纯重构(无新组件)
- 单文件变更
- 纯文档里程碑
- 概述中不含结构性关键词(services、layers、flow、protocol)

跳过时,diagram_graphs 保持为空。这是有效状态。

### 文档工作流

**Developer(plan-code-work)**:创建实现 code_intents 的 code_changes。可包含显而易见的注释。

**TW(plan-docs-work)**:

1. 修订每个 code_change 的 diff,添加模块注释、docstring、内联注释
2. 为 README.md 文件创建新的 code_changes(`intent_ref: null`)

code_changes 上的 `comments` 字段用于不属于代码本身的变更级别上下文。可内联引用决策:「为速度使用 xxhash(DL-003)」。

## 变更归属

| 文件                | 步骤 | Agent            | 变更内容                                                                    |
| ------------------- | ---- | ---------------- | --------------------------------------------------------------------------- |
| plan.json           | 1    | orchestrator     | 创建骨架                                                                    |
| context.json        | 2    | orchestrator     | 创建并冻结                                                                  |
| plan.json           | 3    | architect        | 添加概述、里程碑、code_intents、决策、diagram_graphs(IR)                    |
| qr-plan-design.json | 4    | quality-reviewer | 创建 status: TODO 的条目                                                    |
| qr-plan-design.json | 5    | quality-reviewer | 将各条目状态更新为 PASS/FAIL                                                |
| plan.json           | 7    | developer        | 添加 code_changes                                                           |
| qr-plan-code.json   | 8    | quality-reviewer | 创建 status: TODO 的条目                                                    |
| qr-plan-code.json   | 9    | quality-reviewer | 将各条目状态更新为 PASS/FAIL                                                |
| plan.json           | 11   | technical-writer | 修订 code_changes 的 diff 以添加文档,将 diagram_graphs 渲染为 ASCII        |
| qr-plan-docs.json   | 12   | quality-reviewer | 创建 status: TODO 的条目                                                    |
| qr-plan-docs.json   | 13   | quality-reviewer | 将各条目状态更新为 PASS/FAIL                                                |
| qr-plan-design.json | 6    | orchestrator     | 删除文件(全部 PASS)                                                        |
| qr-plan-code.json   | 10   | orchestrator     | 删除文件(全部 PASS)                                                        |
| qr-plan-docs.json   | 14   | orchestrator     | 删除文件(全部 PASS)                                                        |
| qr-impl-code.json   | E3   | quality-reviewer | 创建 status: TODO 的条目                                                    |
| qr-impl-code.json   | E4   | quality-reviewer | 将各条目状态更新为 PASS/FAIL                                                |
| qr-impl-code.json   | E5   | orchestrator     | 删除文件(全部 PASS)                                                        |
| qr-impl-docs.json   | E7   | quality-reviewer | 创建 status: TODO 的条目                                                    |
| qr-impl-docs.json   | E8   | quality-reviewer | 将各条目状态更新为 PASS/FAIL                                                |
| qr-impl-docs.json   | E9   | orchestrator     | 删除文件(全部 PASS)                                                        |

## 工作流

### 规划工作流(orchestrator/planner.py)

共 14 步。将用户请求转化为可执行计划(IR)。

每个可 QR 的阶段遵循 4 步块模式:

- 工作步骤(1 个子 agent):基于状态检测执行或修复。子 agent 在返回前验证已写入的状态。
- QR 拆解(1 个子 agent):创建验证条目。子 agent 在返回前验证 qr-{phase}.json。
- QR 验证(N 个子 agent):通过批量派发并行验证条目。
- QR 路由(编排器):汇总结果,循环或推进。

```
步骤 1: plan-init
  动作: 创建 state_dir，写入 plan.json 骨架
  下一步: 步骤 2

步骤 2: context-verify
  动作: 将上下文写入 context.json，自验证完整性
  检查清单: 目标可用一句话表述、至少一个超出范围的项、
             至少一个约束（或明确「无」）、入口点已识别
  下一步: 步骤 3

步骤 3: plan-design-work
  Agent: architect
  脚本: architect/plan_design.py（路由器）
  路由: 若 qr-plan-design.json 有 FAIL 条目 -> architect/plan_design_qr_fix.py
        否则 -> architect/plan_design_execute.py
  输出: plan.json 含概述、里程碑、code_intents、决策、diagram_graphs（仅 IR）
  验证: 子 agent 在返回前验证 plan.json 符合 schema
  下一步: 步骤 4

步骤 4: plan-design-qr-decompose
  Agent: quality-reviewer
  脚本: quality_reviewer/plan_design_qr_decompose.py
  输出: qr-plan-design.json 含 status: TODO 的条目
  输出: parallel_dispatch 块，列出所有 --qr-item ID
  下一步: 步骤 5

步骤 5: plan-design-qr-verify
  Agent: quality-reviewer（N 个并行实例）
  脚本: quality_reviewer/plan_design_qr_verify.py --qr-items {ids}
  输入: 编排器脚本解析步骤 4 的 parallel_dispatch，按组批量分配条目
  输出: 每个 agent 验证其批次，将 qr-plan-design.json 中的条目更新为 PASS/FAIL
  下一步: 步骤 6

步骤 6: plan-design-qr-route
  动作: 编排器脚本从 qr-plan-design.json 确定路由
  路由: 全部 PASS -> 删除 qr 文件，推进到步骤 7
        任意 FAIL -> 循环回步骤 3（路由器将派发到 qr_fix）

步骤 7: plan-code-work
  Agent: developer
  脚本: developer/plan_code.py（路由器）
  路由: 若 qr-plan-code.json 有 FAIL 条目 -> developer/plan_code_qr_fix.py
        否则 -> developer/plan_code_execute.py
  输出: 里程碑中添加 code_changes[]
  下一步: 步骤 8

步骤 8: plan-code-qr-decompose
  Agent: quality-reviewer
  脚本: quality_reviewer/plan_code_qr_decompose.py
  输出: qr-plan-code.json 含 status: TODO 的条目
  输出: parallel_dispatch 块，列出所有 --qr-item ID
  下一步: 步骤 9

步骤 9: plan-code-qr-verify
  Agent: quality-reviewer（N 个并行实例）
  脚本: quality_reviewer/plan_code_qr_verify.py --qr-item {id}
  输出: 每个 agent 将 qr-plan-code.json 中的一个条目更新为 PASS/FAIL
  下一步: 步骤 10

步骤 10: plan-code-qr-route
  路由: 全部 PASS -> 删除 qr 文件，推进到步骤 11
        任意 FAIL -> 循环回步骤 7

步骤 11: plan-docs-work
  Agent: technical-writer
  脚本: technical_writer/plan_docs.py（路由器）
  路由: 若 qr-plan-docs.json 有 FAIL 条目 -> technical_writer/plan_docs_qr_fix.py
        否则 -> technical_writer/plan_docs_execute.py
  输出: code_changes 的 diff 追加文档；添加 README.md 变更；
        每张图表的 diagram_graphs.ascii_render 已填充
  下一步: 步骤 12

步骤 12: plan-docs-qr-decompose
  Agent: quality-reviewer
  脚本: quality_reviewer/plan_docs_qr_decompose.py
  输出: qr-plan-docs.json 含 status: TODO 的条目
  输出: parallel_dispatch 块，列出所有 --qr-item ID
  下一步: 步骤 13

步骤 13: plan-docs-qr-verify
  Agent: quality-reviewer（N 个并行实例）
  脚本: quality_reviewer/plan_docs_qr_verify.py --qr-item {id}
  输出: 每个 agent 将 qr-plan-docs.json 中的一个条目更新为 PASS/FAIL
  下一步: 步骤 14

步骤 14: plan-docs-qr-route
  路由: 全部 PASS -> 删除 qr 文件，计划已批准
        任意 FAIL -> 循环回步骤 11
```

### 执行工作流(orchestrator/executor.py)

共 10 步。实现已批准的计划。

```
步骤 1: exec-init
  动作: 分析计划，构建 wave 依赖图

步骤 2: impl-code-work
  Agent: developer（每个 wave 最多 4 个并行）
  脚本: developer/exec_implement.py（路由器）
  路由: 若 qr-impl-code.json 有 FAIL 条目 -> developer/exec_implement_qr_fix.py
        否则 -> developer/exec_implement_execute.py
  输出: 代码变更应用到文件
  下一步: 步骤 3

步骤 3: impl-code-qr-decompose
  Agent: quality-reviewer
  脚本: quality_reviewer/impl_code_qr_decompose.py
  输出: qr-impl-code.json 含 status: TODO 的条目
  输出: parallel_dispatch 块，列出所有 --qr-item ID
  下一步: 步骤 4

步骤 4: impl-code-qr-verify
  Agent: quality-reviewer（N 个并行实例）
  脚本: quality_reviewer/impl_code_qr_verify.py --qr-item {id}
  输出: 每个 agent 将 qr-impl-code.json 中的一个条目更新为 PASS/FAIL
  下一步: 步骤 5

步骤 5: impl-code-qr-route
  路由: 全部 PASS -> 删除 qr 文件，推进到步骤 6
        任意 FAIL -> 循环回步骤 2

步骤 6: impl-docs-work
  Agent: technical-writer
  脚本: technical_writer/exec_docs.py（路由器）
  路由: 若 qr-impl-docs.json 有 FAIL 条目 -> technical_writer/exec_docs_qr_fix.py
        否则 -> technical_writer/exec_docs_execute.py
  输出: 文档写入文件
  下一步: 步骤 7

步骤 7: impl-docs-qr-decompose
  Agent: quality-reviewer
  脚本: quality_reviewer/impl_docs_qr_decompose.py
  输出: qr-impl-docs.json 含 status: TODO 的条目
  输出: parallel_dispatch 块，列出所有 --qr-item ID
  下一步: 步骤 8

步骤 8: impl-docs-qr-verify
  Agent: quality-reviewer（N 个并行实例）
  脚本: quality_reviewer/impl_docs_qr_verify.py --qr-item {id}
  输出: 每个 agent 将 qr-impl-docs.json 中的一个条目更新为 PASS/FAIL
  下一步: 步骤 9

步骤 9: impl-docs-qr-route
  路由: 全部 PASS -> 删除 qr 文件，推进到步骤 10
        任意 FAIL -> 循环回步骤 6

步骤 10: wave-next
  动作: 推进到下一个 wave，对每个 wave 重复步骤 2-9
  路由: 还有 wave -> 循环回步骤 2
        所有 wave 完成 -> 执行完成
```

## 脚本组织

脚本遵循路由器-派发模式。每个可 QR 的阶段有:

- 路由器脚本:检测状态,派发到对应工作流
- 执行脚本:首次执行工作流
- QR 修复脚本:QR 失败后的修复工作流
- QR 拆解脚本:创建验证条目
- QR 验证脚本:验证单个条目(用 --qr-item 调用)

```
skills/planner/
  orchestrator/
    planner.py       -- 14 步规划工作流
    executor.py      -- 12 步执行工作流
  architect/
    plan_design.py            -- 路由器（检测状态，派发）
    plan_design_execute.py    -- 首次执行（6 步）
    plan_design_qr_fix.py     -- QR 失败后的修复工作流
  developer/
    plan_code.py              -- 路由器
    plan_code_execute.py      -- 首次执行（4 步）
    plan_code_qr_fix.py       -- QR 失败后的修复工作流
    exec_implement.py         -- 路由器
    exec_implement_execute.py -- 实现（4 步）
    exec_implement_qr_fix.py  -- QR 失败后的修复工作流
  technical_writer/
    plan_docs.py              -- 路由器
    plan_docs_execute.py      -- 首次执行（6 步）
    plan_docs_qr_fix.py       -- QR 失败后的修复工作流
    exec_docs.py              -- 路由器
    exec_docs_execute.py      -- impl-docs（6 步）
    exec_docs_qr_fix.py       -- QR 失败后的修复工作流
  quality_reviewer/
    plan_design_qr_decompose.py -- 拆解工作流
    plan_design_qr_verify.py    -- 单条目验证
    plan_code_qr_decompose.py   -- 拆解工作流
    plan_code_qr_verify.py      -- 单条目验证
    plan_docs_qr_decompose.py   -- 拆解工作流
    plan_docs_qr_verify.py      -- 单条目验证
    impl_code_qr_decompose.py   -- 拆解工作流
    impl_code_qr_verify.py      -- 单条目验证
    impl_docs_qr_decompose.py   -- 拆解工作流
    impl_docs_qr_verify.py      -- 单条目验证
  shared/
    schema.py         -- Pydantic v2 schema（context、plan、qr），验证
    resources.py      -- 路径辅助函数、资源提供者
    constraints.py    -- 约束构建器
    gates.py          -- 质量门输出构建器
    qr/               -- QR 子系统工具
      utils.py        -- QR 状态加载、条目提取
```

## CLI 接口

所有脚本接受一组公共参数。QR 相关状态基于文件,而非 CLI。

**通用参数(所有脚本):**

| 参数          | 必需   | 说明                            |
| ------------- | ------ | ------------------------------- |
| `--step`      | 是     | 当前步骤编号(从 1 开始)         |
| `--state-dir` | 是\*   | 状态目录路径                    |

\*编排器步骤 1 创建 state_dir;后续步骤需要它。

**QR 验证参数:**

| 参数         | 必需   | 说明                                             |
| ------------ | ------ | ------------------------------------------------ |
| `--qr-item`  | 是\*   | 要验证的单个条目 ID(例如 qa-001)                 |
| `--qr-items` | 是\*   | 批量验证的逗号分隔条目 ID                        |

\*二者选其一。批量验证(--qr-items)将语义相关的条目分组给单个 agent,实现每个 agent 处理一个批次的并行派发。

**质量门步骤参数:**

| 参数          | 必需   | 说明                                                     |
| ------------- | ------ | -------------------------------------------------------- |
| `--qr-status` | 是     | 来自验证 agent 的汇总结论:"pass"/"fail"                  |

**明确禁止的参数:**

| 参数             | 禁止原因                                               |
| ---------------- | ------------------------------------------------------ |
| `--qr-fail`      | QR 文件路径可从 state_dir + phase 计算得出             |
| `--qr-iteration` | 迭代次数存储在 qr-{phase}.json 中,不通过参数传递       |

编排器从不传递失败路径或迭代计数。路由器和修复脚本直接从 qr-{phase}.json 读取这些信息。

## 不变量

**子 agent 不能启动子 agent**。只有编排器负责派发。保持审计追踪,防止隐藏依赖。

**子 agent 不能调用 AskUserQuestion**。需要用户输入的子 agent 通过 `<needs_user_input>` XML 暂停。编排器转发问题,然后携带答案重新调用子 agent。子 agent 不能被恢复;必须携带从状态文件恢复的上下文重新调用。

**编排器 LLM 从不读写状态文件**。编排器 LLM agent 不得对状态文件(plan.json、context.json、qr-{phase}.json)使用 Read()、Write() 或 Edit() 工具。上下文通过派发 prompt 传递。状态文件是子 agent 的领地。

注意:编排器 Python 脚本(planner.py、executor.py)可以在内部读取状态文件以实现可靠编排——例如用 `load_qr_state()` 确定哪些 QR 条目还需要派发。这是对 LLM 不可见的实现机制。该不变量适用于 LLM agent,而非生成 prompt 的 Python 代码。

**编排器是简单的调度器**。编排器基于状态标志(pass/fail)和步骤编号进行路由。它从不做质量判断(「计划看起来很全面」),从不在协议要求迭代时「无论如何继续」,也从不基于主观评估跳过步骤。如果子 agent 返回无效输出或工作流需要迭代,编排器机械地遵循协议。

**子 agent 自验证**。每个写入状态文件(plan.json、qr-{phase}.json)的子 agent 在返回编排器前必须验证已写入的文件。任何状态变更工作流的最后一步:

1. 加载刚写入的文件
2. 通过 `validate_state()` 验证 Pydantic schema
3. 若无效:就地修复,重新验证,循环直到有效
4. 若有效:格式化最终输出并返回

编排器绝不应看到 schema 验证错误。哲学:在问题发生后立即在源头检测。验证失败是子 agent 在交接前需要修复的 bug,而非编排器的关注点。

这对执行和修复工作流同样适用。architect 写入 plan.json 后进行验证。QR 修复 agent 更新 plan.json 后进行验证。编排器只接收有效状态。

**用户权威是绝对的**。Agent 的发现可能是错的。用户决策覆盖一切。

**始终运行脚本**。每个步骤都调用一个 Python 脚本。不做自由格式执行。脚本生成 prompt;LLM 执行工作。路由器脚本派发到工作流脚本;这仍然是基于脚本的执行。

**路由器仅在步骤 1 派发**。路由器脚本在步骤 1 检测状态(qr-{phase}.json 的存在和内容)并派发到对应工作流脚本。工作流脚本内的后续步骤不得派发到其他脚本。

**状态检测优先于 flag**。工作脚本从状态文件是否存在检测模式,而非从 CLI flag。如果 qr-{phase}.json 存在且有 FAIL 条目,路由器派发到修复工作流。编排器无论何种模式都派发到相同的步骤编号。

**无分布式 QR 状态**。所有 QR 状态存在于 qr-{phase}.json 中。QR 失败路径(--qr-fail)和迭代计数(--qr-iteration)的 CLI flag 不得存在。拆解步骤从文件读取/递增迭代;路由器从 state_dir + phase 计算 qr 文件路径。只有 --qr-status(编排器的汇总结论)和 --qr-item(验证 agent 被分配的条目)是有效的 QR 相关 CLI 参数。

**自适应条目生成**。拆解创建内容所需数量的条目。无固定数量或上限。8 步工作流在结构枚举穷尽且覆盖得到验证时自然终止。条目数量随计划复杂度变化。

**QR 拆解输出契约**。拆解脚本必须输出一个 `<parallel_dispatch>` 块供编排器解析以启动 N 个验证 agent。派发块通过 `lib/workflow/ast/` 的 AST 模块生成,使用三种节点类型:

- `SubagentDispatchNode`:单 agent 派发(顺序工作流)
- `TemplateDispatchNode`:带参数化模板的并行派发(SIMD 模式)
- `RosterDispatchNode`:带唯一 prompt 的并行派发(MIMD 模式)

渲染格式:

```xml
<parallel_dispatch agent="quality-reviewer" count="N">
  <groups>
    <group id="component-auth" items="qa-001,qa-002,qa-003">Auth component checks</group>
    <group id="umbrella" items="qa-010,qa-011">Cross-cutting checks</group>
  </groups>
  <template>
    <invoke cmd="python3 -m skills.planner.quality_reviewer.{phase}_qr_verify --step 1 --state-dir {state_dir} --qr-items $GROUP_ITEMS" />
  </template>
</parallel_dispatch>
```

**QR 文件生命周期**。qr-{phase}.json 由拆解步骤创建,由验证 agent 更新,由路由步骤在 PASS 时删除。文件存在表示「QR 进行中」;文件不存在表示「尚未执行 QR」或「QR 通过并已清理」。

**QR 迭代上限**。每个 QR 阶段最多 5 次迭代。所有严重级别在所有迭代中都会阻塞。

## QR 工作流

每个 QR 块由 4 个编排器步骤组成:

**拆解步骤(1 个子 agent):**

```
python3 -m skills.planner.quality_reviewer.<phase>_qr_decompose --step 1 --state-dir {state_dir}
```

子 agent 通过 8 步认知工作流探索被审查的产物,自适应生成验证条目(数量由内容决定,无预设上限),写入 qr-{phase}.json,所有条目状态为 TODO。输出供编排器解析的 parallel_dispatch 块。

### 8 步拆解工作流

拆解遵循「自顶向下再自底向上」的方式生成验证条目。整体式头脑风暴捕获结构枚举无法发现的横切关注点(整体方案有效性、隐式需求、集成风险)。结构枚举作为完整性验证,而非生成来源。

**步骤 1:吸收上下文**
读取 plan.json 和 context.json。用 2-3 句话总结理解。明确计划要实现什么、这个阶段的成功标准是什么。尚不生成条目。

**步骤 2:整体关注点(自顶向下)**
自由头脑风暴:「如果审查这个阶段输出,我会检查什么?」捕获高层有效性、横切模式、质量方面和风险。输出是未经筛选的关注点要点列表。这一步识别结构枚举无法看到的内容。

**步骤 3:结构枚举(自底向上)**
列出计划中这个阶段实际存在的内容。按阶段区分:plan-design 枚举决策/约束/风险,plan-code 枚举 code_changes,impl-code 枚举 acceptance_criteria。输出是带 ID 和计数的结构化枚举。这成为步骤 7 的完整性检查清单。

**步骤 4:差距分析**
比较步骤 2 的关注点与步骤 3 的元素。识别哪些关注点需要伞式条目(横切)、哪些映射到具体元素、哪些元素需要针对性条目,以及两个方向的差距。

**步骤 5:生成初始条目**
使用「伞式 + 具体」模式创建条目。关键关注点同时生成一个广泛的兜底条目(scope: "\*")和具体的针对性条目(scope: 元素引用)。这种有意的重叠确保异常值被伞式条目捕获,而已知关键方面得到明确验证。重叠覆盖可以接受;存在间隙则不可接受。无固定条目数量——根据内容所需生成。

**步骤 6:原子性检查**
审查每个条目:只测试一件事吗?如果通过/失败无歧义且不能「半通过」,则条目是原子性的。如果非原子性条目作为伞式条目捕获异常值,则可以接受。

拆分标准:只有当条目既是非原子性又是关键性时才拆分。关键性判断:

- 与 MUST 严重级别相关:知识损失、生产可靠性
- planning_context 中的架构决策
- 横切的错误处理或安全性
- 公共 API 契约变更

非关键关注点(内部实现细节、格式、优化)保留为伞式条目以获得更广泛覆盖。

拆分时:创建具体条目并保留伞式条目。

**步骤 7:覆盖验证**
以步骤 3 的枚举作为检查清单。每个元素至少有一个覆盖条目?每个关注点至少有一个应对条目?如有不确定,添加条目。重叠优先于间隙。

**步骤 8:最终确定并写入**
写入带最终条目的 qr-{phase}.json。输出 parallel_dispatch 块。条目数量是过程自然产生的——无目标,无上限。内容决定数量。

### 自适应条目生成

工作流基于计划复杂度产生可变数量的条目。决策少、code_changes 简单的阶段产生较少条目。架构决策多、横切关注点多、代码模式复杂的阶段产生更多条目。

8 步工作流在无需人为限制的情况下自然终止:

- 步骤 3 将条目约束在计划中实际存在的内容范围内
- 步骤 7 在检查清单完成时终止
- 伞式条目覆盖多个关注点,无需 1:1 展开

带重叠的更多条目优先于带间隙的更少条目。

**验证步骤(N 个子 agent,并行):**

```
python3 -m skills.planner.quality_reviewer.<phase>_qr_verify --step 1 --state-dir {state_dir} --qr-items qa-001,qa-002
```

每个子 agent 接收一批语义相关的条目进行验证。条目由拆解步骤分组(例如按组件、按关注点或父子关系)。Agent 读取 qr 文件,验证每个被分配的条目,并将状态更新为 PASS 或 FAIL(附带 finding)。

**输出契约**:验证 agent 必须以以下之一结束:

- `PASS` —— 条目验证成功
- `FAIL: <reason>` —— 条目验证失败,必须给出原因

编排器通过 LLM 理解而非程序化方式解析此输出。格式错误的输出视为 FAIL。

**路由步骤(仅编排器):**

编排器脚本(非 LLM)在所有验证 agent 完成后读取 qr-{phase}.json,使用辅助函数检查是否还有失败:

```python
def get_pending_qr_items(state_dir: str, phase: str) -> list[str]:
    """Return item IDs that need processing (status TODO or FAIL)."""
```

脚本确定路由并为 LLM 生成适当的 prompt。LLM 只看到路由决策,而不是文件内容。

若无失败:删除 qr 文件,推进到下一个块。
若存在失败:循环回工作步骤。工作步骤的路由器将检测到带 FAIL 条目的 qr-{phase}.json 并派发到修复工作流。

**修复工作流(通过路由器):**

当编排器循环回工作步骤(例如步骤 3)时,路由器脚本:

1. 检查 qr-{phase}.json 是否存在
2. 若存在且有 FAIL 条目 -> 派发到 {phase}\_qr_fix.py
3. 修复脚本加载失败条目,引导 agent 修复问题
4. Agent 在修复后验证状态文件(与执行工作流相同的自验证要求)
5. 修复和验证后,工作流继续到拆解步骤(全新 QR)

## 上下文交接

编排器启动子 agent 时上下文会丢失。派发 prompt 必须包含所有必要上下文。子 agent 通过读取状态文件获得完整细节。

派发上下文类别:

- 任务规范:我们在构建什么、范围、超出范围的内容
- 约束:MUST/SHOULD/MUST-NOT
- 入口点:从哪里开始探索
- 被拒绝的替代方案:什么被否定了以及原因
- 假设:未经验证的推断
- 隐性知识:设计理由、不变量、权衡取舍
- 用户引言:用户的原话,尤其是纠正性内容

交接 prompt 应简洁。初始交接(步骤 2->3)包含完整细节。后续交接可以简短,因为子 agent 会读取状态文件。

**Prompt 中的 CLI 变更命令**:变更状态文件的子 agent 必须在其 prompt 中看到相关 CLI 命令。Agent 无法使用它不知道的工具。脚本将 CLI 用法示例作为 prompt 的一部分输出:

| 子 agent  | 需要展示的 CLI 命令                                                    |
| --------- | ---------------------------------------------------------------------- |
| architect | `cli.plan set-milestone`、`set-intent`、`set-decision`、`set-diagram`、 |
|           | `add-diagram-node`、`add-diagram-edge`                                 |
| developer | `cli.plan set-change`                                                  |
| tw        | `cli.plan set-doc`、`set-diagram-render`                               |
| qr-verify | `cli.qr update-item`                                                   |

architect 的示例 prompt 片段:

```
State Mutation:
  python3 -m skills.planner.cli.plan --state-dir {state_dir} set-intent \
      --milestone M-001 --file path.py --behavior "description"
```

## 问题转发

当子 agent 需要用户输入时:

1. 子 agent 将状态保存到 plan.json
2. 子 agent 输出 `<needs_user_input>` XML 并停止
3. 编排器检测到 XML,提取问题
4. 编排器调用 AskUserQuestion
5. 用户回答
6. 编排器携带累积的问答历史(在额外的 prompt 字段中)重新调用全新的子 agent
7. 新子 agent 读取 plan.json,结合用户答案继续

重新调用时,答案以 `<user_response>` 块提供:

```xml
<user_response>
  <answer header="Auth">JWT with refresh tokens</answer>
  <answer header="Scope">Not in initial implementation</answer>
</user_response>
```

子 agent 不能被恢复。必须重新调用并显式读取状态文件。
