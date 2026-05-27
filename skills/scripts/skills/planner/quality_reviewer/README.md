# QR（Quality Review）

## 概述

Quality Review 模块执行基于严重性阻塞阈值的验证。每个模块在指定的工作流 gate 处验证计划或实现的特定方面。

## 模块

**plan_completeness.py**：验证计划结构、milestone 定义与验收标准的完整性。

**plan_code.py**：审查代码 diff 的正确性、边界情况与实现质量。

**plan_docs.py**：验证文档的完整性、清晰度及与实现的一致性。

**post_impl_code.py**：根据计划规范对实现后代码进行验证。

**post_impl_doc.py**：审查实现后文档的准确性与完整性。

**reconciliation.py**：验证计划与实现是否匹配，确保所有 milestone 均已交付。

## QA 状态追踪集成

QR gate 现已集成 QA 状态追踪，用于结构化验证。QA 拆解将「计划与求解」方法论应用于质量验证，将整体性审查拆分为可并行的清单条目。

### 哲学：最小状态、哑主 agent、即时 prompt

**最小状态文件**：只存储权威数据——条目状态（TODO/PASS/FAIL）。子 agent 在需要时按需计算状态概览。不存储派生值。

**哑主 agent**：主 agent 负责派发和路由。它只需要一位信息：PASS 还是 FAIL。LLM 自然地读取响应并按指令行事。主 agent 中无需任何解析逻辑。

**即时 prompt**：执行子 agent（修复者）调用脚本并获取关于失败内容的 prompt。主 agent 从不看到这些细节。细节仅在需要时、仅注入给需要的 agent。

### 为何主 agent 不解析

LLM 自然地读取子 agent 响应并按指令行事。无需 JSON 解析，无需状态提取逻辑。子 agent 返回嵌入指令的文本响应，如「PASS: 继续下一阶段」或「FAIL: 使用条目 [...] 调用修复者」。主 agent 读取并遵循。

这消除了一整类 bug：解析错误、schema 不匹配、JSON 转义问题。LLM 的自然语言理解处理所有响应解释。

### 为何状态概览只对子 agent 可见

执行者需要看到「5 条条目：3 PASS，2 FAIL」才能决定修复什么。主 agent 不需要。主 agent 只需知道：验证通过还是失败？

状态概览由执行者在运行时按需计算。不存入 qr-{phase}.json，因为它是派生数据。存储它会违反最小状态原则，并产生一致性风险（如果计数与条目不匹配怎么办？）。

### 响应格式

**DECOMPOSE 模式**：返回 QA 条目 ID 及验证指令。

```
DECOMPOSE COMPLETE

Items created: 7
- plan-001: Verify milestone definitions (scope: *)
- plan-002: Check acceptance criteria (scope: milestone:M1)
- plan-003: Validate diff syntax (scope: file:planner/qa/verify.py)
...

NEXT: Invoke verifiers for each item.
```

**VERIFY 模式**：返回 PASS/FAIL 裁定。

```
VERIFICATION COMPLETE

Status: PASS
Items: 7 total, 7 PASS, 0 FAIL

NEXT: Continue to next phase.
```

或

```
VERIFICATION COMPLETE

Status: FAIL
Items: 7 total, 5 PASS, 2 FAIL
Failed items:
- plan-002: Acceptance criteria missing for M1
- plan-005: Diff has merge conflict markers

NEXT: Invoke fixer with failed items.
```

**FIX_GUIDANCE 模式**：为失败条目返回具体修复指令。

```
FIX GUIDANCE

Item plan-002: Acceptance criteria missing for M1
Scope: milestone:M1
Fix: Add acceptance criteria to milestone M1 definition. Include:
  - Success conditions
  - Verification steps
  - Exit criteria

Item plan-005: Diff has merge conflict markers
Scope: file:planner/qa/verify.py:45-52
Fix: Remove conflict markers (<<<<<<, ======, >>>>>>) and resolve merge conflicts.
```

### 三种模式

**DECOMPOSE**：将制品拆分为可验证的 QA 条目。子 agent 读取制品，识别质量维度，输出含 id/scope/check/status/finding 字段的条目。条目初始状态为 TODO。

**VERIFY**：为每个条目执行验证。宏条目（scope=`*`）顺序运行，微条目（scope=具体路径）并行运行。每个验证者将条目状态更新为 PASS/FAIL，并附上 finding 说明。

**FIX_GUIDANCE**：为失败条目生成具体修复指令。子 agent 读取失败条目，为每个失败生成可操作的指令。

### 状态文件 Schema（qr-{phase}.json）

```json
{
  "schema_version": "1.0",
  "phase": "plan-structure",
  "items": [
    {
      "id": "plan-001",
      "scope": "*",
      "check": "Verify milestone definitions are complete with acceptance criteria",
      "status": "PASS",
      "finding": null
    },
    {
      "id": "plan-002",
      "scope": "milestone:M1",
      "check": "Validate acceptance criteria for milestone M1",
      "status": "FAIL",
      "finding": "Acceptance criteria missing. Need success conditions and verification steps."
    },
    {
      "id": "plan-003",
      "scope": "file:planner/qa/verify.py",
      "check": "Check diff syntax and formatting",
      "status": "TODO",
      "finding": null
    }
  ]
}
```

**schema_version**：qr-{phase}.json 格式的版本标识符（当前为 "1.0"）。

**phase**：验证阶段——取值之一：`plan-structure`、`plan-code`、`plan-docs`、`impl-code`、`impl-docs`。

**items**：QA 条目数组，每条目恰好 5 个字段：

- **id**：并行派发的关联键（格式：`{phase}-{seq:03d}`）
- **scope**：内容目标，也是并行化提示（`*` 为宏，具体路径为微）
- **check**：自由格式的验证指令
- **status**：TODO/PASS/FAIL 之一
- **finding**：非 PASS 时的说明（TODO/PASS 时为 null）

### 主 agent 流程（哑路由器）

```
用户请求
     |
     v
步骤 1: plan-init
创建状态目录
     |
     v
步骤 2: plan-structure-execute
主 agent 派发 planner 子 agent
     |
     v
步骤 3: plan-structure-qr
主 agent 调用 QA decompose
     |
     v
Decompose 子 agent 返回：「Items created: 7」
     |
     v
主 agent 读取响应，看到「NEXT: Invoke verifiers」
     |
     v
主 agent 调用 verify（宏条目顺序，微条目并行）
     |
     v
Verify 子 agent 返回：「Status: FAIL, 2 failed items」
     |
     v
步骤 4: plan-structure-qr-gate
主 agent 读取响应，看到「NEXT: Invoke fixer」
     |
     v
步骤 2（带 --qr-fail）: plan-structure-execute
主 agent 调用修复者
     |
     v
修复者返回：「Fixes applied」
     |
     v
主 agent 回到步骤 3: plan-structure-qr
```

无状态检查，无 JSON 解析。主 agent 读取文本并遵循指令。

### 执行者流程（即时 prompt）

```
执行者被主 agent 调用
     |
     v
从 STATE_DIR 读取 qr-{phase}.json
     |
     v
计算状态概览（3 PASS，2 FAIL）
     |
     v
生成含失败详情的 prompt：
  - 「Item plan-002 failed: Acceptance criteria missing」
  - 「Item plan-005 failed: Diff has conflict markers」
     |
     v
执行修复
     |
     v
用新状态更新 qr-{phase}.json
     |
     v
返回响应：「Status: PASS, all items fixed」
```

状态概览按需计算，不存入 qr-{phase}.json，主 agent 从不看到。

### 为何用 JSON

**一致性**：JSON 是通用格式。每种语言、每种工具都支持。YAML 需要 pyyaml 依赖，且有缩进陷阱。

**避免 pyyaml 依赖**：少一个需要安装的包，少一个版本冲突风险。

**简洁性**：JSON schema 无歧义。YAML 对同一结构有多种语法（流式 vs 块状、带引号 vs 不带引号）。

**工具支持**：每个编辑器都内置 JSON 验证。JSON Schema 验证器随处可用。

**LLM 友好**：现代 LLM 原生处理 JSON。ChatML 和 Claude 都有 JSON 模式。简单结构如 QA 条目不存在转义问题。

## QR 迭代阻塞

严重性阈值随迭代深度变化，以防止无限重试循环：

| 迭代次数  | 阻塞严重性      | 理由                                |
| --------- | --------------------- | ---------------------------------------- |
| 1–2       | 全部（MUST/SHOULD/MAY） | 失败率高，强制立即修复 |
| 3–4       | MUST/SHOULD           | 处理细微问题                   |
| 5+        | 仅 MUST             | 防止无限重试循环             |

## LoopState 追踪

QR gate 使用 LoopState 枚举追踪迭代进度：

- **INITIAL**：首次审查尝试
- **RETRY**：修复上一次迭代的问题
- **COMPLETE**：通过审查

状态转换：

```
INITIAL -> (QRStatus.PASS) -> COMPLETE [终止]
INITIAL -> (QRStatus.NEEDS_CHANGES) -> RETRY -> (iteration++) -> RETRY -> ...
```

## 与 QA 工作流的集成

QR gate 在执行审查前调用 QA 拆解：

1. 触发 QR gate（如 plan_completeness）
2. 调用 qa/decompose.py 生成验证条目
3. 派发验证者（微条目并行，宏条目顺序）
4. 将结果汇总到 qr-{phase}.json
5. 根据汇总结果路由：
   - PASS：继续下一工作流步骤
   - FAIL：调用修复者，回到验证

此集成提供了结构化、可并行的验证，具有显式失败追踪和自动重试逻辑。
