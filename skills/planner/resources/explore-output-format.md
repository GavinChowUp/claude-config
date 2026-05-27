# 探索输出 Schema

使用此 XML 格式组织探索发现。每个区段对应一个特定的规划消费步骤。

```xml
<exploration_output task="[brief task description]">

  <approach_inputs>
    <!-- For Step 3: Approach Generation -->
    <!-- What patterns exist? What constraints apply? What favors which approach? -->

    <patterns>
      <pattern name="[name]" location="[file:line]">
        [How it works. Constraints it imposes. Why it matters for approach selection.]
      </pattern>
    </patterns>

    <constraints>
      <constraint impact="[which approach it favors]">
        [What the constraint is and why it affects approach choice]
      </constraint>
    </constraints>
  </approach_inputs>

  <assumption_inputs>
    <!-- For Step 4: Assumption Surfacing -->
    <!-- What's ambiguous? What policies are implicit? What needs user confirmation? -->

    <ambiguities>
      <ambiguity needs_confirmation="true|false">
        [What's unclear. Options available. Why confirmation needed.]
      </ambiguity>
    </ambiguities>

    <implicit_policies>
      <policy area="[timeout|retry|lifecycle|etc]">
        [Current behavior observed. Whether explicit choice needed.]
      </policy>
    </implicit_policies>
  </assumption_inputs>

  <milestone_inputs>
    <!-- For Step 5: Milestone Planning -->
    <!-- What files? What can fail? What's testable? -->

    <files>
      <file path="[exact path]" purpose="[why modify]">
        [Dependencies. Role in system. Key functions/structures.]
      </file>
    </files>

    <failure_modes>
      <failure risk="[high|medium|low]">
        [What can fail. Impact. Mitigation approach.]
      </failure>
    </failure_modes>

    <test_coverage>
      <tests path="[test file]" type="[unit|integration|property]">
        [What behaviors are tested. Patterns used. Reusable fixtures.]
      </tests>
      <gaps>
        [What's NOT tested that acceptance criteria will need.]
      </gaps>
    </test_coverage>
  </milestone_inputs>

</exploration_output>
```

## 区段说明

### approach_inputs(约 500 token)

包含影响方案选择的模式和约束:

- 现有代码如何处理类似关注点
- 架构约束(依赖关系、接口、约定)
- 不同实现策略的复杂度因素

### assumption_inputs(约 500 token)

包含歧义和隐式策略:

- 推进前需要用户确认的事项
- 观察到的策略默认值(超时、重试、错误处理)
- 存在多个有效选项的架构决策

### milestone_inputs(约 500 token)

包含里程碑规划所需的信息:

- 需要修改的文件及其用途和依赖关系
- 需要缓解的失败模式和风险
- 可测试的行为以及现有测试覆盖情况
