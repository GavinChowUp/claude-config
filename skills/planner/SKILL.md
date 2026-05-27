---
name: planner
description: 复杂任务的交互式规划与执行。当用户要求使用 planner 时立即调用。
---

## 激活

当此 skill 激活时,立即调用对应脚本。脚本本身即工作流。

| 模式      | 意图                               | 命令                                                                                                          |
| --------- | ---------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| planning  | "plan"、"design"、"architect"      | `<invoke working-dir=".claude/skills/scripts" cmd="python3 -m skills.planner.orchestrator.planner --step 1" />`  |
| execution | "execute"、"implement"、"run plan" | `<invoke working-dir=".claude/skills/scripts" cmd="python3 -m skills.planner.orchestrator.executor --step 1" />` |
