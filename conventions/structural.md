# 默认规范

这些规范在项目文档未另行指定时适用。

## 优先级层级

高层级覆盖低层级。审计时引用支持来源。

| 层级 | 来源            | 操作                                |
| ---- | --------------- | ----------------------------------- |
| 1    | user-specified  | 用户明确指令：直接应用              |
| 2    | doc-derived     | CLAUDE.md / 项目文档：直接应用      |
| 3    | default-derived | 本文档：直接应用                    |
| 4    | assumption      | 无支持来源：**与用户确认**          |

## 严重度级别

完整定义见 `severity.md`。

| 级别   | 含义                     |
| ------ | ------------------------ |
| MUST   | 遗漏后不可恢复           |
| SHOULD | 可维护性债务             |
| COULD  | 可自动修复，影响低       |

---

## 结构规范

<default-conventions domain="god-object">
**God Object**：公共方法 >15 OR 依赖 >10 OR 关注点混杂（网络 + UI + 数据）
Severity: SHOULD
</default-conventions>

<default-conventions domain="god-function">
**God Function**：代码行 >50 OR 多个抽象层 OR 嵌套层级 >3
Severity: SHOULD
Exception: 本质上顺序执行的算法或状态机
</default-conventions>

<default-conventions domain="duplicate-logic">
**Duplicate Logic**：复制粘贴块、重复错误处理、并行的几乎相同函数
Severity: SHOULD
</default-conventions>

<default-conventions domain="dead-code">
**Dead Code**：无调用者、不可达分支、未读变量、未使用的导入
Severity: COULD
</default-conventions>

<default-conventions domain="inconsistent-error-handling">
**Inconsistent Error Handling**：混用异常/错误码、类型不一致、吞掉错误
Severity: SHOULD
Exception: 项目为不同错误类别指定了不同处理方式
</default-conventions>

---

## 文件组织规范

<default-conventions domain="test-organization">
**Test Organization**：扩展现有测试文件；仅在以下情况创建新文件：
- 明确的模块边界 OR 超过 500 行 OR 需要不同 fixture
Severity: SHOULD（针对不必要的碎片化）
</default-conventions>

<default-conventions domain="file-creation">
**File Creation**：优先扩展现有文件；仅在以下情况创建新文件：
- 明确的模块边界 OR 300-500 行以上 OR 独立职责
Severity: COULD
</default-conventions>

---

## 测试规范

<default-conventions domain="testing">
**Principle**：测试行为，不测试实现。快速反馈。

**Test Type Hierarchy**（优先顺序）：

1. **集成测试**（最高价值）
   - 测试终端用户可验证的行为
   - 使用真实系统/依赖（如 testcontainers）
   - 在边界处验证组件交互
   - 这里才是真正的价值所在

2. **基于属性的/生成式测试**（推荐）
   - 用不变量断言覆盖广泛输入空间
   - 捕获人类忽视的边界情况
   - 用于具有明确输入/输出契约的函数

3. **单元测试**（谨慎使用）
   - 仅用于高度复杂或关键逻辑
   - 风险：维护负担、重构时易碎
   - 优先使用覆盖相同行为的集成测试

**测试放置**：测试是实现里程碑的一部分，不单独形成里程碑。里程碑在其测试通过之前不算完成。这在开发期间创造快速反馈。

**DO**：

- 使用真实依赖的集成测试（testcontainers 等）
- 对不变量丰富的函数使用基于属性的测试
- 用参数化 fixture 替代重复测试体
- 测试终端用户可观察的行为

**DON'T**：

- 测试外部库/依赖的行为（超出范围）
- 对简单代码进行单元测试（维护成本超过价值）
- mock 自有依赖（使用真实实现）
- 测试可能改变的实现细节
- 适用参数化时逐个变体写测试

Severity: SHOULD（违规），COULD（错过的机会）
</default-conventions>

---

## 现代化规范

<default-conventions domain="version-constraints">
**Version Constraint Violation**：使用了项目文档目标版本中不存在的特性
Requires: 已记录的目标版本
Severity: SHOULD
</default-conventions>

<default-conventions domain="modernization">
**Modernization Opportunity**：遗留 API、冗长模式、手动重新实现标准库功能
Severity: COULD
Exception: 项目要求使用遗留模式
</default-conventions>

---

## 测试策略默认值

<default-conventions domain="testing-strategy">
**Default Test Type Preferences**（项目文档未指定时适用）：

| 类型        | 默认策略                    | 原因                      |
| ----------- | --------------------------- | ------------------------- |
| Unit        | 基于属性（quickcheck）      | 少量测试，多变量          |
| Integration | 行为导向，真实依赖          | 终端用户可验证            |
| E2E         | 生成式数据集                | 确定性重放                |

这些是 Tier 3 默认值。用户确认（Tier 1）可覆盖。

Severity: TESTING_STRATEGY_VIOLATION（SHOULD），如在无覆盖的情况下违反。
</default-conventions>
