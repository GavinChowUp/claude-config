# Claude Code 历史分析

分析 Claude Code 对话历史文件的参考文档。本 skill 提供查询模式和结构性知识，用于从 JSONL 对话日志中提取洞察。

## 适用场景

- 分析历史对话中的 token 用量模式
- 按日期、内容或 skill 使用情况查找对话
- 理解主 agent/子 agent 交互模式
- 调试对话为何变大或行为异常
- 从历史记录中提取特定消息或工具调用

## 不适用场景

- 实时对话分析（使用当前上下文代替）
- 修改对话历史（文件是只追加的日志）
- 跨项目分析（每个项目有独立的历史记录）

## 架构

Claude Code 将对话历史存储在 `~/.claude/projects/` 中，目录名为工作目录路径的编码形式。

```
~/.claude/projects/
  |-- -Users-leon--claude/              # /Users/leon/.claude
  |   |-- {session-uuid}.jsonl          # 主对话
  |   |-- {session-uuid}/
  |       |-- subagents/
  |       |   |-- agent-{hash}.jsonl    # 子 agent 对话
  |       |-- tool-results/             # 大型工具输出
  |-- -Users-leon-git-myproject/        # /Users/leon/git/myproject
      |-- ...
```

### 路径编码

工作目录路径的编码规则：

| 原始路径       | 编码结果       | 规则                |
| -------------- | -------------- | ------------------- |
| `/Users/leon`  | `-Users-leon`  | 首 `/` -> `-`  |
| `/git/project` | `-git-project` | 内部 `/` -> `-` |
| `/.claude`     | `--claude`     | `/.` -> `--`        |

### 消息格式

JSONL 文件中每一行是一条独立消息，包含：

- `type`：消息类型（user、assistant、system、queue-operation）
- `uuid`：此消息的唯一标识符
- `parentUuid`：链接到前驱消息（构成对话链）
- `timestamp`：ISO 8601 时间戳
- `message`：包含角色、内容和用量统计的载荷

assistant 消息有结构化的内容块：

- thinking 块：内部推理（签名保护）
- tool use 块：工具调用，包含名称和输入
- text 块：展示给用户的响应文本

## 隐性知识

### 为何只有文档（没有 Python 脚本）

Shell 命令 + jq 比自定义工具更适合此用途：

1. **格式稳定**：JSONL 具有一致的 schema
2. **查询即席**：没有两次分析是完全相同的
3. **jq 足够强大**：能处理所需的所有 JSON 转换
4. **维护负担**：格式变化时 Python 代码需要更新

文档方式让 LLM 能按需组合查询，而无需学习自定义 API。

### Skill 识别模式

Skill 通过 bash 调用，模式为 `python3 -m skills.{name}.{module}`。该模式足够通用，无需枚举即可捕获所有 skill：

```regex
python3 -m skills\.([a-z_]+)\.
```

捕获组 1 提取 skill 名称。无需维护有效 skill 名称列表。

### 子 agent 关联挑战

子 agent 文件命名为 `agent-{hash}.jsonl`，但哈希值不存储在父对话的 Task 工具调用中。关联需要：

1. 列出该会话下的所有子 agent 文件
2. 读取每个子 agent 的第一条用户消息（包含任务描述）
3. 将描述文本与父对话中的 Task tool_use 输入匹配

这略有不便，但不值得为此构建专用工具——这是很少发生的操作。

### Token 用量字段

assistant 消息中的 `usage` 对象包含：

- `input_tokens`：prompt 中的 token 数（不含缓存）
- `output_tokens`：响应中的 token 数
- `cache_read_input_tokens`：从缓存读取的 token 数
- `cache_creation_input_tokens`：写入缓存的 token 数

总计费输入 = `input_tokens + cache_creation_input_tokens`（缓存读取更便宜）。

## 使用示例

### 查找大型对话

```bash
# 查找超过 1MB 的对话
find "$PROJECT_DIR" -name "*.jsonl" -size +1M

# 获取每个对话的 token 总量
for f in "$PROJECT_DIR"/*.jsonl; do
  tokens=$(jq -s '[.[].message.usage? | select(.) | .input_tokens] | add' "$f")
  echo "$tokens $f"
done | sort -rn | head -10
```

### 分析 Skill 使用情况

```bash
# 某对话中使用了哪些 skill？
grep -oE "python3 -m skills\.[a-z_]+" file.jsonl | \
  sed 's/python3 -m skills\.//' | \
  cut -d. -f1 | \
  sort -u

# 查找所有使用 planner skill 的对话
grep -l "python3 -m skills\.planner\." "$PROJECT_DIR"/*.jsonl
```

### Token 增长分析

```bash
# 显示 token 增长进程（识别上下文在哪里膨胀）
jq -c 'select(.type=="assistant" and .message.usage.input_tokens > 50000) |
  {ts: .timestamp[11:19], tokens: .message.usage.input_tokens}' file.jsonl
```

## 相关 Skill

本 skill 提供历史分析的结构性知识。针对特定模式的分析：

- **refactor**：分析历史会话中的代码质量模式时使用
- **problem-analysis**：调查历史记录中发现的问题根因时使用
