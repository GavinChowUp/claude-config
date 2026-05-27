---
name: doc-sync
description: 跨 repo 同步文档。当用户要求同步文档时使用。Synchronizes docs across a repository. Use when user asks to sync docs.
---

# Doc Sync

维护整个 repo 中的 CLAUDE.md 导航层级和 README.md 隐性知识文档。本 skill 自包含，直接执行所有文档工作。

## 文档规范

关于 CLAUDE.md 和 README.md 的权威格式规范：

<file working-dir=".claude" uri="conventions/documentation.md" />

`conventions/` 目录包含所有通用文档标准。

## 范围确定

首先确定范围：

| 用户请求                                            | 范围                                     |
| ------------------------------------------------------- | ----------------------------------------- |
| 「同步文档」/ 「更新文档」/ 未指定路径 | 整个仓库                           |
| 「同步 src/validator/ 中的文档」                           | 目录：src/validator/ 及其子目录 |
| 「为 parser.py 更新 CLAUDE.md」                        | 文件：单个文件的父目录      |

对于整个仓库范围，执行完整审计。对于更窄的范围，只在指定边界内操作。

## 工作流

### 阶段 1：发现

映射需要 CLAUDE.md 验证的目录：

```bash
# 查找所有目录（排除 .git、node_modules、__pycache__ 等）
find . -type d \( -name .git -o -name node_modules -o -name __pycache__ -o -name .venv -o -name target -o -name dist -o -name build \) -prune -o -type d -print
```

对范围内的每个目录，记录：

1. CLAUDE.md 是否存在？
2. 如果存在，是否有所需的表格式索引结构？
3. 哪些文件/子目录需要被索引？

### 阶段 2：审计

对每个目录，检查漂移和内容错位：

```
<audit_check dir="[path]">
CLAUDE.md exists: [YES/NO]
Has table-based index: [YES/NO]
Files in directory: [list]
Files in index: [list]
Missing from index: [list]
Stale in index (file deleted): [list]
Triggers are task-oriented: [YES/NO/PARTIAL]
Contains misplaced content: [YES/NO] (architecture/design docs that belong in README.md)
README.md exists: [YES/NO]
README.md warranted: [YES/NO] (invisible knowledge present?)
</audit_check>
```

### 阶段 3：内容迁移

**关键：** 如果 CLAUDE.md 包含不属于其中的内容，须迁移：

必须从 CLAUDE.md 移到 README.md 的内容：

- 架构说明或图表
- 设计决策文档
- 组件交互描述
- 带散文的概述章节（超过一句话）
- 不变量或规则文档
- 除简单触发器之外的任何「为什么」解释
- 关键不变量章节
- 依赖项章节（解释性的——索引可以注明依赖项存在）
- 约束章节
- 带散文的目的章节（超过一句话）
- 解释原理的任何项目列表

可以留在 CLAUDE.md 中的内容（操作性章节）：

- 本目录特有的构建命令
- 本目录特有的测试命令
- 重新生成/同步命令（如 protobuf 重新生成）
- 部署命令
- 其他可直接复制执行的操作性命令

**测试：** 问「这是在解释为什么，还是在说明如何做？」解释性内容（架构、决策、原理）进入 README.md。操作性内容（命令、流程）留在 CLAUDE.md。

迁移流程：

1. 识别 CLAUDE.md 中的错位内容
2. 创建或更新 README.md，写入架构内容
3. 将 CLAUDE.md 精简为纯索引格式
4. 将 README.md 添加到 CLAUDE.md 的索引表中

### 阶段 4：索引更新

对需要处理的每个目录：

**创建/更新 CLAUDE.md：**

1. 使用合适的模板（根目录或子目录）
2. 用所有文件和子目录填充表格
3. 「内容」列：写实际内容描述
4. 「何时阅读」列：写面向动作的触发器
5. 如果 README.md 存在，将其纳入文件表

**创建 README.md（当隐性知识存在时）：**

1. 验证隐性知识确实存在（语义触发，而非结构性）
2. 记录架构、设计决策、不变量、权衡
3. 应用内容测试：删除从代码中即可看到的内容
4. 尽可能简洁，同时捕获所有隐性知识
5. 必须自包含：不引用外部权威来源

### 阶段 5：验证

所有更新完成后，验证：

1. 范围内每个目录都有 CLAUDE.md
2. 所有 CLAUDE.md 使用表格式索引格式（纯导航）
3. 没有漂移（文件与索引条目一一对应）
4. CLAUDE.md 中没有错位内容（解释性散文已移到 README.md）
5. README.md 文件已在父目录的 CLAUDE.md 中索引
6. CLAUDE.md 只包含：一句话概述 + 表格索引 + 操作性章节
7. 凡有隐性知识处均有 README.md
8. README.md 是自包含的（无外部权威引用）

## 输出格式

```
## Doc Sync 报告

### 范围：[整个仓库 | 目录路径]

### 已完成的变更
- 创建：[新建的 CLAUDE.md 列表]
- 更新：[修改的 CLAUDE.md 列表]
- 迁移：[从 CLAUDE.md 移到 README.md 的内容列表]
- 创建：[新建的 README.md 列表]
- 标记：[需要人工决策的问题]

### 验证
- 已审计目录：[数量]
- CLAUDE.md 覆盖率：[数量]/[总计]（100%）
- CLAUDE.md 格式：[数量] 纯索引 / [数量] 需迁移
- 检测到漂移：[数量] 条已修复
- 内容迁移：[数量]（散文已移到 README.md）
- README.md 文件：[数量]（凡有隐性知识处）
- 自包含：[是/否]（无外部权威引用）
```

## 排除项

不为以下目录创建 CLAUDE.md：

- 生成文件目录（dist/、build/、编译输出）
- 第三方依赖（node_modules/、vendor/、third_party/）
- Git 内部目录（.git/）
- IDE/编辑器配置（.idea/、.vscode/，除非是项目特有设置）
- **存根目录**（只含 `.gitkeep` 或无代码文件）——在添加代码之前不需要 CLAUDE.md

不索引（CLAUDE.md 中跳过这些文件）：

- 生成文件（_.generated._、编译输出）
- 第三方依赖文件

需索引：

- 影响开发的隐藏配置文件（.eslintrc、.env.example、.gitignore）
- 测试文件和测试目录
- 文档文件（包括 README.md）

## 反模式

### 索引反模式

**过于宽泛（什么都能匹配）：**

```markdown
| `config/` | 配置 | 处理配置时 |
```

**只有内容描述，没有触发器：**

```markdown
| `cache.rs` | 包含 LRU 缓存实现 | - |
```

**缺少动作动词：**

```markdown
| `parser.py` | 输入解析 | 输入解析和格式处理 |
```

### 正确示例

```markdown
| `cache.rs` | O(1) get/set 的 LRU 缓存 | 实现缓存、调试缓存未命中、调优驱逐策略 |
| `config/` | YAML 配置解析、环境变量覆盖 | 添加配置选项、修改默认值、调试配置加载 |
```

## 不适用场景

- 单个文件的文档（行内注释、文档字符串）——直接处理
- 代码注释——直接处理
- 函数/模块 docstring——直接处理
- 本 skill 专门用于 CLAUDE.md/README.md 同步

## 参考

更多触发模式示例，参见 `references/trigger-patterns.md`。
