# 参考资料选择指南

这些参考文档包含基于研究的 prompt 工程技术，按问题类型组织。每份文件将多篇论文的研究成果综合提炼为可直接使用的指导建议。

## 决策树

```
你的主要问题是什么？
|
+-> 输入问题（上下文过长/有噪声/信息缺失）？
|   是 -> context/reframing.md 或 context/augmentation.md
|
+-> 输出问题：
    |
    +-> 模型无法推理完成问题          -> reasoning/*.md
    +-> 模型能推理但答案错误           -> correctness/*.md
    +-> 输出过于冗长或成本过高         -> efficiency.md
    +-> 需要特定格式                   -> structure.md
```

## 导航反模式

避免直接阅读以下内容：

- `papers/**/*.md` - 原始论文摘要，粒度过细，不适合优化工作流
- `papers/**/*.yaml` - 论文元数据，不可直接使用
- `papers/**/*.pdf` - 原始论文，无法直接消费

这里的参考文件已将这些论文综合整理为可操作的技术指南。

## 在 Prompt 优化工作流中的使用方式

optimize.py 脚本会根据诊断出的问题选择对应的参考资料：

1. **分流（Triage）**：确定范围（single-prompt、ecosystem、greenfield、problem）
2. **评估/诊断（Assess/Diagnose）**：识别具体的失败模式
3. **规划/设计（Plan/Design）**：根据失败模式读取相关参考：
   - 推理失败 -> reasoning/\*.md
   - 一致性失败 -> correctness/\*.md
   - 上下文问题 -> context/\*.md
   - 冗长问题 -> efficiency.md
   - 格式问题 -> structure.md
