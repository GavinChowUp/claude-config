# 代码质量指南

面向 LLM agent 检测代码异味的 prompt，按认知模式分类组织：

| 文件                               | 认知模式                             | 类别数     |
| ---------------------------------- | ------------------------------------ | ---------- |
| `01-naming-and-types.md`           | 「名称和类型是否表达了意图？」       | 5          |
| `02-structure-and-composition.md`  | 「结构是否合理？」                   | 5          |
| `03-patterns-and-idioms.md`        | 「是否符合惯用法？」                 | 5          |
| `04-repetition-and-consistency.md` | 「是否 DRY 且一致？」                | 5          |
| `05-documentation-and-tests.md`    | 「是否有完善的文档和测试？」         | 4          |
| `06-module-and-dependencies.md`    | 「边界是否清晰？」                   | 2          |
| `07-cross-file-consistency.md`     | 「跨文件是否一致？」                 | 4          |
| `08-codebase-patterns.md`          | 「出现了哪些模式？」                 | 3          |

## 适用性矩阵

| 分组                        | Design Review | Diff Review | Codebase Review | Refactor Design | Refactor Code |
| --------------------------- | :-----------: | :---------: | :-------------: | :-------------: | :-----------: |
| 01 Naming & Types           |      Yes      |     Yes     |       Yes       |       Yes       |      Yes      |
| 02 Structure & Composition  |      Yes      |     Yes     |       Yes       |       Yes       |      Yes      |
| 03 Patterns & Idioms        |      No       |     Yes     |       Yes       |       No        |      Yes      |
| 04 Repetition & Consistency |      No       |     Yes     |       Yes       |       No        |      Yes      |
| 05 Documentation & Tests    |      No       |     Yes     |       Yes       |       No        |      Yes      |
| 06 Module & Dependencies    |      Yes      |     No      |       Yes       |       Yes       |      Yes      |
| 07 Cross-file Consistency   |      Yes      |     No      |       Yes       |       Yes       |      Yes      |
| 08 Codebase Patterns        |      No       |     No      |       Yes       |       No        |      Yes      |

**阶段定义**：

- **Design Review**：在 diff 存在之前评估代码意图
- **Diff Review**：评估计划中提出的代码变更
- **Codebase Review**：在实现完成后评估代码
- **Refactor Design**：分析现有代码的架构/意图质量
- **Refactor Code**：分析现有代码的实现质量

## 格式设计原理

每个文档由前言（primer）加上带编号的分类组成：

```markdown
# [Group Name]

[PRIMER: 2-3 段，确立认知模式]

## Applicability

[表格，说明该文档适用于哪些阶段]

## Evaluation Modes

<design-mode>...</design-mode>
<code-mode>...</code-mode>

---

## 1. Category Name

<principle>
统一所有示例的抽象规则，放在最前面以引导泛化。
</principle>

Detect: 描述评估视角的检测问题。

<grep-hints>
可能表示问题的词项（起点，非定论）：
`pattern1`, `pattern2`
</grep-hints>

<violations>
说明性模式（非穷举）——类似违规也存在：

[severity] Category label

- 示例，带 "e.g." 前缀
- 开放式结尾："Any X that causes Y"
  </violations>

<exceptions>
带原则性测试的边界情形。
</exceptions>

<threshold>
标记的严重度门槛。
</threshold>
```

### 为什么这样有效

| 特性                              | 机制                                                    |
| --------------------------------- | ------------------------------------------------------- |
| Primer 在前                       | 在分类之前建立认知模式                                  |
| 每个分类首先给出 `<principle>`    | 首因效应——早期内容塑造解读方式                         |
| 「起点，非定论」                  | 限定语打破字面锚定                                      |
| 「e.g.,」前缀                     | 明示这是举例而非穷举                                    |
| 开放式兜底                        | 保持违规列表的开放性                                    |
| XML 语义标记                      | 为 LLM 提供结构；对行范围提取透明                       |

## 集成方式

skill 通过行范围提取各节内容（正则：`^## \d+\. (.+)$`）。节内内容自由格式——解析器提取原始文本，由 LLM 解读结构。

### Skill Prompt 补充内容

用以下内容包裹提取的代码块：

```
<interpretation>
Examples illustrate a PRINCIPLE, not exhaustive checklist.
Detect ANY violation of the principle, including unlisted patterns.
</interpretation>
```

在代码块后添加类比提示：

```
GENERALIZATION:
Before searching, identify 2-3 OTHER patterns violating the SAME principle.
Search for BOTH listed exemplars AND self-generated patterns.
```

这能触发领域特定的联想，使检测能力迁移到列表之外的示例。

---

## 隐性知识：设计决策

### 为什么是 8 个文档？

这些文档按**认知模式**组织——评估者在关注什么、如何推理。这带来以下优势：

1. **专注的 agent**：重构 agent 接收单个文档中的某一个分类。文档前言确立认知模式，分类提供聚焦方向。

2. **全面的 QR**：QR agent 接收一个**完整文档**。同一文档中的所有分类使用相同的认知模式，因此在单次 pass 中检查多个分类效率很高。

3. **渐进式披露**：Python 脚本先注入角色/上下文，再由文档前言确立认知模式，最后由分类提供具体内容。

### 为什么要拆分分类？

三个分类（Zombie Code、Naming Consistency、Error Pattern Consistency）同时有文件级和代码库级的变体，因此拆分：

- **文件级**：可在单个 diff 上检查，用于 Diff Review。
- **代码库级**：需要全局代码库视图，用于 Codebase Review/Refactor。

要求 Diff Review agent 发现代码库级问题是不可能的；要求 Refactor agent 忽略代码库级模式则是浪费其能力。

拆分分配：

| 类别                      | 文件范围                                                | 代码库范围                                   |
| ------------------------- | ------------------------------------------------------- | -------------------------------------------- |
| Zombie Code               | Group 03（注释块、不可达分支）                          | Group 08（零引用导出、死模块）               |
| Naming Consistency        | Group 01（同文件内，命名不一致）                        | Group 07（跨模块漂移）                       |
| Error Pattern Consistency | Group 04（类内不一致）                                  | Group 07（跨抽象层级）                       |

### Design 与 Code 两种评估面

每个文档有两种评估模式：

- **Design-mode**：用于 Design Review 阶段评估代码意图。基于描述判断所提设计是否存在问题。

- **Code-mode**：用于 Diff Review、Codebase Review 或 Refactor 阶段评估实际代码。判断实现中是否存在问题。

Group 03、05 没有 design facet（需要实际代码）。Group 01、02、06、07 同时有两种 facet。Group 04、08 有部分 design facet。

### 各 Agent 使用哪些文档？

| 阶段            | 文档           | 模式   | Agent 模型                                     |
| --------------- | -------------- | ------ | ---------------------------------------------- |
| Design Review   | 01, 02, 06, 07 | design | 4 个并行 QR agent（每个文档一个）              |
| Diff Review     | 01-05          | code   | 5 个并行 QR agent（每个文档一个）              |
| Codebase Review | 01-08          | code   | 8 个并行 QR agent（每个文档一个）              |
| Refactor Design | 01, 02, 06, 07 | design | N 个并行 Explore agent（每个分类一个）         |
| Refactor Code   | 01-08          | code   | N 个并行 Explore agent（每个分类一个）         |

每个 QR agent 接收**一个完整文档**并检查其中的所有分类。每个 Refactor agent 接收**一个分类**（从文档中随机采样）。

### 可机器解析的元数据

每个文档在第 1 行包含可机器解析的适用性元数据（HTML 注释）：

```markdown
<!-- applicable_phases: design_review, diff_review, codebase_review, refactor_design, refactor_code -->
```

采用此方案的原因：

1. **无外部依赖**：HTML 注释可用标准库正则解析，无需依赖 YAML/TOML 解析器或 frontmatter 库。

2. **对读者不可见**：元数据不会占用文档可见空间，让前言作为第一个可见内容。

3. **单一事实来源**：评估模式（design vs code）从阶段推导而来。若阶段映射到 design 模式，提取函数使用 `<design-mode>` 内容；若映射到 code 模式，则使用 `<code-mode>` 内容。

提取函数（`lib/workflow/quality_docs.py`）工作方式如下：

```python
def extract_content(doc_path: Path, phase: Phase) -> ExtractedContent | None:
    """Extract phase-appropriate content from code quality document.

    Returns None if document doesn't apply to this phase.
    Returns ExtractedContent with primer, mode guidance, and categories otherwise.
    """
    content = doc_path.read_text()

    # Parse applicability from HTML comment
    phases = _extract_applicable_phases(content)
    if phase.value not in phases:
        return None

    # Derive mode from phase
    mode = PHASE_TO_MODE[phase]

    # Extract sections
    primer = _extract_primer(content)
    mode_guidance = _extract_mode_content(content, mode)
    categories = _extract_categories(content)

    return ExtractedContent(primer, mode_guidance, categories)
```

`PHASE_TO_MODE` 映射确保模式始终以一致方式从阶段推导：

```python
PHASE_TO_MODE = {
    Phase.DESIGN_REVIEW: Mode.DESIGN,
    Phase.DIFF_REVIEW: Mode.CODE,
    Phase.CODEBASE_REVIEW: Mode.CODE,
    Phase.REFACTOR_DESIGN: Mode.DESIGN,
    Phase.REFACTOR_CODE: Mode.CODE,
}
```

这种设计从根本上杜绝了模式与阶段不一致的情况，因为模式是从阶段计算得出的，而非单独存储。
