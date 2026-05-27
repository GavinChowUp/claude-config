# Planner Resources

## 概述

由 planner 脚本在运行时注入的模板。脚本通过直接 Path 解析加载这些文件,而非使用 `shared/resources.py` 中的 `get_resource()`。

## 加载机制

各脚本按需内联加载资源:

```python
# planner.py:168
format_path = Path(__file__).parent.parent / "resources" / "plan-format.md"

# explore.py:52
format_path = Path(__file__).parent.parent / "resources" / "explore-output-format.md"
```

`shared/resources.py` 中的 `get_resource()` 函数存在但对这些文件未使用。脚本更倾向于内联 Path 解析以保持显式性。
