# Prompt Engineer

Prompt 就是代码。它有 bug、边界情况和失败模式。本 skill 将 prompt 优化视为一门系统性学科——分析问题、应用有文档支撑的模式、并在提出改动时明确标注所依据的理论依据。

我在自己的工作流中使用它。这个 skill 本身也是用它自己优化出来的——当然。

## 使用场景

- 行为异常的子 agent 定义（agents/developer.md）
- 内嵌 prompt 表现不佳的 Python 脚本（skills/planner/scripts/planner.py）
- 产出结果不一致的多 prompt 工作流
- 任何没有达到预期效果的 prompt

## 工作原理

本 skill 会：

1. 读取 prompt 工程模式参考文档
2. 分析目标 prompt 存在的问题
3. 提出改动，并明确标注所依据的模式
4. 等待确认后再应用改动
5. 呈现优化结果，并附带自验证

我使用逐字复述和谨慎的输出排序来将 skill 锚定在参考模式上，防止模型自行发明技术。

## 使用示例

优化一个子 agent：

```
使用你的 prompt engineer skill 优化以下 Claude Code 子 agent 的系统 prompt：agents/developer.md
```

优化多 prompt 工作流：

```
参考 @skills/planner/scripts/planner.py，识别其中所有 prompt，
理解它们之间的交互关系，然后使用你的 prompt engineer skill 对每个 prompt 进行优化。
```

## 输出示例

每条改动建议包含范围、问题描述、所用技术、修改前后对比，以及改动理由。单次调用可能产出多条建议：

```
  +==============================================================================+
  |  CHANGE 1: Add STOP gate to Step 1 (Exploration)                             |
  +==============================================================================+
  |                                                                              |
  |  SCOPE                                                                       |
  |  -----                                                                       |
  |  Prompt:      analyze.py step 1                                              |
  |  Section:     Lines 41-49 (precondition check)                               |
  |  Downstream:  All subsequent steps depend on exploration results             |
  |                                                                              |
  +------------------------------------------------------------------------------+
  |                                                                              |
  |  PROBLEM                                                                     |
  |  -------                                                                     |
  |  Issue:    Hedging language allows model to skip precondition                |
  |                                                                              |
  |  Evidence: "PRECONDITION: You should have already delegated..."              |
  |            "If you have not, STOP and do that first"                         |
  |                                                                              |
  |  Runtime:  Model proceeds to "process exploration results" without having    |
  |            any results, produces empty/fabricated structure analysis         |
  |                                                                              |
  +------------------------------------------------------------------------------+
  |                                                                              |
  |  TECHNIQUE                                                                   |
  |  ---------                                                                   |
  |  Apply:    STOP Escalation Pattern (single-turn ref)                         |
  |                                                                              |
  |  Trigger:  "For behaviors you need to interrupt, not just discourage"        |
  |  Effect:   "Creates metacognitive checkpoint--the model must pause and       |
  |             re-evaluate before proceeding"                                   |
  |  Stacks:   Affirmative Directives                                            |
  |                                                                              |
  +------------------------------------------------------------------------------+
  |                                                                              |
  |  BEFORE                                                                      |
  |  ------                                                                      |
  |  +----------------------------------------------------------------------+    |
  |  | "PRECONDITION: You should have already delegated to the Explore      |    |
  |  |  sub-agent.",                                                        |    |
  |  | "If you have not, STOP and do that first:",                          |    |
  |  +----------------------------------------------------------------------+    |
  |                                                                              |
  |                                    |                                         |
  |                                    v                                         |
  |                                                                              |
  |  AFTER                                                                       |
  |  -----                                                                       |
  |  +----------------------------------------------------------------------+    |
  |  | "STOP. Before proceeding, verify you have Explore agent results.",   |    |
  |  | "",                                                                  |    |
  |  | "If your --thoughts do NOT contain Explore agent output, you MUST:", |    |
  |  | "  1. Use Task tool with subagent_type='Explore'                     |    |
  |  | "  2. Prompt: 'Explore this repository. Report directory structure,  |    |
  |  | "     tech stack, entry points, main components, observed patterns.' |    |
  |  | "  3. WAIT for results before invoking this step again               |    |
  |  | "",                                                                  |    |
  |  | "Only proceed below if you have concrete Explore output to process." |    |
  |  +----------------------------------------------------------------------+    |
  |                                                                              |
  +------------------------------------------------------------------------------+
  |                                                                              |
  |  WHY THIS IMPROVES QUALITY                                                   |
  |  -------------------------                                                   |
  |  Transforms soft precondition into hard gate. Model must explicitly verify   |
  |  it has Explore results before processing, preventing fabricated analysis.   |
  |                                                                              |
  +==============================================================================+

  ... 以及更多改动


  ---
  兼容性检查：
  - STOP Escalation + Affirmative Directives：兼容（STOP 用于中断特定行为）
  - History Accumulation + Completeness Checkpoint Tags：协同增强（两者都强制状态追踪）
  - Quote Extraction + Chain-of-Verification：互补（两者均能防止幻觉）
  - Progressive depth + Pre-Work Context Analysis：顺序关系（规划为更深层执行铺路）

  反模式验证：
  - 无模糊措辞螺旋（将「should have」替换为「STOP. Verify...」）
  - 无「万事皆关键」（CRITICAL 仅用于状态要求）
  - 使用肯定式指令（将否定改为肯定）
  - 无隐式类别陷阱（提供了明确的检查清单）

  ---
  这个方案看起来合理吗？确认后我就应用这些改动。
```

## 注意事项

当你让一个 LLM「找出问题和优化机会」时，它会找出问题。这正是你要求它做的。其中某些可能并非真正的问题。

我建议对难度较高的 prompt 多次调用本 skill，但要懂得适可而止。边际收益递减是真实存在的。
