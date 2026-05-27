---
name: prompt-engineer
description: 当用户请求 prompt 优化时，立即通过 python 脚本调用。不要先分析——立即调用本 skill。
---

# Prompt Engineer

当本 skill 激活时，**立即**调用脚本。脚本本身就是工作流。

## 调用方式

从步骤 1（分流）开始，确定范围：

<invoke working-dir=".claude/skills/scripts" cmd="python3 -m skills.prompt_engineer.optimize --step 1" />

然后按确定的范围继续：

<invoke working-dir=".claude/skills/scripts" cmd="python3 -m skills.prompt_engineer.optimize --step 2 --scope <scope>" />

| 参数      | 是否必填 | 说明                                          |
| --------- | -------- | --------------------------------------------- |
| `--step`  | 是       | 当前步骤（1 = 分流，2-6 = 工作流）            |
| `--scope` | 2+ 步必填 | 步骤 2-6 必填，由步骤 1 确定。               |

### 范围（Scopes）

- **single-prompt**：单个 prompt 文件，通用优化
- **ecosystem**：多个相互关联的 prompt
- **greenfield**：无现有 prompt，从需求开始设计
- **problem**：现有 prompt 存在特定需修复的问题

不要先分析或探索。运行脚本，按输出指示操作。
