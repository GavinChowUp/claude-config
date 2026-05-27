---
name: arxiv-to-md
description: 将 arXiv 论文转换为 LLM 可消费的 Markdown。当用户提供 arXiv ID 或 URL，或需要将 PDF 文件夹中的学术论文同步到 Markdown 目标时调用。Invoke when user provides an arXiv ID or URL, or when syncing academic papers from a PDF folder to a markdown destination.
---

# arXiv 转 Markdown

将 arXiv 论文（TeX 源码）转换为干净的 Markdown，供 LLM 消费。

## 调用

<invoke working-dir=".claude/skills/scripts" cmd="python3 -m skills.arxiv_to_md.main --step 1" />

不要先探索或分析。直接运行脚本并遵照其输出操作。
