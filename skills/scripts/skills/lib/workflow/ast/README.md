# AST 模块

工作流输出的类型安全 AST 表示，提供构建器 API 与可插拔渲染器。

## 架构

```
Skills（26 个调用点）
       |
       v
+------------------+
| Builder API      |
| W.header()       |
| W.text_output()  |
+------------------+
       |
       v
+----------------------------------------+
|              AST 节点                  |
| TextNode | HeaderNode | DispatchNode   |
| CodeNode | ActionsNode | RoutingNode   |
| RawNode  | CommandNode | GuidanceNode  |
| ElementNode | TextOutputNode           |
+----------------------------------------+
       |
       v
+------------------+     +------------------+
| XMLRenderer      |     | PlainTextRenderer|
| （主要）         |     | （未来）         |
+------------------+     +------------------+
       |
       v
    str 输出
```

## 数据流

```
Skill 步骤 handler
       |
       | 调用 W.header(script="x", step=1, total=5)
       v
ASTBuilder 累积节点
       |
       | .build() 返回 Document
       v
Document(children=[HeaderNode, ActionsNode, ...])
       |
       | render(doc, XMLRenderer())
       v
XMLRenderer.render() 匹配各节点类型
       |
       | 递归渲染子节点
       v
"<step_header script='x' step='1' total='5'>...</step_header>"
```

## 为何如此设计

### 模块组织

- **nodes.py**：节点定义与构造逻辑分离。导入类型时无需引入构建器依赖。
- **builder.py**：流式 API 与类型定义分离。构建器可独立演进（新增便捷方法），而不影响节点结构。
- **renderer.py**：渲染逻辑与 AST 解耦。多种渲染器（XML、纯文本、JSON）实现同一接口。
- **（已删除）compat.py**：迁移期间的过渡兼容层。所有 skill 迁移至 W.\* 构建器 API 后已移除。

### 设计选择

**冻结 dataclass**：不可变性契合函数式风格，防止意外修改，并允许节点在多次渲染和缓存中安全共享。

**扁平 Union**：工作流输出是顺序组合（Header + Actions + Command），而非嵌套散文。扁平 union 加 `children: list[Node]` 比分层行内/块状区分更契合实际模式。

**每种类型独立 dataclass**：类型安全的字段访问配合 IDE 自动补全。比共享 attrs 字典更明确。是 Python 判别联合的标准模式。

**Builder API**：直接构造需要了解字段名与类型。Builder 提供带自动补全的流式 API，降低 skill 作者的认知负担。

**不可变 Builder 模式**：每个 builder 方法返回带有已累积节点的新 builder 实例，无可变共享状态。函数式风格与用户对简洁 FP 的偏好一致。

**外部 render() 函数**：关注点分离——Document 无需了解渲染器。新增渲染器无需修改 Document 类。无需将节点与渲染器接口耦合即可实现多分派。

## 不变量

AST 模块强制执行的核心不变量：

1. **节点类型为冻结 dataclass**：构造后不可变，不允许字段修改。
2. **Node = 11 种 dataclass 类型的 Union**：按类类型判别，非字段判别。match 语句提供穷举性检查。
3. **children 始终为 list[Node]，永不为 None**：叶节点为空列表。简化渲染逻辑。
4. **RawNode.content 永不为空**：有意使用空字符串时用 TextNode。RawNode 是非结构化内容的应急出口。
5. **渲染器必须处理全部 11 种节点类型**：通过 assertNever 模式强制 match 穷举性。
6. **Builder 方法返回新 builder 实例**：不可变链式调用。最终 .build() 返回 Document。

## 权衡

### 扁平 union vs 类型化子节点

**选择**：扁平 `children: list[Node]`，而非 `children: list[InlineNode]`。

**原因**：虽然失去了编译期嵌套约束，但简化了 API。工作流输出是顺序组合，而非嵌套散文。如有需要，运行期验证可检测无效嵌套。

### Builder vs 直接构造

**选择**：Builder 增加了一层间接，但改善了人体工程学。

**原因**：skill 作者使用 `W.header()`，而非 `HeaderNode(type=NodeType.HEADER, ...)`。带自动补全的流式 API 降低认知负担。直接构造会暴露实现细节。

### 兼容垫片 vs 即时迁移

**选择**：垫片增加了临时代码，但允许渐进式推出。

**原因**：短期复杂度换来风险降低是值得的。Strangler Fig 模式将风险隔离在各个 skill。无需协调部署。大爆炸式重写会同时影响全部 26 个调用点。

## RawNode 应急出口

RawNode 区分「无法结构化」与有意文本（TextNode）。追踪 `raw_nodes/total_nodes` 比例可识别 AST 何时需要扩展。

**阈值**：若超过 20% 的节点是 RawNode，则需扩展 AST。这表明存在设计缺口，需要新增节点类型。

## 扩展 AST

新增节点类型时：

1. 在 `nodes.py` 中添加带类型字段的冻结 dataclass
2. 加入 `Node` 联合类型
3. 在 `builder.py` 的 `ASTBuilder` 中添加 builder 方法
4. 在 `renderer.py` 的 `XMLRenderer` 中添加渲染方法
5. 在 `_render_node()` 的 match 语句中添加对应 case
6. 更新测试以覆盖新节点类型的穷举性检查

如果某个 case 未处理，match 语句会在运行时捕获。
