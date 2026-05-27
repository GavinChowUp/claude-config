# tests/

skill 工作流框架的测试套件。

## 规则

所有测试必须放在 `tests/` 目录下，并通过 pytest 运行。代码库其他位置不得放置测试文件。

## 文件

| 文件                         | 内容                                                 | 何时阅读                               |
| ---------------------------- | ---------------------------------------------------- | ------------------------------------------ |
| `README.md`                  | 测试框架架构、设计决策        | 理解测试设计、修改测试时 |
| `conftest.py`                | pytest 配置、fixture、共享工具函数     | 修改测试配置、添加 fixture 时      |
| `test_workflow_import.py`    | skill 模块导入测试                            | 调试导入失败时                  |
| `test_workflow_structure.py` | 工作流结构验证测试                 | 调试验证失败时              |
| `test_workflow_steps.py`     | 所有工作流步骤的穷举参数化测试 | 运行工作流测试、调试失败时 |
| `test_domain_types.py`       | BoundedInt、ChoiceSet、Constant 的单元测试   | 测试领域类型行为时               |
| `test_generation.py`         | 测试用例的 schema 提取与输入生成     | 修改测试用例生成逻辑时              |
| `test_ast.py`                | AST 节点与渲染器的基于属性的测试   | 测试 AST 构造与渲染时              |

## 测试执行

```bash
# 运行所有测试
pytest tests/ -v

# 运行指定测试文件
pytest tests/test_workflow_steps.py -v

# 运行指定工作流的测试
pytest tests/ -k deepthink -v

# 仅运行导入测试
pytest tests/test_workflow_import.py -v

# 仅运行结构验证测试
pytest tests/test_workflow_structure.py -v
```
