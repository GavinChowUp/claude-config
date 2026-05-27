# workflow/

工作流编排框架：类型定义、发现机制与基于 AST 的 XML 生成。

## 架构

skill 采用基于 CLI 的步骤调用方式。工作流执行流程为：

```
main() -> format_output() -> print() -> LLM 读取 -> 跟随 <invoke_after>
```

Workflow 与 StepDef 是供内省使用的元数据容器。执行引擎
（Workflow.run()、Outcome、StepContext）已作为死代码移除。

## 文件

| 文件           | 内容                                      | 何时阅读                                        |
| -------------- | ----------------------------------------- | --------------------------------------------------- |
| `core.py`      | Workflow、StepDef、Arg（仅元数据）    | 定义新 skill、工作流结构时             |
| `discovery.py` | 通过 importlib 扫描发现工作流 | 理解拉取式发现机制、排查问题时 |
| `__init__.py`  | 公共 API 导出                        | 导入工作流类型时                            |
| `cli.py`       | 工作流入口点的 CLI 辅助函数     | 添加 CLI 参数、步骤输出辅助函数时           |
| `constants.py` | 共享常量，QR 常量重导出 | 新增常量时                                |
| `types.py`     | 领域类型：Dispatch、AgentRole 等   | QR gate、子 agent 派发、测试域时      |

## 子目录

| 目录          | 内容                            | 何时阅读                   |
| ------------- | ------------------------------- | ------------------------------ |
| `ast/`        | AST 节点、构建器、渲染器    | 为步骤输出生成 XML 时 |
| `formatters/` | 从 ast/ 重导出以保持兼容性 | 新代码直接用 ast/      |

## 测试

```bash
pytest tests/ -v
pytest tests/ -k deepthink -v
```
