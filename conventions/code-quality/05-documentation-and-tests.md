<!-- applicable_phases: diff_review, codebase_review, refactor_code -->

# Documentation & Tests

评估代码是否有完善的文档和测试。

**核心问题**：这是否有文档和测试？与代码矛盾的文档比没有文档更糟糕。不传达行为的测试作为文档是失败的。Schema 漂移会导致运行时错误。没有来源文档的生成代码会误导维护者。

**关注点**：

- 与实际代码矛盾的文档
- 测试名称不提供信息
- CLAUDE.md 中缺少生成/vendor 代码的来源说明
- Schema 与代码不匹配（代码中有字段但 schema 没有，反之亦然）

**门槛**：仅标记可证明的错误，而非不完整。过时文档会导致幻觉；缺失文档只是上下文减少。标记不提供行为信息的测试。标记没有 CLAUDE.md 文档的生成/vendor 代码。仅在可证明存在不匹配时才标记 schema 漂移。

<design-mode>
不适用——此组需要实际代码才能评估。
</design-mode>

<code-mode>
评估实际代码时（Diff Review、Codebase Review、Refactor）：

- 文档是否与代码矛盾？
- 测试名称是否传达了行为？
- 生成/vendor 代码是否在 CLAUDE.md 中有文档？
- Schema 定义是否与代码用法一致？

证据格式：引用代码/文档（含 file:line），指出问题所在。
</code-mode>

---

## 1. Documentation Staleness

<principle>
与代码矛盾的文档比没有文档更糟糕。过时的文档会误导读者并导致 bug。
</principle>

Detect: 文档是否与代码矛盾？文档中是否有代码在结构上违反的断言？

<grep-hints>
模式指示词（起点，非定论）：
含参数名的 docstring，@param，@return，TODO，FIXME
</grep-hints>

<violations>
说明性模式（非穷举——类似违规也存在）：

[high] Active contradictions

- docstring 中的参数名不在函数签名中
- docstring 类型与类型注解冲突（当注解存在时）
- 任何与代码在结构上矛盾的断言文档

[medium] Stale claims

- docstring 描述代码从未返回的值
- 注释包含强断言（「always」、「never」、「must」）且代码在结构上违反

[low] Orphaned references

- TODO/FIXME 引用了已完成或已删除的工作
  </violations>

<exceptions>
不完整的文档。缺失的文档。文档中的过时风格。
</exceptions>

<threshold>
仅当文档可证明是错误的时标记，而非仅仅不完整。错误的文档会导致幻觉。
</threshold>

## 2. Test Quality as Documentation

<principle>
测试记录预期行为。当测试名称不传达它们所验证的行为时，它们作为文档是失败的。
</principle>

Detect: 测试是否传达了预期行为？我能仅凭测试名称理解在测试什么吗？

<grep-hints>
模式指示词（起点，非定论）：
`test_works`, `test_ok`, `test_success`, `test_case_`, `test_1`, `assert True`
</grep-hints>

<violations>
说明性模式（非穷举——类似违规也存在）：

[high] Uninformative tests

- 测试名称符合低信息量模式（如 test_works, test_ok, test_success, test_case_1）
- 测试包含 0 个断言
- 任何名称不提供行为信息的测试

[medium] Weak naming

- 测试名称少于 3 个词（不含 test\_ 前缀）
- 测试名称描述实现而非行为

[low] Test smells

- 测试只断言 True、None 或琐碎值
- 多个仅有细微输入差异的相似测试函数（使用参数化/表格驱动）
  </violations>

<exceptions>
引用工单号的测试（如 TEST-1234, JIRA-567）用于可追溯性。名为 test_works 的冒烟测试。
</exceptions>

<threshold>
当测试名称不提供行为信息**且**不是工单/回归引用时标记。
</threshold>

## 3. Generated and Vendored Code Awareness

<principle>
不可维护的代码（生成的、vendor 的）必须明确标记。若无来源文档，维护者可能会尝试修改本应重新生成的代码。
</principle>

Detect: 不可维护的代码是否在 CLAUDE.md 中明确标记？维护者能否判断哪些代码是生成的或 vendor 的？

<grep-hints>
模式指示词（起点，非定论）：
`_generated`, `_pb`, `.pb.go`, `vendor/`, `third_party/`, `node_modules/`
</grep-hints>

<violations>
说明性模式（非穷举——类似违规也存在）：

[high] Missing provenance

- 生成文件在 CLAUDE.md 中缺少重新生成命令
- vendor 目录在 CLAUDE.md 中缺少上游来源
- 任何没有来源文档的生成/vendor 代码

[medium] Unclear ownership

- 复制到 repo 中的外部库缺少来源文档
  </violations>

<exceptions>
有重新生成命令文档的生成文件。有明确上游引用的 vendor 代码。
</exceptions>

<threshold>
当文件/目录符合生成模式（如 *.pb.go, *_generated.*, vendor/, third_party/）**且** CLAUDE.md 中缺少相应的来源说明时标记。
</threshold>

## 4. Schema-Code Coherence

<principle>
Schema 和代码必须保持同步。代码中引用但 schema 中没有的字段（反之亦然）表示会导致运行时错误的漂移。
</principle>

Detect: 代码是否引用了不存在的 schema 字段？是否有任何代码路径都未使用的 schema 字段？

<grep-hints>
模式指示词（起点，非定论）：
Schema 文件扩展名（.proto, .graphql, .json schema），字段访问模式
</grep-hints>

<violations>
说明性模式（非穷举——类似违规也存在）：

[high] Schema drift

- 代码引用 schema 定义中不存在的字段
- 任何代码路径都未使用的 schema 字段（死字段）
- Schema 定义与代码用法之间的任何不匹配

[medium] Type drift

- Schema 与代码表示之间的类型不匹配
  </violations>

<exceptions>
用 :SCHEMA: 标记记录的有意偏差。仅在特定部署配置中使用的字段。
</exceptions>

<threshold>
当代码中的字段名在对应 schema 文件中 0 匹配，或 schema 字段在代码库中 0 引用时标记。
</threshold>

Intent marker：使用 `:SCHEMA:` 来抑制有意偏差的检查（如 `:SCHEMA: field 'legacy_id' unused; migration pending`）。
