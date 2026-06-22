# Review Workflow 使用说明

`review-workflow` 用于把一次 review 拆成三个阶段：

1. **Intake**：确认 Review Brief。
2. **Parallel Review**：并行执行 `code-review` 和 `trace-review`。
3. **Aggregate**：按来源聚合结果，保留各自原始章节，不合并问题分类。

## 插件声明

本目录同时提供两套插件声明：

- Codex：`.codex-plugin/plugin.json`
- Claude Code：`.claude-plugin/plugin.json`

两套声明共享同一套 `skills/`、命令和 agent 方法论，但加载机制不同。Claude Code 通过 `.claude-plugin/plugin.json` 中的 `skills`、`commands` 和 `agents` 字段加载对应组件。

Claude Code 只有在该目录被作为插件安装，或作为 skills-directory plugin 加载时，才会读取 `.claude-plugin/plugin.json`。单独把 `review-workflow/` 放在普通仓库目录里不会自动生效。

## Review Brief 的边界

Review Brief **只确认意图**，包括：

- 变更目标：这次为什么改。
- 预期行为：改完后行为应该变成什么样。
- 不变行为：哪些稳定行为不应改变。

Review Brief **不是**审查范围、测试计划、业务路径清单、trace 清单或验收标准。

## code-review 和 trace-review 的关系

`code-review` 和 `trace-review` 是两个独立技能。它们共享同一份已确认 Review Brief，但各自决定自己的审查重点：

- `code-review` 判断代码实现是否满足 Review Brief，以及是否存在正确性、可读性、架构、安全、性能问题；同时检查测试覆盖、依赖纪律和死代码。输出必须保留 Review Basis 覆盖说明、五维审查标签和 Supplemental Quality Gates 状态。
- `trace-review` 判断上线后关键链路、异常点和分支路径是否有足够日志、埋点、关联上下文和错误信息用于排查业务问题。

这两个技能可以审到不同文件、不同分支或不同问题。代码实现可以是正确的，但 trace 仍然可能缺失；trace 补齐也不能替代代码质量问题。

## 推荐用法

使用 `/review` 时：

1. 如果本轮已经有用户明确确认的 Review Brief，直接进入 Parallel Review。
2. 如果没有已确认 Review Brief，先使用 `review-intake` 产出并确认。
3. Review Brief 确认后，并行执行 `code-review` 和 `trace-review`。
4. 最终输出按来源聚合：
   - Code Review：保留完整 `code-review` 来源输出，包括 Review Basis、Findings、Suggestions、Supplemental Quality Gates、Verification Evidence、Verification Gaps 和 Conclusion；不得压缩掉 Suggestion、Nit、五维审查标签或补充质量门状态。
   - Trace Review
   - Aggregate Notes
   - Conflicts
   - Overall Gate

## 禁止做法

- 禁止让 `code-review` 的结论成为 `trace-review` 的前置条件。
- 禁止用 `code-review` 的通过结论覆盖 `trace-review` 的阻断或无法判断结论。
- 禁止用 `trace-review` 的补齐建议替代代码质量问题。
- 禁止把 Review Brief 扩展成审查范围或后续审查材料收集流程。
- 禁止在 Aggregate 中静默丢弃 `code-review` 的 Suggestion、Nit、五维审查标签或补充质量门状态。
