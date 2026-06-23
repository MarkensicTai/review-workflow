---
name: review-workflow
description: 先确认 Review Brief，再通过 reviewer agent 或直接执行 SKILL 完成两类独立审查
---

使用 review-workflow 插件，按三个阶段执行：

1. Intake：
   - 已有用户确认的 Review Brief 时，直接进入 Parallel Review。
   - 否则使用 `review-workflow:review-intake` skill 产出并确认 Review Brief；skill 不可用且能定位插件目录时，回退读取 `skills/review-intake/SKILL.md`。
   - Intake 只输出待确认或已确认的 Review Brief，禁止生成审查关注点或后续审查结构。
   - 未得到已确认 Review Brief 时，停止流程。
2. Parallel Review：Review Brief 确认后，基于同一份已确认 Review Brief 和当前代码变更，执行两个独立审查：
   - 如果当前环境支持 agent 委托，必须在同一个 assistant turn 同时委托 `code-reviewer` 和 `trace-reviewer`，分别读取并遵循对应 skill。
   - `code-reviewer` 执行合并前代码质量审查。
   - `trace-reviewer` 执行上线后可追踪性审查。
   - 如果当前环境不支持 agent 委托，则由主 agent 直接使用 `review-workflow:code-review` 和 `review-workflow:trace-review` skill。
   - 如果 skill 能力不可用，且当前环境能定位插件目录，才回退读取插件内 `skills/code-review/SKILL.md` 和 `skills/trace-review/SKILL.md`。
   - 回退到主 agent 时，可以不实际并行，但必须保持输入、结论和问题分类独立；不得把任一方输出作为另一方前置条件。
3. Aggregate：按来源聚合展示、排序和标注冲突；保留 `code-review` 和 `trace-review` 的独立结论、原始章节和缺失项。
   - Code Review：保留 `code-review` 的完整来源输出，包括 Review Basis、Findings、Suggestions、Supplemental Quality Gates、Verification Evidence、Verification Gaps 和 Conclusion；不得压缩掉 Suggestion、Nit、五维审查标签或补充质量门状态。
   - Trace Review：保留 `trace-review` 的原始章节、问题分类、缺失项和结论。
   - Aggregate Notes：只写跨来源排序、重复点和补充说明；不得替代来源输出。
   - Conflicts：标注互相冲突或需要用户决断的结论。
   - Overall Gate：只做聚合状态，不覆盖来源结论。任一来源为 `Request changes` 时为 `Request changes`；没有 `Request changes` 但任一来源为 `Unable to determine` 时为 `Unable to determine`；仅当 `code-review` 和 `trace-review` 均为 `Approve` 时为 `Approve`。
