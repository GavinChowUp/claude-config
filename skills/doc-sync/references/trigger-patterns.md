# 触发模式参考

CLAUDE.md 索引表条目的良好触发器示例。

## 列公式

| 文件         | 内容                             | 何时阅读                          |
| ------------ | -------------------------------- | ------------------------------------- |
| `[filename]` | [基于名词的内容描述] | [动作动词] [具体上下文/任务] |

## 按类别划分的动作动词

### 实现类任务

implementing, adding, creating, building, writing, extending

### 修改类任务

modifying, updating, changing, refactoring, migrating

### 调试类任务

debugging, troubleshooting, investigating, diagnosing, fixing

### 理解类任务

understanding, learning, reviewing, analyzing, exploring

## 按文件类型举例

### 源代码文件

| 文件           | 内容                                | 何时阅读                                                                       |
| -------------- | ----------------------------------- | ---------------------------------------------------------------------------------- |
| `cache.rs`     | LRU cache with O(1) operations      | Implementing caching, debugging cache misses, modifying eviction policy            |
| `auth.rs`      | JWT validation, session management  | Implementing login/logout, modifying token validation, debugging auth failures     |
| `parser.py`    | Input parsing, format detection     | Modifying input parsing, adding new input formats, debugging parse errors          |
| `validator.py` | Validation rules, constraint checks | Adding validation rules, modifying validation logic, understanding validation flow |

### 配置文件

| 文件           | 内容                             | 何时阅读                                                                  |
| -------------- | -------------------------------- | ----------------------------------------------------------------------------- |
| `config.toml`  | Runtime config options, defaults | Adding new config options, modifying defaults, debugging configuration issues |
| `.env.example` | Environment variable template    | Setting up development environment, adding new environment variables          |
| `Cargo.toml`   | Rust dependencies, build config  | Adding dependencies, modifying build configuration, debugging build issues    |

### 测试文件

| 文件                 | 内容                        | 何时阅读                                                                     |
| -------------------- | --------------------------- | -------------------------------------------------------------------------------- |
| `test_cache.py`      | Cache unit tests            | Adding cache tests, debugging test failures, understanding cache behavior        |
| `integration_tests/` | Cross-component test suites | Adding integration tests, debugging cross-component issues, validating workflows |

### 文档文件

| 文件              | 内容                                     | 何时阅读                                                                             |
| ----------------- | ---------------------------------------- | ---------------------------------------------------------------------------------------- |
| `README.md`       | Architecture, design decisions           | Understanding architecture, design decisions, component relationships                    |
| `ARCHITECTURE.md` | System design, component boundaries      | Understanding system design, component boundaries, data flow                             |
| `API.md`          | Endpoint specs, request/response formats | Implementing API endpoints, understanding request/response formats, debugging API issues |

### 索引文件（跨领域关注点）

| 文件                      | 内容                               | 何时阅读                                                                    |
| ------------------------- | ---------------------------------- | ------------------------------------------------------------------------------- |
| `error-handling-index.md` | Error handling patterns reference  | Understanding error handling patterns, failure modes, error recovery strategies |
| `performance-index.md`    | Performance optimization reference | Optimizing latency, throughput, resource usage, understanding cost models       |
| `security-index.md`       | Security patterns reference        | Implementing authentication, encryption, threat mitigation, compliance features |

## 按目录类型举例

### 功能目录

| 目录       | 内容                                    | 何时阅读                                                                          |
| ---------- | --------------------------------------- | ------------------------------------------------------------------------------------- |
| `auth/`    | Authentication, authorization, sessions | Implementing authentication, authorization, session management, debugging auth issues |
| `api/`     | HTTP endpoints, request handling        | Implementing endpoints, modifying request handling, debugging API responses           |
| `storage/` | Persistence, data access layer          | Implementing persistence, modifying data access, debugging storage issues             |

### 分层目录

| 目录        | 内容                          | 何时阅读                                                                     |
| ----------- | ----------------------------- | -------------------------------------------------------------------------------- |
| `handlers/` | Request handlers, routing     | Implementing request handlers, modifying routing, debugging request processing   |
| `models/`   | Data models, schemas          | Adding data models, modifying schemas, understanding data structures             |
| `services/` | Business logic, service layer | Implementing business logic, modifying service interactions, debugging workflows |

### 工具目录

| 目录       | 内容                              | 何时阅读                                                                       |
| ---------- | --------------------------------- | ---------------------------------------------------------------------------------- |
| `utils/`   | Helper functions, common patterns | Needing helper functions, implementing common patterns, debugging utility behavior |
| `scripts/` | Maintenance tasks, automation     | Running maintenance tasks, automating workflows, debugging script execution        |
| `tools/`   | Development tools, CLI utilities  | Using development tools, implementing tooling, debugging tool behavior             |

## 反模式

### 过于宽泛（什么都能匹配）

| 文件       | 内容          | 何时阅读               |
| ---------- | ------------- | -------------------------- |
| `config/`  | Configuration | Working with configuration |
| `utils.py` | Utilities     | When you need utilities    |

### 只有内容描述（没有触发器）

| 文件       | 内容                                          | 何时阅读 |
| ---------- | --------------------------------------------- | ------------ |
| `cache.rs` | Contains the LRU cache implementation         | -            |
| `auth.rs`  | Authentication logic including JWT validation | -            |

### 缺少动作动词

| 文件           | 内容             | 何时阅读                      |
| -------------- | ---------------- | --------------------------------- |
| `parser.py`    | Input parsing    | Input parsing and format handling |
| `validator.py` | Validation rules | Validation rules and constraints  |

## 触发器编写指南

- 每个条目用逗号或「or」组合 2-4 个触发器
- 使用动作动词：implementing、debugging、modifying、adding、understanding
- 要具体：「debugging cache misses」而非「debugging」
- 如果触发器超过 4 个，该文件可能职责过多
