---
name: incoherence
description: 检测并解决文档、代码、规范与实现之间的不一致。Detect and resolve incoherence in documentation, code, specs vs implementation.
---

# 一致性检测器

本 skill 激活时，立即调用脚本。脚本就是工作流。

## 调用

<invoke working-dir=".claude/skills/scripts" cmd="python3 -m skills.incoherence.incoherence --step-number 1 --thoughts '<context>'" />

| 参数            | 是否必填 | 描述                               |
| --------------- | -------- | ----------------------------------------- |
| `--step-number` | 是      | 当前步骤（从 1 开始）                |
| `--thoughts`    | 是      | 所有前置步骤积累的状态 |

不要先探索或检测。直接运行脚本并遵照其输出操作。

## 工作流阶段

1. **检测（步骤 1-12）**：扫查代码库、探索维度、验证候选问题
2. **解决（步骤 13-15）**：通过 AskUserQuestion 呈现问题，收集用户决策
3. **应用（步骤 16-21）**：应用解决方案，呈现最终报告

解决阶段是交互式的——用户在线回答结构化问题。不需要手动编辑文件。
