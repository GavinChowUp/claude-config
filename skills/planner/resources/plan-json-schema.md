# Plan JSON Schema v2

JSON-IR 优先架构。`plan.json` 保持权威地位,直到 TW 阶段将其转译为 Markdown。

## Schema 概览

```
plan.json
  plan_id: uuid
  created_at: ISO-8601
  frozen_at: null | ISO-8601

  overview:
    title: string
    problem: string
    approach: string

  planning_context:
    decision_log: [DecisionLogEntry]
    rejected_alternatives: [RejectedAlternative]
    constraints: [Constraint]
    known_risks: [KnownRisk]

  invisible_knowledge:
    architecture: Diagram
    data_flow: Diagram
    structure_rationale: string
    invariants: [string]
    tradeoffs: [string]

  milestones: [Milestone]
  milestone_dependencies: MilestoneDependencies
```

---

## Decision Log Entry

由 Architect 填充。需要多步推理链。

```json
{
  "id": "DL-001",
  "decision": "What was decided",
  "reasoning_chain": "premise -> implication -> conclusion",
  "timestamp": "2024-01-15T10:30:00Z"
}
```

ID 格式:`DL-###`(顺序递增)

---

## Rejected Alternative

关联到导致拒绝该方案的决策。

```json
{
  "id": "RA-001",
  "alternative": "Use Redis for caching",
  "rejection_reason": "Team has no Redis ops experience",
  "decision_ref": "DL-001"
}
```

---

## Constraint

```json
{
  "id": "C-001",
  "type": "technical|organizational|dependency",
  "description": "Must use Python 3.10+",
  "source": "user-specified|doc-derived|inferred"
}
```

---

## Known Risk

```json
{
  "id": "R-001",
  "risk": "API rate limits may cause timeouts",
  "mitigation": "Implement exponential backoff",
  "anchor": "src/client.py:L45-L60",
  "decision_ref": "DL-002"
}
```

---

## Invisible Knowledge

应传递到未来 LLM 会话的知识。

```json
{
  "architecture": {
    "diagram_ascii": "Client --> Gateway --> Services",
    "description": "Request routing pattern..."
  },
  "data_flow": {
    "diagram_ascii": "Input -> Validate -> Transform -> Store",
    "description": "Data pipeline..."
  },
  "structure_rationale": "Why we organized code this way...",
  "invariants": [
    "All public APIs must validate input before processing",
    "Database connections must use connection pooling"
  ],
  "tradeoffs": [
    "Chose simplicity over performance for initial implementation",
    "Using sync IO to avoid complexity; can migrate to async later"
  ]
}
```

---

## Milestone

```json
{
  "id": "M-001",
  "number": 1,
  "name": "Implement rate limiter",
  "files": ["src/ratelimit.py", "tests/test_ratelimit.py"],
  "flags": ["error-handling", "needs-rationale"],
  "requirements": ["Limit to 100 requests per minute per client"],
  "acceptance_criteria": ["Test demonstrates rate limiting behavior"],

  "tests": {
    "files": ["tests/test_ratelimit.py"],
    "type": "unit|integration|property-based",
    "backing": "user-specified|doc-derived|default-derived",
    "scenarios": {
      "normal": ["Under limit requests succeed"],
      "edge": ["Exactly at limit"],
      "error": ["Over limit returns 429"]
    },
    "skip_reason": null
  },

  "code_intents": [...],
  "code_changes": [...],
  "documentation": {...},

  "is_documentation_only": false,
  "delegated_to": null
}
```

---

## Code Intent

由 Architect 填充。描述「做什么」,不描述「怎么做」。

```json
{
  "id": "CI-M-001-001",
  "file": "src/ratelimit.py",
  "function": "check_rate_limit",
  "behavior": "Return True if request allowed, False if rate limited. Use sliding window algorithm.",
  "decision_refs": ["DL-001"],
  "params": {
    "window_size": {
      "value": 60,
      "unit": "seconds",
      "decision_ref": "DL-002"
    }
  }
}
```

ID 格式:`CI-{milestone_id}-###`

---

## Code Change

由 Developer 填充。实现一个 Code Intent。

```json
{
  "id": "CC-M-001-001",
  "intent_ref": "CI-M-001-001",
  "file": "src/ratelimit.py",
  "diff": "--- a/src/ratelimit.py\n+++ b/src/ratelimit.py\n@@ -1,0 +1,15 @@\n+def check_rate_limit(client_id: str) -> bool:\n+    ...",
  "context_lines": {
    "before": ["import time", "from collections import defaultdict"],
    "after": ["class RateLimitError(Exception):"]
  },
  "why_comments": [
    {
      "line_offset": 5,
      "comment": "Sliding window chosen over fixed window to prevent burst at window boundary",
      "decision_ref": "DL-001"
    }
  ]
}
```

ID 格式:`CC-{milestone_id}-###`

关键:`intent_ref` 必须引用现有的 `code_intent.id`

---

## Documentation

由 TW 在代码变更后填充。

```json
{
  "module_comment": "Rate limiting module using sliding window algorithm...",
  "docstrings": [
    {
      "function": "check_rate_limit",
      "docstring": "Check if request from client_id is within rate limit.\n\nArgs:\n    client_id: Unique client identifier\n\nReturns:\n    True if allowed, False if rate limited"
    }
  ],
  "function_blocks": [
    {
      "function": "check_rate_limit",
      "comment": "Sliding window implementation:\n1. Get current timestamp\n2. Remove expired entries\n3. Count remaining entries\n4. Return count < limit",
      "decision_ref": null,
      "source": null
    }
  ],
  "inline_comments": [
    {
      "location": "check_rate_limit:15",
      "comment": "Atomic increment to handle concurrent requests",
      "decision_ref": "DL-003"
    }
  ]
}
```

---

## Milestone Dependencies

```json
{
  "diagram_ascii": "M-001 --> M-002\n        \\--> M-003\nM-002 --> M-004\nM-003 --> M-004",
  "waves": [
    { "wave": 1, "milestones": ["M-001"] },
    { "wave": 2, "milestones": ["M-002", "M-003"] },
    { "wave": 3, "milestones": ["M-004"] }
  ]
}
```

---

## 验证规则

### 引用完整性

1. `code_change.intent_ref` 必须指向同一里程碑中现有的 `code_intent.id`
2. `why_comment.decision_ref` 必须指向现有的 `decision_log.id`
3. `code_intent.decision_refs[]` 必须指向现有的 `decision_log.id`
4. `rejected_alternative.decision_ref` 必须指向现有的 `decision_log.id`
5. `known_risk.decision_ref` 必须指向现有的 `decision_log.id`
6. `inline_comment.decision_ref` 必须指向现有的 `decision_log.id`

### 阶段完整性

**plan-design(Architect)**:

- `overview.title` 必填
- `overview.problem` 必填
- 至少一个里程碑
- 每个里程碑至少有一个 `code_intent`

**plan-code(Developer)**:

- 每个 `code_intent` 有匹配的 `code_change`,且 `intent_ref` 有效

**plan-docs(TW)**:

- 必要位置已填充文档

---

## 时态污染

所有字符串字段必须避免:

1. **变更相对时态**:"will be added"、"new function"、"modified to"
2. **基线引用**:"original"、"existing"、"current"
3. **位置指令**:"see below"、"above section"
4. **规划产物**:"TODO"、"FIXME"、"implement later"
5. **意图泄露**:"should"、"needs to"、"must be implemented"

写作时假设代码已以最终状态存在。
