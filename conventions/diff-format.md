# 计划代码变更的统一 Diff 格式

本文档是实现计划中代码变更的权威规范。

## 目的

统一 diff 格式在单一结构中同时编码**位置**和**内容**。这消除了在注释中使用位置指令（如「在第 42 行插入」）的需要，即使行号发生漂移，也能提供可靠的锚点。

## 结构解析

```diff
--- a/path/to/file.py
+++ b/path/to/file.py
@@ -123,6 +123,15 @@ def existing_function(ctx):
    # Context lines (unchanged) serve as location anchors
    existing_code()

+   # NEW: Comments explain WHY - transcribed verbatim by Developer
+   # Guard against race condition when messages arrive out-of-order
+   new_code()

    # More context to anchor the insertion point
    more_existing_code()
```

## 组成部分

| 组成部分                                   | 权威性                    | 用途                                                        |
| ------------------------------------------ | ------------------------- | ----------------------------------------------------------- |
| 文件路径 (`--- a/path/to/file.py`)        | **AUTHORITATIVE**         | 精确的目标文件                                              |
| 行号 (`@@ -123,6 +123,15 @@`)      | **APPROXIMATE**           | 随前面的里程碑修改文件后可能发生漂移                        |
| 函数上下文 (`@@ ... @@ def func():`) | **SCOPE HINT**            | 包含该变更的函数/方法                                       |
| 上下文行（未变更）                  | **AUTHORITATIVE ANCHORS** | Developer 通过匹配这些模式来定位插入点                      |
| `+` 行                                  | **NEW CODE**              | 要添加的代码，包含 WHY 注释                                 |
| `-` 行                                  | **REMOVED CODE**          | 要删除的代码                                                |

## 双层定位策略

代码变更使用两种互补层来定位：

1. **散文范围提示**（可选）：用自然语言描述概念层面的位置
2. **带上下文的 diff**：通过上下文行匹配精确定位插入点

### 第一层：散文范围提示

对于复杂变更，在 diff 块前添加散文描述：

````markdown
Add validation after input sanitization in `UserService.validate()`:

```diff
@@ -123,6 +123,15 @@ def validate(self, user):
     sanitized = sanitize(user.input)

+    # Validate format before proceeding
+    if not is_valid_format(sanitized):
+        raise ValidationError("Invalid format")
+

     return process(sanitized)
`` `
```
````

散文告诉 Developer **概念层面在哪里**（哪个方法、前置操作是什么）。diff 告诉 Developer **精确在哪里**（要匹配的上下文行）。

**何时使用散文提示：**

- 大文件变更（>300 行）
- 一个里程碑中对同一文件进行多处修改
- 复杂嵌套结构中仅凭函数上下文不足以定位
- 周围代码逻辑对理解插入位置有帮助时

**何时散文可省略：**

- 小文件且结构清晰
- 单处变更且上下文行唯一
- `@@` 行的函数上下文已提供足够范围信息

### 第二层：@@ 行中的函数上下文

`@@` 行可在行号后附上函数/方法上下文：

```diff
@@ -123,6 +123,15 @@ def validate(self, user):
```

这遵循标准统一 diff 格式（git 自动生成）。它告诉 Developer 变更所在函数，即使行号发生漂移也便于导航。

## 为什么上下文行很重要

当一个计划包含多个修改同一文件的里程碑时，前面的里程碑会导致行号偏移。里程碑 3 中的 `@@ -123` 在里程碑 1 和 2 执行后可能已不准确。

**上下文行解决了这个问题**：Developer 在实际文件中搜索未更改的上下文模式。这些模式是稳定的锚点，能在行号漂移后依然有效。

变更前后各保留 2-3 行上下文，以确保可靠匹配。

## 注释放置

`+` 行中的注释解释 **WHY**，不解释 WHAT。这些注释：

- 由 Developer 原文转录
- 来源于规划上下文（决策日志、被否决的替代方案）
- 使用具体术语，不引入隐藏基线
- 必须通过时间污染审查（参见 `.claude/conventions/temporal.md`）

**重要**：规划阶段编写的注释常含有时间污染——变更相对语言、基线引用或位置指令。@agent-technical-writer 会在 @agent-developer 转录前审查并修正。

<example type="CORRECT" category="why_comment">
```diff
+   # Polling chosen over webhooks: 30% webhook delivery failures in third-party API
+   # WebSocket rejected to preserve stateless architecture
+   updates = poll_api(interval=30)
```
解释了为什么选择这种方案。
</example>

<example type="INCORRECT" category="what_comment">
```diff
+   # Poll the API every 30 seconds
+   updates = poll_api(interval=30)
```
重述了代码的 WHAT——与代码本身冗余。
</example>

<example type="INCORRECT" category="hidden_baseline">
```diff
+   # Generous timeout for slow networks
+   REQUEST_TIMEOUT = 60
```
「宽裕」与什么相比？隐藏基线没有提供可操作的信息。
</example>

<example type="CORRECT" category="concrete_justification">
```diff
+   # 60s accommodates 95th percentile upstream response times
+   REQUEST_TIMEOUT = 60
```
具体论证，解释了为什么选择这个特定值。
</example>

## 位置指令：禁止使用

diff 结构已处理位置信息。注释中的位置指令是冗余且容易出错的。

<example type="INCORRECT" category="location_directive">
```python
# Insert this BEFORE the retry loop (line 716)
# Timestamp guard: prevent older data from overwriting newer
get_ctx, get_cancel = context.with_timeout(ctx, 500)
```
位置指令泄露到注释中——行号会过时。
</example>

<example type="CORRECT" category="location_directive">
```diff
@@ -714,6 +714,10 @@ def put(self, ctx, tags):
    for tag in tags:
        subject = tag.subject

-       # Timestamp guard: prevent older data from overwriting newer
-       # due to network delays, retries, or concurrent writes
-       get_ctx, get_cancel = context.with_timeout(ctx, 500)

        # Retry loop for Put operations
        for attempt in range(max_retries):

```
上下文行（`for tag in tags`、`# Retry loop`）是稳定锚点，在行号漂移后依然有效。
</example>

## 何时使用 Diff 格式

<diff_format_decision>

| 代码特征                                | 用 Diff？ | 边界测试                                     |
| --------------------------------------- | --------- | -------------------------------------------- |
| 条件语句、循环、错误处理、状态机        | YES       | 有分支逻辑                                   |
| 同一文件多处插入                        | YES       | 超过 1 个变更位置                            |
| 删除或替换                              | YES       | 移除/修改现有代码                            |
| 纯赋值/返回（CRUD、getter）             | NO        | 单条语句，无分支                             |
| 来自模板的样板代码                      | NO        | Developer 可从模式名称生成                   |

边界测试：「Developer 是否需要看到精确的位置和上下文才能正确实现？」

- YES → diff 格式
- NO（仅凭描述即可实现）→ 散文足够

</diff_format_decision>

## 验证清单

在计划中确定代码变更之前：

- [ ] 文件路径精确（不是「auth files」，而是 `src/auth/handler.py`）
- [ ] 上下文行在目标文件中存在（验证模式与实际代码匹配）
- [ ] 注释解释 WHY，不解释 WHAT
- [ ] 注释中无位置指令
- [ ] 无隐藏基线（测试：「[形容词]与什么相比？」）
- [ ] 2-3 行上下文以确保可靠锚定
```
