---
name: decision-critic
description: 当用户需要对 decision 和 reasoning 进行压力测试时，立即通过 python 脚本调用。不要先分析——脚本负责编排整个评审工作流。
---

# Decision Critic

此 skill 激活时，立即调用脚本。脚本本身就是工作流。

## 调用方式

<invoke working-dir=".claude/skills/scripts" cmd="python3 -m skills.decision_critic.decision_critic --step 1 --decision '<decision text>'" />

| 参数            | 是否必填   | 说明                                   |
| --------------- | -------- | --------------------------------------- |
| `--step`        | 是       | 当前步骤(1-7)                          |
| `--decision`    | 步骤 1   | 待评审的决策陈述                        |

不要先分析或评审。运行脚本并跟随其输出执行。
