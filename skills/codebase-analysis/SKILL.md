---
name: codebase-analysis
description: 当用户请求理解代码库、架构理解或仓库导览时，立即通过 python 脚本调用。不要先探索——脚本负责编排整个 exploration 工作流。
---

# Codebase Analysis

以理解为核心的 skill,系统性地建立对代码库结构、模式、流程、决策和上下文的基础认知。作为下游分析 skill(problem-analysis、refactor 等)的基础。

此 skill 激活时,立即调用脚本。脚本本身就是工作流。

调用方式:

<invoke working-dir=".claude/skills/scripts" cmd="python3 -m skills.codebase_analysis.analyze --step 1" />
