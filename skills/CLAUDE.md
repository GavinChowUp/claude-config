# skills/

基于脚本的 agent 工作流，共享同一套编排框架。

## 必读：修改前请先阅读

**停。在编辑 `skills/scripts/` 中的任何 Python 文件之前，你必须先读 `README.md`。**

README 定义了：

- 文件区段顺序（SHARED PROMPTS -> CONFIGURATION -> MESSAGE TEMPLATES -> MESSAGE BUILDERS -> STEP DEFINITIONS -> OUTPUT FORMATTING -> ENTRY POINT）
- MESSAGE TEMPLATES 内按步骤划分的 prompt 组织方式
- prompt 常量的命名约定（`[PHASE]_[TYPE]`）
- dispatch prompt 的模式（静态模板 vs 构建函数）
- 需要避免的反模式（action factories、前向引用）

不遵循这些模式会产生技术债并导致各 skill 之间不一致。这些模式的存在是因为它们解决了 prompt 可读性和维护性上的真实问题。

**如果还没读过，现在就去读 `README.md`。**

## 文件

| 文件        | 内容                                                      | 何时阅读                    |
| ----------- | --------------------------------------------------------- | ------------------------------- |
| `README.md` | 文件组织、prompt 模式、命名规范、反模式 | 修改任何 skill 代码之前 |

## 子目录

| 目录                  | 内容                                      | 何时阅读                                             |
| --------------------- | ----------------------------------------- | ---------------------------------------- |
| `scripts/`            | 所有 skill 代码的 Python 包根目录    | 执行 skill、调试行为     |
| `planner/`            | 规划与执行工作流          | 创建实现计划            |
| `refactor/`           | 多维度重构分析    | 技术债审查、代码质量      |
| `problem-analysis/`   | 结构化问题拆解          | 理解复杂问题             |
| `decision-critic/`    | 决策压力测试与评审      | 验证架构选择         |
| `deepthink/`          | 开放性问题的结构化推理   | 无现成框架的分析类问题  |
| `codebase-analysis/`  | 系统性代码库探索           | 仓库架构审查           |
| `prompt-engineer/`    | Prompt 优化与工程       | 改进 agent prompt                  |
| `incoherence/`        | 一致性检测                     | 发现规范/实现不匹配   |
| `doc-sync/`           | 文档同步             | 跨 repo 同步文档                |
| `leon-writing-style/` | 风格匹配内容生成          | 生成符合用户风格的内容    |
| `arxiv-to-md/`        | arXiv 论文转 Markdown        | 将论文转为 LLM 可消费格式    |
| `cc-history/`         | Claude Code 对话历史分析 | 查询历史对话、token 用量 |

## 脚本调用

所有 Python skill 脚本均以模块形式从 `scripts/` 调用：

<invoke working-dir=".claude/skills/scripts" cmd="python3 -m skills.<skill_name>.<module> --step 1" />

示例：

<invoke working-dir=".claude/skills/scripts" cmd="python3 -m skills.problem_analysis.analyze --step 1" />
