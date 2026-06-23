---
name: review-workflow
description: 先确认 Review Brief，再通过 reviewer agent 或直接执行 SKILL 完成两类独立审查
---

使用 review-workflow 插件，按三个阶段执行：

1. Intake：如果本轮输入已经包含用户明确确认的 Review Brief，直接进入 Parallel Review；否则必须先使用 `review-workflow:review-intake` skill 产出并获得用户明确确认的 Review Brief。未得到已确认 Review Brief 时，停止流程，不得进入 `code-review` 或 `trace-review`。
2. Parallel Review：Review Brief 确认后，基于同一份已确认 Review Brief 和当前代码变更，执行两个独立审查：
   - 如果当前环境支持 agent 委托，优先并行委托 `code-reviewer` agent 读取并遵循 `review-workflow:code-review` skill，执行合并前代码质量审查。
   - 如果当前环境支持 agent 委托，优先并行委托 `trace-reviewer` agent 读取并遵循 `review-workflow:trace-review` skill，执行上线后可追踪性审查。
   - 如果当前环境不支持 agent 委托，则由主 agent 直接使用 `review-workflow:code-review` 和 `review-workflow:trace-review` skill。
   - 如果 skill 能力不可用，且当前环境能定位插件目录，才回退读取插件内 `skills/code-review/SKILL.md` 和 `skills/trace-review/SKILL.md`。
   - 回退到主 agent 时，可以不实际并行，但必须保持输入、结论和问题分类独立；不得把任一方输出作为另一方前置条件。
3. Aggregate：`code-review` 和 `trace-review` 分别保留独立结论、原始章节和缺失项（`Verification Gaps` / `Trace Gaps`）。主 agent 只负责按来源聚合展示、排序和标注冲突；不得合并问题分类，也不得用任一审查结论覆盖另一方的 `Request changes` 或 `Unable to determine`。
   - Code Review：保留 `code-review` 的完整来源输出，包括 Review Basis、Findings、Suggestions、Supplemental Quality Gates、Verification Evidence、Verification Gaps 和 Conclusion；不得压缩掉 Suggestion、Nit、五维审查标签或补充质量门状态。
   - Trace Review：保留 `trace-review` 的原始章节、问题分类、缺失项和结论。
   - Aggregate Notes：只写跨来源排序、重复点和补充说明；不得替代来源输出。
   - Conflicts：标注互相冲突或需要用户决断的结论。
   - Overall Gate：只汇总阻断状态，不覆盖任一来源结论，也不得作为压缩版 review report 替代 Code Review 或 Trace Review。
