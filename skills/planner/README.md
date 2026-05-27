# Planner

LLM 生成的计划存在漏洞——我见过缺失错误处理、验收标准模糊、没人能真正落地的规范。我带着两个工作流构建了这个 skill:规划和执行,中间用质量门把这些问题挡在早期。

**权威规范**:完整的设计理由、不变量和状态文件 schema 见 INTENT.md。本 README 提供操作层概述;架构决策以 INTENT.md 为准。

## 规划工作流

```
  Planning ----+
      |        |
      v        |
     QR -------+  [fail: 重新规划]
      |
      v
     TW -------+
      |        |
      v        |
   QR-Docs ----+  [fail: 重新运行 TW]
      |
      v
   APPROVED
```

| 步骤                    | 动作                                                                       |
| ----------------------- | -------------------------------------------------------------------------- |
| 上下文 & 范围           | 确认路径、定义范围、识别方案、列出约束                                     |
| 决策 & 架构             | 评估方案、带推理链选定、绘图、拆分为里程碑                                 |
| 精化                    | 记录风险、添加不确定性标记、明确路径和标准                                 |
| 最终验证                | 验证完整性、检查规范、写入文件                                             |
| QR-完整性               | 验证决策日志完整、确认策略默认值、检查计划结构                             |
| QR-代码                 | 读代码库、验证 diff 上下文、对提议代码应用 RULE 0/1/2                      |
| Technical Writer        | 清理时态注释、添加 WHY 注释、丰富设计理由                                  |
| QR-文档                 | 验证无时态污染、注释说明「为什么」而非「做什么」                           |

为什么要有这些反馈循环?QR-完整性和 QR-代码在 TW 之前运行,提前捕获结构性问题。QR-文档在 TW 之后运行,验证文档质量。文档问题只重启 TW;结构问题重启整个规划。循环持续到两者都通过为止。

## 执行工作流

```
  Plan --> Milestones --> QR --> Docs --> Retrospective
               ^          |
               +- [fail] -+

  * 恢复部分工作时,Milestones 之前会先执行对账阶段
```

规划完成并清空上下文(`/clear`)后,开始执行:

| 步骤                   | 目的                                                            |
| ---------------------- | --------------------------------------------------------------- |
| 执行规划               | 分析计划、检测对账信号、输出策略                                |
| 对账                   | (条件触发)验证现有代码是否符合计划                              |
| 里程碑执行             | 委派给 agent、运行测试;重复直到全部完成                        |
| 实现后 QR              | 对已实现代码进行质量复审                                        |
| 问题解决               | (条件触发)呈现问题、收集决策、委派修复                          |
| 文档                   | Technical Writer 更新 CLAUDE.md/README.md                       |
| 回顾                   | 呈现执行摘要                                                    |

我把协调器设计为从不直接写代码——它始终委派给 developer。协调与实现分离能产生更干净的结果。协调器会:

- 每个里程碑最多并行调度 4 个 developer
- 所有里程碑完成后才运行质量复审
- 循环推进问题解决直到 QR 通过
- 只有 QR 通过后才调用 technical writer

**对账**处理恢复场景。当用户请求包含「已实现」「恢复」「部分完成」等信号时,工作流会先验证现有代码是否满足计划要求,再执行剩余里程碑。在未经验证的代码之上继续开发意味着返工。

**问题解决**逐条呈现 QR 发现的问题,提供选项(修复/跳过/替代方案)。修复委派给 developer 或 technical writer,然后再次运行 QR。此循环重复直到 QR 通过。

## 隐性知识

### 为什么移除 session.yaml

最初设计包含 session.yaml 用于跨调用追踪工作流状态。后来移除,因为 context.json 已经捕获了任务和架构决策——这才是子 agent 真正需要的关键状态。会话级追踪(当前步骤、时间戳)属于编排器的上下文窗口,不应持久化。单独引入一个文件只增加了冗余而没有带来价值。

### 为什么采用 6 字段决策 schema

早期设计每条决策有 11 个字段(id、question、status、raised_at、decided_at、decided_by、answer、rationale、options、blocking、superseded_by)。精简为 6 个字段(id、question、status、decided_by、answer、rationale),原因:

- raised_at/decided_at:时间戳增加噪音,不改善决策推理
- options:在 EXPLORING 阶段更适合放在 findings.json 中捕获
- blocking:隐含于 status=READY 时编排器等待用户输入
- superseded_by:可通过 status=SUPERSEDED + 同问题的新决策来追踪

schema 越简单,LLM 在写决策时出错的概率就越低。

### 为什么用每阶段独立的 qr-<phase>.json 而非单个 qa.json

独立的 qr-<phase>.json 文件(qr-plan-structure.json、qr-plan-code.json、qr-plan-docs.json、qr-impl-code.json、qr-impl-docs.json)防止跨阶段污染。如果使用单个 qa.json:

- 计划 QR 条目和实现 QR 条目混在一起(修复人员会混淆)
- 验证范围不清晰(这个条目在检查哪个阶段?)
- 无法隔离每个阶段的 QR 结果(计划 QR 应独立于实现 QR)

每阶段独立文件允许具有清晰边界的独立验证循环。每个文件在其所在阶段通过质量门后删除。

## 计划 Schema

plan.json 的关键字段:

- milestones[].documentation.function_blocks[](Tier 2 函数级设计理由)
- milestones[].documentation.inline_comments[](Tier 1 WHY 注释)
- readme_entries[](跨里程碑的架构说明)
