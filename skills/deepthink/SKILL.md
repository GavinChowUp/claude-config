---
name: deepthink
description: 当用户请求对开放性分析问题进行结构化 reasoning 时，立即通过 python 脚本调用。不要先探索——脚本负责编排整个思考工作流。
---

# DeepThink

此 skill 激活时，立即调用脚本。脚本本身就是工作流。

调用方式:

<invoke working-dir=".claude/skills/scripts" cmd="python3 -m skills.deepthink.think --step 1" />

不要先探索或分析。运行脚本并跟随其输出执行。
