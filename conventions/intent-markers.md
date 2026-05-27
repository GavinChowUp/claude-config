# Intent Markers

标记用于抑制对故意代码模式的 QR 检查。

## Format

`:MARKER: [what]; [why]`

- 分号分隔符**必须**存在
- `[what]` = 被标记的具体模式
- `[why]` = 理由（所依赖的不变量、安全保证等）

## Markers

| Marker     | Purpose                          | Example                                              |
| ---------- | -------------------------------- | ---------------------------------------------------- |
| `:PERF:`   | 性能关键的故意选择               | `:PERF: unchecked bounds; loop invariant i<len`      |
| `:UNSAFE:` | 安全关键的故意选择               | `:UNSAFE: raw pointer; caller ensures lifetime`      |
| `:SCHEMA:` | 数据契约偏差                     | `:SCHEMA: field unused; migration pending, rollback` |

## Validation

- 缺少分号或 `[why]` 为空 = MARKER_INVALID（MUST）
- 有效标记 = 跳过对被标记代码的相关检查
- 未标记代码 = 接受全面审查

## QR Behavior

QR 脚本检测到标记后：

1. 验证格式（结构：是否有分号、why 是否非空）
2. 若有效：跳过对被标记代码的分类检查
3. 若无效：报告 MARKER_INVALID（MUST 严重度）
