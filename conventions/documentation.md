# 文档规范

本文件是权威的文档规范。所有代码附属文档（CLAUDE.md、README.md）必须遵循以下原则。

## 核心原则

**自包含文档**：所有代码附属文档（CLAUDE.md、README.md）必须自包含。不得引用外部权威来源（doc/ 目录、wiki、外部文档）。如果知识存在于权威来源中，必须在本地做摘要。允许重复；维护成本是局部化的代价。

**CLAUDE.md = 纯索引**：CLAUDE.md 文件只是导航辅助工具，包含目录中**有什么**内容以及**何时**阅读每个文件。所有解释性内容（架构、决策、不变量）属于 README.md。

**README.md = 隐性知识**：README.md 文件记录从阅读源代码**无法获得**的知识。若某目录存在任何隐性知识，则必须有 README.md。

## CLAUDE.md 格式规范

### 索引格式

使用含 What 和 When 列的表格格式：

```markdown
## Files

| File        | What                           | When to read                              |
| ----------- | ------------------------------ | ----------------------------------------- |
| `cache.rs`  | LRU cache with O(1) operations | Implementing caching, debugging evictions |
| `errors.rs` | Error types and Result aliases | Adding error variants, handling failures  |

## Subdirectories

| Directory   | What                          | When to read                              |
| ----------- | ----------------------------- | ----------------------------------------- |
| `config/`   | Runtime configuration loading | Adding config options, modifying defaults |
| `handlers/` | HTTP request handlers         | Adding endpoints, modifying request flow  |
```

### 列说明

- **File/Directory**：名称加反引号：`cache.rs`、`config/`
- **What**：内容的事实描述（名词，非动作）
- **When to read**：以动作动词引导的任务触发条件（implementing, debugging, modifying, adding, understanding）
- 至少一列必须有内容；空单元格用 `-`

### 触发质量测试

给定任务「添加新的验证规则」，LLM 能否扫描「When to read」列并识别出正确文件？

### 生成代码与 vendor 代码

CLAUDE.md **必须**标记不应手动编辑的文件/目录：

| 目录           | 内容                              | 何时阅读            |
| -------------- | --------------------------------- | ------------------- |
| `proto/gen/`   | Generated from proto/. Run `make` | Never edit directly |
| `vendor/`      | Vendored deps, upstream: go.mod   | Never edit directly |
| `third_party/` | Copied from github.com/foo v1.2.3 | Never edit directly |

「When to read」列应说明这些文件不可编辑。在「What」列或专门的 Regenerate 节中注明重新生成命令。

这可防止 LLM 浪费精力分析或「改进」自动生成的代码，也防止手动编辑被覆写或引发合并冲突。

另见：conventions/code-quality/baseline.md「Generated and Vendored Code Awareness」。

### 根目录 vs 子目录 CLAUDE.md

**ROOT CLAUDE.md：**

```markdown
# [Project Name]

[一句话：这是什么]

## Files

| File | What | When to read |
| ---- | ---- | ------------ |

## Subdirectories

| Directory | What | When to read |
| --------- | ---- | ------------ |

## Build

[可直接粘贴的命令]

## Test

[可直接粘贴的命令]

## Development

[安装说明、环境要求、工作流注意事项]
```

**SUBDIRECTORY CLAUDE.md：**

```markdown
# [directory-name]/

## Files

| File | What | When to read |
| ---- | ---- | ------------ |

## Subdirectories

| Directory | What | When to read |
| --------- | ---- | ------------ |
```

**关键约束：** CLAUDE.md 是导航辅助工具，不是解释性文档。其内容包含：

- 文件/目录索引（必须）：含 What/When 列的表格格式
- 一句话概述（可选）：该目录是什么
- 操作节（可选）：Build、Test、Regenerate、Deploy 等该目录特定制品的命令

不包含：

- 架构解释（→ README.md）
- 设计决策或原理（→ README.md）
- 不变量或约束（→ README.md）
- 多段落散文（→ README.md）

操作节必须是可直接粘贴的命令，附最少上下文，不是关于构建方式的解释性散文。

## README.md 规范

### 创建标准（隐性知识测试）

当目录包含**任何**隐性知识时创建 README.md——即无法通过阅读代码获得的知识：

- 规划决策（实现期间决策日志中的内容）
- 业务背景（产品为何这样工作）
- 架构原理（为何采用这种结构）
- 取舍（牺牲了什么换取了什么）
- 不变量（必须保持但未在类型中体现的规则）
- 历史背景（为何不选用替代方案）
- 性能特性（非显而易见的效率属性）
- 多个组件通过非显然的契约交互
- 目录结构本身编码了领域知识
- 故障模式或边界情况无法从读取单个文件中看出
- 开发者必须遵守但编译器/linter 不强制的「规则」

**以上任意一条存在，则 README.md 为必须。** 触发条件是语义性的（隐性知识的存在），而非结构性的（文件数量、复杂度）。

**不要创建 README.md 的情况：**

- 目录纯粹是组织性的，其结构背后没有决策
- 所有知识均可从源代码中获得
- 只是在复述代码已经展示的内容

### 内容测试

对 README.md 中的每一句话问：「开发者能通过阅读源文件学到这一点吗？」

- 如果是：删除该句
- 如果否：保留

README.md 通过提供**隐性**知识来赚取其 token 预算：即代码背后的推理，而非对代码的描述。

### README.md 结构

```markdown
# [Component Name]

## Overview

[一段话：解决什么问题、高层次方案]

## Architecture

[子组件如何交互；数据流；核心抽象]

## Design Decisions

[取舍及原因；考虑过的替代方案]

## Invariants

[必须维护的规则；代码未强制的约束]
```

## 架构文档

对于跨越多个目录的横切关注点和系统级关系，创建专门的架构文档。

### 结构

```markdown
# Architecture: [System/Feature Name]

## Overview

[一段话：问题与高层次方案]

## Components

[每个组件的单一职责及边界]

## Data Flow

[关键路径——复杂流程优先使用图表]

## Design Decisions

[核心取舍与原理]

## Boundaries

[该系统不做什么；职责边界在哪里]
```

### 质量标准

组件说明必须阐释关系，而非仅列出职责。

错误——只列清单，没有关系：

```markdown
## Components

- UserService: Handles user operations
- AuthService: Handles authentication
- Database: Stores data
```

正确——阐释边界和数据流：

```markdown
## Components

- UserService: User CRUD only. Delegates auth to AuthService. Never queries auth
  state directly.
- AuthService: Token validation, session management. Stateless; all state in
  Redis.
- PostgreSQL: Source of truth for user data. AuthService has no direct access.

Flow: Request -> AuthService (validate) -> UserService (logic) -> Database
```

优先使用图表而非散文来描述关系。

## 代码内文档

代码级文档在最接近所描述代码的位置捕获知识。原则：知识应尽可能靠近其描述的代码。无法局部化的横切知识属于 README.md。

### 第一层：行内注释

位于语句或表达式上方，适用于选择不显而易见的地方。

记录*为什么*选择此方案，永远不记录代码*做了什么*。读者能看到代码做什么；他们看不到为何选择而非替代方案。

好：

```python
# Polling: 30% webhook delivery failures observed in production
result = poll_endpoint(url, interval=30)

# Mutex-free: single-writer guarantee from caller contract
counter.fetch_add(1, Ordering::Relaxed)
```

差：

```python
# Poll the endpoint
result = poll_endpoint(url, interval=30)

# Increment the counter
counter.fetch_add(1, Ordering::Relaxed)
```

若存在决策日志条目，引用它：`# DL-003: Polling over webhooks`

### 第二层：函数级说明块

位于非平凡函数顶部（签名之后、主体逻辑之前）。当函数有超过 3 个不同的转换步骤、协调多个子系统，或实现非显然算法时必须添加。

内容：函数做什么、如何做、在整体架构中的位置、解决什么问题。

```python
def reconcile_state(local, remote):
    # Reconciles local state against remote source of truth. Operates in
    # three phases:
    # 1. Diff local vs remote to find divergent keys
    # 2. For each divergence, apply conflict resolution (remote wins)
    # 3. Write merged state back to local store
    #
    # Called by the sync loop after each heartbeat. Remote state is
    # authoritative -- local is a cache that may lag behind.
    ...
```

CRUD 操作和标准模式中，代码本身已足够清晰的可跳过。

### 第三层：Docstring

**私有函数**：一行摘要 + 触发子句（何时调用）。

```python
def _normalize_key(k):
    """Strip whitespace and lowercase. Use before cache lookup."""
```

**公共函数**：摘要 + 触发子句 + 参数语义 + 示例。为 LLM 使用优化——触发子句和示例使工具选择更准确。

```python
def validate_config(path, strict=False):
    """Validate configuration file against schema.

    Use when loading user-provided config at startup or after hot-reload.
    In strict mode, unknown keys are errors; otherwise warnings.

    Args:
        path: Absolute path to YAML config file.
        strict: Treat unknown keys as errors.

    Returns:
        Validated Config object.

    Example:
        cfg = validate_config("/etc/app/config.yaml", strict=True)
    """
```

### 第四层：模块文档

文件顶部注释或模块 docstring。记录模块包含什么以及为何作为独立单元存在。

```python
"""Rate limiting using sliding window counters.

Provides per-client rate limiting for the API gateway. Sliding window
chosen over fixed window to prevent burst-at-boundary attacks (DL-007).
Token bucket rejected: memory overhead per client unacceptable at
projected scale (>100k concurrent clients).
"""
```

### 第五层：隐性知识放置

隐性知识是无法通过阅读代码获得的知识：业务背景、架构原理、取舍、约束、被否决的替代方案。

**位置层级**（选最近的可行位置）：

1. **行内注释**：知识适用于特定语句时
2. **函数级块**：知识适用于整个函数的方案或算法时
3. **模块 docstring**：知识适用于该模块为何存在或其整体设计时
4. **README.md**：知识横切多个文件/模块，或无法局部化到单个代码点时

不可接受的做法：隐性知识只存在于规划制品（决策日志、计划文档、对话历史）中，而未带入代码库。每个决策、约束和取舍必须落入代码或 README.md。

### 优先级顺序

决定记录什么时，按不确定性排序：

| 优先级   | 代码模式                     | 为什么问题             |
| -------- | ---------------------------- | ---------------------- |
| HIGH     | 多种有效方案                 | 为何选择这种？         |
| HIGH     | 阈值、超时、限制             | 为何是这些值？         |
| HIGH     | 错误处理路径                 | 恢复策略？             |
| HIGH     | 外部系统交互                 | 依赖了哪些假设？       |
| MEDIUM   | 非标准模式用法               | 为何偏离规范？         |
| MEDIUM   | 性能关键路径                 | 为何做此优化？         |
| LOW      | 样板/既定模式                | 非异常情况可跳过       |
| LOW      | 简单 CRUD 操作               | 非异常情况可跳过       |
