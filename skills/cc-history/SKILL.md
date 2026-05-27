---
name: cc-history
description: 分析 Claude Code 对话历史文件的参考文档。Reference documentation for analyzing Claude Code conversation history files.
---

# Claude Code 历史分析

查询和分析 Claude Code 对话历史的参考文档。使用 shell 命令和 jq 从 JSONL 对话文件中提取信息。

## 目录结构

```
~/.claude/projects/{encoded-path}/
  |-- {session-uuid}.jsonl          # 主对话
  |-- {session-uuid}/
      |-- subagents/
      |   |-- agent-{hash}.jsonl    # 子 agent 对话
      |-- tool-results/             # 大型工具输出
```

## 项目路径解析

将工作目录转换为项目目录：

```bash
PROJECT_DIR="~/.claude/projects/$(echo "$PWD" | sed 's|^/|-|; s|/\.|--|g; s|/|-|g')"
```

编码规则：

- 首 `/` 变为 `-`
- 普通 `/` 变为 `-`
- `/.`（隐藏目录）变为 `--`

示例：

- `/Users/bill/.claude` -> `-Users-bill--claude`
- `/Users/bill/git/myproject` -> `-Users-bill-git-myproject`

## 消息类型

| 类型              | 描述                                   |
| ----------------- | --------------------------------------------- |
| `user`            | 用户输入消息                           |
| `assistant`       | 模型响应（thinking、tool_use、text）    |
| `system`          | 系统消息                               |
| `queue-operation` | 后台任务通知（子 agent 完成） |

## 消息结构

JSONL 文件中每一行是一个消息对象：

```json
{
  "type": "assistant",
  "uuid": "abc123",
  "parentUuid": "xyz789",
  "timestamp": "2025-01-15T19:39:16.000Z",
  "sessionId": "session-uuid",
  "message": {
    "role": "assistant",
    "content": [...],
    "usage": {
      "input_tokens": 20000,
      "output_tokens": 500,
      "cache_read_input_tokens": 15000,
      "cache_creation_input_tokens": 5000
    }
  }
}
```

assistant 消息内容块：

- `type: "thinking"` - 模型推理（包含 `thinking` 字段）
- `type: "tool_use"` - 工具调用（包含 `name`、`input` 字段）
- `type: "text"` - 文本响应（包含 `text` 字段）

## 常用查询

### 查找对话

```bash
# 按修改时间列出（最新在前）
ls -lt "$PROJECT_DIR"/*.jsonl

# 按日期查找
ls -la "$PROJECT_DIR"/*.jsonl | grep "Jan 15"

# 按内容查找
grep -l "search term" "$PROJECT_DIR"/*.jsonl
```

### 提取消息

```bash
# 按行号获取消息（从 1 开始）
sed -n '42p' file.jsonl | jq .

# 按 uuid 获取消息
jq -c 'select(.uuid=="abc123")' file.jsonl

# 所有用户消息
jq -c 'select(.type=="user")' file.jsonl

# 所有 assistant 消息
jq -c 'select(.type=="assistant")' file.jsonl
```

### 工具调用分析

```bash
# 列出所有工具调用
jq -c 'select(.type=="assistant") | .message.content[]? | select(.type=="tool_use") | {name, input}' file.jsonl

# 按名称统计工具调用次数
jq -c 'select(.type=="assistant") | .message.content[]? | select(.type=="tool_use") | .name' file.jsonl | sort | uniq -c | sort -rn

# 查找特定工具调用
jq -c 'select(.type=="assistant") | .message.content[]? | select(.type=="tool_use" and .name=="Bash")' file.jsonl
```

### Skill 调用检测

模式：`python3 -m skills\.([a-z_]+)\.`

```bash
# 查找所有 skill 调用
grep -oE "python3 -m skills\.[a-z_]+" file.jsonl | sort -u

# 查找使用特定 skill 的对话
grep -l "python3 -m skills\.planner\." "$PROJECT_DIR"/*.jsonl
```

### Token 用量

```bash
# 对话的 token 总量
jq -s '[.[].message.usage? | select(.) | .input_tokens + .output_tokens] | add' file.jsonl

# Token 明细
jq -s '[.[].message.usage? | select(.)] | {
  input: (map(.input_tokens) | add),
  output: (map(.output_tokens) | add),
  cached: (map(.cache_read_input_tokens // 0) | add)
}' file.jsonl

# Token 随时间的变化
jq -c 'select(.type=="assistant") | {ts: .timestamp[11:19], inp: .message.usage.input_tokens, out: .message.usage.output_tokens}' file.jsonl
```

### 分类聚合

```bash
# 按类型统计消息数
jq -s 'group_by(.type) | map({type: .[0].type, count: length})' file.jsonl

# 用户消息的字符数
jq -s '[.[] | select(.type=="user") | .message.content | length] | add' file.jsonl

# thinking 块的字符数
jq -s '[.[] | select(.type=="assistant") | .message.content[]? | select(.type=="thinking") | .thinking | length] | add' file.jsonl
```

### 子 Agent 分析

```bash
# 列出某会话的子 agent
ls "${SESSION_DIR}/subagents/"

# 获取子 agent 的任务描述（第一条用户消息）
jq -c 'select(.type=="user") | .message.content' agent-*.jsonl | head -1

# 在父对话中查找 Task 工具调用（这些调用会派发子 agent）
jq -c 'select(.type=="assistant") | .message.content[]? | select(.type=="tool_use" and .name=="Task") | .input' file.jsonl
```

## 对话分支

每个 `.jsonl` 文件包含**完整的对话树**（所有分支），而非每个分支独立文件。分支通过 `parentUuid` 追踪：

- 当用户回退历史并发出新命令时，新消息获得与分支起点相同的 `parentUuid`
- 多条消息共享同一 `parentUuid` = 兄弟分支（分叉点）

### 检测分叉点

```bash
# 查找所有分叉点（有多个子节点的消息）
jq -s 'group_by(.parentUuid) | map(select(length > 1)) | .[] | {
  parentUuid: .[0].parentUuid,
  branches: length,
  timestamps: [.[].timestamp]
}' file.jsonl

# 显示已知分叉点处的兄弟节点
FORK_POINT="parent-uuid-here"
jq -c --arg fp "$FORK_POINT" 'select(.parentUuid==$fp) | {uuid, ts: .timestamp, preview: (.message.content | tostring)[:100]}' file.jsonl
```

### 提取单一分支

要只筛选某一分支，先在该分支中找到唯一标识符，然后沿祖先链追溯到根节点。

**步骤 1：找到目标消息的 uuid**

```bash
# 按唯一内容
TARGET=$(jq -r 'select(.message.content | tostring | contains("unique-identifier")) | .uuid' file.jsonl | tail -1)

# 按时间戳前缀
TARGET=$(jq -r 'select(.timestamp | startswith("2026-01-28T11:23")) | .uuid' file.jsonl | head -1)
```

**步骤 2：将分支提取为 JSONL 流**

```bash
# 每行输出一条消息（JSONL），从旧到新排序
extract_branch() {
  jq -c -s --arg target "$1" '
    (map({(.uuid): .}) | add) as $lookup |
    {chain: [], current: $target} |
    until(.current == null or ($lookup[.current] | not);
      ($lookup[.current]) as $msg |
      .chain += [$msg] |
      .current = $msg.parentUuid
    ) |
    .chain | reverse | .[]
  ' "$2"
}

# 用法：extract_branch <target-uuid> <file>
extract_branch "$TARGET" file.jsonl | jq -s 'length'
extract_branch "$TARGET" file.jsonl | jq 'select(.type=="user")'
```

**步骤 3：常见分支查询**

```bash
# 消息数
extract_branch "$TARGET" file.jsonl | jq -s 'length'

# 仅用户消息
extract_branch "$TARGET" file.jsonl | jq 'select(.type=="user")'

# 工具调用
extract_branch "$TARGET" file.jsonl | jq 'select(.type=="assistant") | .message.content[]? | select(.type=="tool_use") | {name}'

# 第一条和最后一条消息（验证分支正确）
extract_branch "$TARGET" file.jsonl | jq -s '[.[0], .[-1]] | .[] | {type, ts: .timestamp}'
```

### 工作流：定位与探索

```bash
# 1. 查找对话文件
FILE=$(grep -l "unique-identifier" "$PROJECT_DIR"/*.jsonl)

# 2. 查找匹配消息（可能显示多个分支）
jq -c 'select(.message.content | tostring | contains("unique-identifier")) | {uuid, ts: .timestamp, parentUuid}' "$FILE"

# 3. 从目标分支选取 uuid，然后查询
TARGET="uuid-from-step-2"
extract_branch "$TARGET" "$FILE" | jq 'select(.type=="user") | .message.content'
```

## 关联

子 agent 文件（`agent-{hash}.jsonl`）不直接链接到父 Task 调用。关联步骤：

1. 列出 `{session}/subagents/` 下的所有子 agent 文件
2. 读取每个子 agent 的第一条用户消息，获取任务描述
3. 将描述与父对话中的 Task tool_use 块匹配
