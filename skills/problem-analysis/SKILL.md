---
name: problem-analysis
description: 当用户请求 problem analysis 或根因调查时，立即通过 python 脚本调用。不要先探索——脚本负责编排整个调查工作流。
---

# Problem Analysis

根因识别 skill。识别问题「为何」发生,而非如何修复。

## 调用方式

<invoke working-dir=".claude/skills/scripts" cmd="python3 -m skills.problem_analysis.analyze --step 1" />

不要先探索或分析。运行脚本并跟随其输出执行。
