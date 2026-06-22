# Review Workflow

`review-workflow` 是一个本地 review 工作流插件，同时提供 Codex 和 Claude Code 两套插件声明，用于把一次代码评审拆成三个阶段：

1. **Intake**：先确认 Review Brief。
2. **Parallel Review**：基于同一份 Review Brief，并行且独立执行 `code-review` 和 `trace-review`。
3. **Aggregate**：按来源聚合结果，保留各自原始章节，不让一个审查结论覆盖另一个。

## 核心概念

Review Brief 只确认本次变更的意图：

- 变更目标：这次为什么改。
- 预期行为：改完后行为应该变成什么样。
- 不变行为：哪些稳定行为不应改变。

Review Brief 不是审查范围、测试计划、业务路径清单、trace 清单或验收标准。`code-review` 和 `trace-review` 使用同一份 Review Brief，但各自决定自己的审查重点。

## 技能

- `review-intake`：在审查前确认对变更目标、预期行为和不变行为的理解是否偏移，产出用户确认后的 Review Brief。
- `code-review`：执行合并前代码质量审查，覆盖正确性、可读性、架构、安全、性能，并检查测试覆盖、依赖纪律和死代码；输出保留 Review Basis 覆盖说明、五维审查标签和 Supplemental Quality Gates 状态。
- `trace-review`：执行上线后可追踪性审查，检查关键链路、异常分支、日志、埋点、关联上下文、错误信息、隐私、噪声和成本边界。

## 命令

- Codex 插件声明：`.codex-plugin/plugin.json`
- Claude Code 插件声明：`.claude-plugin/plugin.json`
- Codex：`commands/review.toml`
- Claude Code：`.claude/commands/review.md`

`/review` 的固定编排是：

1. Intake：没有已确认 Review Brief 时，必须先执行 `review-intake`。
2. Parallel Review：Review Brief 确认后，并行执行 `code-review` 和 `trace-review`。
3. Aggregate：主 agent 只整合展示结果，保留两类审查的独立结论、原始章节和缺失项。

## Agents

- Codex custom agents：
  - `.codex/agents/code-reviewer.toml`
  - `.codex/agents/trace-reviewer.toml`
- Claude Code agents：
  - `.claude/agents/code-reviewer.md`
  - `.claude/agents/trace-reviewer.md`

这些 agent 是只读评审模板。它们不修改文件，不重新访谈需求，也不把 Review Brief 扩展成测试计划、验收标准或完整回归范围。

## 加载方式

- Codex 读取 `.codex-plugin/plugin.json`、`skills/`、`commands/review.toml` 和 `.codex/agents/*.toml`。
- Claude Code 在该目录被作为 Claude Code plugin 安装后，或作为 skills-directory plugin 加载时，读取 `.claude-plugin/plugin.json`，并由其中的 `skills`、`commands`、`agents` 字段指向同一套 skill 目录和 Claude Code command/agent 文件。

两套插件声明共享同一套 review 方法论，但加载机制不同。单独把 `review-workflow/` 放在普通仓库目录里，不会让 Claude Code 自动加载它；它必须进入 Claude Code 的插件加载路径。

## 使用约束

- `code-review` 和 `trace-review` 没有先后依赖。
- 不把 `code-review` 输出作为 `trace-review` 的前置条件。
- 不把 `trace-review` 输出作为 `code-review` 的前置条件。
- 代码实现可以正确，但 trace 仍然可能缺失。
- trace 补齐不能替代代码质量问题。
- `Aggregate` 只做聚合、排序和冲突标注，不覆盖任一来源结论。
- `Aggregate` 必须保留 `Code Review` 的 Review Basis、Findings、Suggestions、Supplemental Quality Gates、Verification Evidence、Verification Gaps 和 Conclusion；不得压缩掉 Suggestion、Nit、五维审查标签或补充质量门状态。

## 目录结构

```text
review-workflow/
+-- .claude-plugin/plugin.json
+-- .codex-plugin/plugin.json
+-- .codex/agents/
|   +-- code-reviewer.toml
|   +-- trace-reviewer.toml
+-- .claude/agents/
|   +-- code-reviewer.md
|   +-- trace-reviewer.md
+-- .claude/commands/review.md
+-- commands/review.toml
+-- skills/
|   +-- review-intake/SKILL.md
|   +-- code-review/SKILL.md
|   +-- trace-review/SKILL.md
+-- README.md
+-- USAGE.md
```

更多使用细节见 `USAGE.md`。
