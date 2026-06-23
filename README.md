# Review Workflow

`review-workflow` 是一个本地 review 工作流插件，同时提供 Codex、Claude Code 和 Qoder 三套插件声明，用于把一次代码评审拆成三个阶段：

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
- `code-review`：执行合并前代码质量审查，覆盖正确性、可读性、架构、安全、性能，并检查测试覆盖、依赖纪律和死代码；输出包含 Review Basis、五维覆盖说明和 Supplemental Quality Gates 状态。
- `trace-review`：执行上线后可追踪性审查，检查关键链路、异常分支、日志、埋点、关联上下文、错误信息、隐私、噪声和成本边界。

## 命令

- Codex 插件声明：`.codex-plugin/plugin.json`
- Claude Code 插件声明：`.claude-plugin/plugin.json`
- Qoder 插件声明：`.qoder-plugin/plugin.json`
- Codex：`commands/review.toml`
- Claude Code：`.claude/commands/review.md`
- Qoder：`commands/review-workflow.md`，注册为 `/review-workflow`，避免和 Qoder 内置 `/review` 冲突。

## Agents

- Codex custom agents：
  - `.codex/agents/code-reviewer.toml`
  - `.codex/agents/trace-reviewer.toml`
- Claude Code agents：
  - `.claude/agents/code-reviewer.md`
  - `.claude/agents/trace-reviewer.md`
- Qoder agents：
  - `agents/code-reviewer.md`
  - `agents/trace-reviewer.md`

这些 agent 是只读评审模板。它们不修改文件，不重新访谈需求，也不把 Review Brief 扩展成测试计划、验收标准或完整回归范围。

## 使用方式

使用 Codex / Claude Code 的 `/review`，或 Qoder 的 `/review-workflow` 时，固定执行三段流程：

1. **Intake**：如果本轮已经有用户明确确认的 Review Brief，直接使用它进入 Parallel Review；如果没有，必须先使用 `review-intake` 产出并确认。
2. **Parallel Review**：Review Brief 确认后，以同一份 Review Brief 和当前代码变更为共同输入，独立执行 `code-review` 和 `trace-review`。如果当前环境支持 subagent/custom agent，优先分别委托 `code-reviewer` 和 `trace-reviewer` 读取并遵循对应 SKILL；如果不支持，由主 agent 直接读取并执行两个 SKILL。
3. **Aggregate**：最终结果由主 agent 整合展示，但保留两类审查的独立结论、原始章节和缺失项。

Aggregate 输出应保留：

- Code Review：Review Basis、Findings、Suggestions、Supplemental Quality Gates、Verification Evidence、Verification Gaps 和 Conclusion。
- Trace Review：Trace Basis、Findings、Suggestions、Trace Evidence、Trace Gaps、Trace Coverage、Privacy / Noise / Cost 和 Conclusion。
- Aggregate Notes。
- Conflicts。
- Overall Gate：任一来源为 `Request changes` 时为 `Request changes`；没有 `Request changes` 但任一来源为 `Unable to determine` 时为 `Unable to determine`；仅当 `code-review` 和 `trace-review` 均为 `Approve` 时为 `Approve`。

## 加载方式

- Codex 读取 `.codex-plugin/plugin.json`、`skills/`、`commands/review.toml` 和 `.codex/agents/*.toml`。
- Claude Code 在该目录被作为 Claude Code plugin 安装后，或作为 skills-directory plugin 加载时，读取 `.claude-plugin/plugin.json`，并由其中的 `skills`、`commands`、`agents` 字段指向同一套 skill 目录和 Claude Code command/agent 文件。
- Qoder 在该目录被作为 local plugin 加载后，读取 `.qoder-plugin/plugin.json`，并扫描 `skills/*/SKILL.md`、`commands/*.md` 和 `agents/*.md`；插件技能以 `review-workflow:<skill>` 形式注册。

三套插件声明共享同一套 review 方法论，但加载机制不同。单独把 `review-workflow/` 放在普通仓库目录里，不会让 Claude Code 或 Qoder 自动加载它；它必须进入对应工具的插件加载路径。

## 独立性约束

- `code-review` 和 `trace-review` 没有先后依赖。
- 不把 `code-review` 输出作为 `trace-review` 的前置条件。
- 不把 `trace-review` 输出作为 `code-review` 的前置条件。
- 代码实现可以正确，但 trace 仍然可能缺失。
- trace 补齐不能替代代码质量问题。
- `Aggregate` 只做聚合、排序和冲突标注，不覆盖任一来源结论。
- 禁止把 Review Brief 扩展成审查范围或后续审查材料收集流程。
- 禁止在 Aggregate 中静默丢弃任一来源的 Suggestion、Nit、缺失项、覆盖说明或结论。

## 目录结构

```text
review-workflow/
+-- .claude-plugin/plugin.json
+-- .codex-plugin/plugin.json
+-- .qoder-plugin/plugin.json
+-- agents/
|   +-- code-reviewer.md
|   +-- trace-reviewer.md
+-- .codex/agents/
|   +-- code-reviewer.toml
|   +-- trace-reviewer.toml
+-- .claude/agents/
|   +-- code-reviewer.md
|   +-- trace-reviewer.md
+-- .claude/commands/review.md
+-- commands/review-workflow.md
+-- commands/review.toml
+-- skills/
|   +-- review-intake/SKILL.md
|   +-- code-review/SKILL.md
|   +-- trace-review/SKILL.md
+-- README.md
```
