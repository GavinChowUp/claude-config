# ast/

用于工作流 XML 生成的简化 AST 模块。

## 文件

| 文件                   | 内容                                             | 何时阅读                    |
| ---------------------- | ------------------------------------------------ | ------------------------------- |
| `nodes.py`             | 节点类型（TextNode、CodeNode、ElementNode）     | 理解节点结构    |
| `builder.py`           | 流式构建器 API（W.el()）                      | 构造 AST 节点          |
| `renderer.py`          | XMLRenderer 与 render() 函数                | 将 AST 渲染为 XML 输出     |
| `dispatch.py`          | 派发节点类型（Subagent、Template、Roster） | 子 agent 编排模式 |
| `dispatch_renderer.py` | 派发节点的渲染函数              | 渲染派发 XML          |
| `__init__.py`          | 公共 API 导出                               | 导入 AST 类型             |

## 使用方式

```python
from skills.lib.workflow.ast import W, render, XMLRenderer, TextNode

# 构建步骤标头
doc = W.el("step_header", TextNode("Title"),
           script="myskill", step="1", total="5").build()
output = render(doc, XMLRenderer())

# 构建 current_action 块
action_nodes = [TextNode(a) for a in actions]
doc = W.el("current_action", *action_nodes).build()

# 构建 invoke_after
doc = W.el("invoke_after", TextNode(next_command)).build()
```

## 节点类型

| 类型          | 用途                           |
| ------------- | --------------------------------- |
| `TextNode`    | 纯文本内容                |
| `CodeNode`    | 代码块，可选语言 |
| `ElementNode` | 通用 XML 元素（通过 W.el()）  |

所有专用节点（HeaderNode、ActionsNode 等）已移除。skill 统一使用
`W.el("tag_name", ...)` 生成 XML。

## 派发节点类型

| 类型                   | 模式 | 使用场景                                     |
| ---------------------- | ------- | -------------------------------------------- |
| `SubagentDispatchNode` | 单一  | 顺序工作流（plan -> dev -> QR）     |
| `TemplateDispatchNode` | SIMD    | 同一模板对 N 个目标（$var 替换） |
| `RosterDispatchNode`   | MIMD    | 共享上下文，每个 agent 有各自的 prompt     |

```python
from skills.lib.workflow.ast import (
    TemplateDispatchNode, render_template_dispatch,
    RosterDispatchNode, render_roster_dispatch,
)

# 模板派发：$var 按目标替换
node = TemplateDispatchNode(
    agent_type="general-purpose",
    template="Explore $category in $mode mode",
    targets=({"category": "Naming", "mode": "code"}, ...),
    command='python3 -m skills.explore --category $category',
    model="haiku",
)
xml = render_template_dispatch(node)

# Roster 派发：各自的 prompt，固定命令
node = RosterDispatchNode(
    agent_type="general-purpose",
    shared_context="Background...",
    agents=("Task 1...", "Task 2...", "Task 3..."),
    command='python3 -m skills.subagent --step 1',
    model="sonnet",
)
xml = render_roster_dispatch(node)
```
