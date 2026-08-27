# AI Review Gate 规范

AI Review Gate 是 Spec-Driven 主流程里的自动收敛门禁。它通过独立、只读的 subagent 检查当前 scope 是否可继续，不以反复扩张审查范围来追求“还能发现更多问题”。reviewer 不修改输入，只允许写自己的 review 结果文件。

## Gate 基本规则

- 主 agent 不得自评放行；Review 必须由 Codex native subagent 执行。
- Review 结果写入 `docs/dev/sdd/reviews/`，只服务当前治理，不进入功能归档。
- 主流程只在当前输入 revision 的 `Decision: PASS` 且 `Blocking: false` 时继续。
- `FAIL`、`Blocking: true`、结果缺失、元数据不完整或输入 revision 已变化，均视为 gate 未通过。
- Review 不是默认人工暂停点。除非用户明确要求、设计调用链必须确认，或问题需要产品取舍/破坏性决策，否则 PASS 后自动继续。

## Reviewer Execution Boundary

- Review subagent 本身就是当前 Gate 的独立 reviewer。除非主 agent 明确授权，reviewer 禁止 spawn 或 delegate 子 agent、调用外部 LLM 或 CLI 做二次 review、启动同类 review，或要求另一个 agent 复核。
- reviewer 必须单次完成当前 `Review-Mode`。读取范围只限 prompt 明确提供的输入、上游 gate 元数据、`Previous-Review` 的 prior finding ledger，以及当前 `Scope` 判定所必需的章节；不得为了“更全面”读取无关文件、未变更内容或未来 Phase。
- `Evidence` 只能记录本 reviewer 实际读取或执行且有工具结果支持的动作与事实，不得声称未执行、未经工具证据支持的“二次独立复核”。
- 一旦满足当前 Review-Mode 的停止条件，reviewer 必须立即输出最终 Gate；不得再增加复核轮次、扩大范围或请求额外 reviewer。

## ReviewMode

`Review-Mode` 只能是以下三种之一：

| Mode | 使用时机 | 允许检查的范围 |
| --- | --- | --- |
| `initial` | 某 gate 在当前规格上的首次 review，或明确重建基线 | 当前 `Scope` 的完整基线、直接依赖和全局安全/数据不变量 |
| `closure` | 上轮有 blocker，输入经修复后复审 | 上轮 blocker closure、从 `Previous-Review` 到当前 revision 的 diff 影响、全局不变量回归 |
| `incremental-phase` | 新增或变更某个 Phase | 该 Phase、它的直接依赖，以及全局安全/数据不变量 |

`closure` 不得重新扩张到未变更的后续 Phase，也不得把上轮未列出的历史细节重新包装成 blocker。closure 中的新 blocker 只允许来自：

1. 当前 diff 新引入的问题；
2. 当前 diff 暴露、且会使当前 scope 或直接依赖无法验收的问题；
3. 全局安全/数据不变量回归。

`incremental-phase` 不得重审已通过且未变化的 Phase。被当前 Phase 直接依赖的旧内容只检查接口和不变量，不重新做完整扫描。

## Blocking 边界

Finding 只有满足以下任一条件时才能标为 blocker：

1. **需求直接冲突**：正式需求之间冲突，或设计/任务/实现直接违反当前正式需求。
2. **跨 Phase 不可逆不变量**：会破坏跨 Phase 的安全或数据不变量，且进入下一阶段后无法低风险逆转。
3. **当前验收不可实现**：当前 scope 或它的直接依赖缺失、冲突或不可执行，导致当前验收标准无法实现或无法验证。

下列内容不得阻断当前 gate：

- 后续 Phase 才需要补齐的实现细节、优化、观测、扩展能力或任务拆分；
- 不影响当前验收的命名、表达、偏好或可选增强；
- 与当前 diff、当前 scope、直接依赖和全局不变量无关的历史问题；
- 仅以“可能还可以深挖”为理由提出的新一轮审查。

不属于 blocker 但必须在后续 Phase 处理的内容，记为 `Deferred Finding`。reviewer 必须给出 `Target Phase`、evidence 和 required fix；主 agent 必须把它回写到该 Phase 的正式 `requirements.md`、`design.md`、`task-execution-plan.md` 或 `tasks-phase-{N}.md`。Deferred Finding 不阻断早期 Phase，但在目标 Phase 开始前必须已进入正式文档，并在目标 Phase review 中重新判定。

## Input Revision 与失效

每份 review 必须记录以下元数据：

- `Review-Mode`
- `Scope`
- `Input-Revision`：逐个输入文件记录内容 hash；推荐 SHA-256
- `Previous-Review`：前一份同 gate review 的路径和内容 hash；首次 review 写 `none`

`Input-Revision` 是一个文件到 hash 的映射，不能只写提交号、分支名或“最新”。例如：

```yaml
Input-Revision:
  docs/dev/sdd/requirements.md: sha256:...
  docs/dev/sdd/design.md: sha256:...
Previous-Review: docs/dev/sdd/reviews/requirements-design-review.md@sha256:...
```

任一输入 hash 变化后，旧结论立即 `stale`：

- 不得使用旧 PASS 放行下游；
- 必须按变化性质选择 `closure` 或 `incremental-phase` 生成新结论；
- 复审前先读取并保留 Previous-Review 的 finding ledger；
- review 输出覆盖固定路径时，必须先取得旧文件 hash 并把它写入 `Previous-Review`。

唯一例外是 **Review 授权的单任务完成状态转换**。它用于避免“code review PASS → 标记完成 → tasks 文件 hash 变化 → 同一 Gate 立即 stale”的自触发循环，并且必须同时满足：

1. 对应 `TASK-P{N}-{NNN}-code-review` 已对状态变更前的任务定义和实现给出 `PASS / Blocking: false`，`Continue` 明确记录 TASK ID、状态变更前 `tasks-phase-{N}.md` hash，以及该状态单元格的精确 before/after；
2. `tasks-phase-{N}.md` 的变化只有该 TASK 的一个状态单元格从待办变为完成，不得同时修改任务名称、范围、依赖、TC、验收、证据文字、其他 TASK 状态或其他文件输入；
3. 主 agent 一次只更新一个任务状态；后续 reviewer 必须把当前文件中的该状态单元格反向恢复为 before，复算结果必须等于授权记录的 pre-hash，并在 `Decision Evidence` 中记录 `pre-hash -> post-hash`。

满足这三项时，原 `tasks-phase-{N}-review` 与该 TASK code review 继续有效；当前文件的新 hash 仍应在后续 review 中如实记录。多个任务顺序完成时，每个 task review 的当前 tasks hash 是下一次转换的 pre-hash；每次转换都必须有独立 PASS 授权并能按上述反向复算形成连续链。链缺失、TASK ID 不匹配或任一复算失败时，按普通 revision 变化立即 stale。完成状态回退、一次批量状态更新和任何语义变更均不适用此例外。

## Finding Ledger

Review 文件必须维护 ledger，不能只列本轮新问题。每条 finding 至少包含：

| 字段 | 规则 |
| --- | --- |
| `ID` | 同一 gate 内稳定且唯一，如 `RD-001`、`TP2-001`、`P2T003-001` |
| `Status` | 只能是 `open`、`closed`、`deferred` |
| `Class` | `blocker`、`deferred` 或 `non-blocking` |
| `Target Phase` | 当前处理写当前 Phase；deferred 必须写明确目标 Phase |
| `Evidence` | 文件、章节、TC、US、diff 或验证结果的具体证据 |
| `Required Fix` | 可验证的修复或正式文档回写要求；无需修复时写 `none` |

Ledger 延续规则：

- 旧 finding 不得消失、重编号或复用 ID；状态变化在原 ID 上更新。
- blocker 未修复时保持 `open`；证据证明已修复后改为 `closed`。
- 后续 Phase 事项改为 `deferred`，不能删掉或降格成无追踪建议。
- 本轮新增 finding 使用下一个未占用序号，不填补已删除或已关闭编号。
- `closed` 和 `deferred` finding 继续保留，供后续 review 追踪来源。

## 上下游串行 Gate

依赖链严格串行：

```text
requirements-design-review(current requirements/design revision) PASS
  -> tasks-phase-{N}-review
  -> Phase implementation
  -> task/code review
```

启动 `tasks-phase-{N}-review` 前，必须先校验：

1. `requirements-design-review` 存在且格式完整；
2. `Decision: PASS` 且 `Blocking: false`；
3. 其中 requirements/design 的 `Input-Revision` hash 与当前文件一致。

任一条件不满足时，tasks reviewer **只返回**：

```text
UPSTREAM-GATE-BLOCKED
```

此时不得扫描 `task-execution-plan.md` 或 `tasks-phase-{N}.md`，不得写入/覆盖 tasks review 结果，也不得生成全量 findings。

## Review 并发

只允许无依赖且输出隔离的同层 review 并发。例如，彼此无任务依赖、输入 scope 不互相改变、各写独立文件的多个 task code review 可以并发。

以下情况必须串行：

- requirements-design 与 tasks-phase 等上下游 gate；
- 同一 gate 的 `initial`、修复和 `closure`；
- 共享直接依赖、共享可变输入或相同输出文件的 review；
- Phase 级 review 与其下属 task review；
- 任一 review 的结论会改变另一 review 的合法输入范围。

“只读”或“输出文件不同”单独都不足以允许并发。并发前必须同时证明：同层、无依赖、输入不会互相失效、输出隔离。

## 判定与停止条件

Review 完成后按以下规则判定：

```text
prior blockers closed
+ current scope no blocker
+ invariants no regression
= PASS / Blocking: false / stop review loop
```

只要三项同时成立，就必须 PASS 并停止本 gate 的 review 循环。Deferred Finding 和 non-blocking suggestion 可以继续存在，但不得成为继续 closure 的理由。

若存在允许范围内的 open blocker，则 `Decision: FAIL`、`Blocking: true`，主 agent 修复后以 `closure` 复审。若问题可以根据正式文档和仓库事实直接修复，不应停下来询问用户；只有产品取舍、需求边界或破坏性决策才暂停。

## Review 类型

| Review 类型 | 输入 | 输出 | 放行条件 |
| --- | --- | --- | --- |
| `requirements-design-review` | `requirements.md`, `design.md` | `reviews/requirements-design-review.md` | 当前 revision PASS 后才可用户确认和任务拆解 |
| `tasks-phase-{N}-review` | 上游 review、`design.md`, `task-execution-plan.md`, `tasks-phase-{N}.md` | `reviews/tasks-phase-{N}-review.md` | 上游有效且当前 Phase PASS 后才可开发 |
| `TASK-P{N}-{NNN}-code-review` | 对应任务、代码 diff、测试结果、交付凭证 | `reviews/TASK-P{N}-{NNN}-code-review.md` | 当前任务 PASS 后才可标完成 |

任务标记完成后，不为该授权状态变更重复发起 closure；按上一节的窄例外保留 Gate 有效性。下一个任务 reviewer 必须验证上一授权转换的 pre-hash、当前 post-hash 和单元格反向复算，再为当前任务签发下一次精确授权。

大型 Phase、跨模块改造或用户明确要求时，可发起 `phase-{N}-code-review`。它不是默认强制 gate；一旦发起，就遵守同样的 mode、revision、ledger、blocking 和停止条件。

## Review 结果模板

```markdown
# AI Review — {review-type}

Review-Type: {review-type}
Review-Mode: {initial|closure|incremental-phase}
Scope: {明确到 Phase、US、TASK 或文件范围}
Input-Revision:
  {input-path}: sha256:{hash}
Previous-Review: {none|review-path@sha256:hash}
Decision: {PASS|FAIL}
Blocking: {false|true}
Reviewer: subagent
Reviewed-At: YYYY-MM-DD

## Finding Ledger
| ID | Status | Class | Target Phase | Evidence | Required Fix |
| --- | --- | --- | --- | --- | --- |
| {stable-id} | {open|closed|deferred} | {blocker|deferred|non-blocking} | {phase} | {evidence} | {fix} |

## Invariant Check
- Safety invariants: {no regression|evidence}
- Data invariants: {no regression|evidence}

## Decision Evidence
- Prior blockers: {none|all closed|open IDs}
- Current scope blockers: {none|IDs}
- Diff impact: {summary}

## Deferred Writeback
- {None|finding ID -> target formal document and section}

## Continue
{PASS 时：允许仅将 TASK-P{N}-{NNN} 在 tasks-phase-{N}.md@sha256:{pre-hash} 中的状态单元格从 `{before}` 改为 `{after}`；不得同时修改其他状态或任务内容。|FAIL 时：禁止继续；修复 open blocker 后以 closure 复审。}
```

没有 finding 时保留表头并写一行 `None`。`Decision: PASS` 必须同时满足 `Blocking: false`；`Decision: FAIL` 必须至少有一个 `open`、`Class: blocker` 的 finding。

## 短 Prompt

### initial

```text
你是 spec-driver initial reviewer。只读检查指定输入，只写指定 review 文件。
Review-Mode: initial。按给定 Scope 建立完整基线，只检查当前 scope、直接依赖和全局安全/数据不变量。
Blocking 仅限需求直接冲突、跨 Phase 不可逆安全/数据不变量、当前 scope 或直接依赖无法实现验收。
后续 Phase 细节记 deferred，给出 target phase 和正式文档回写位置。
记录逐输入 SHA-256、Previous-Review: none，并创建稳定 ID finding ledger。
满足 current scope no blocker + invariants no regression 时 PASS 并停止；不要因可继续深挖而扩张范围。
```

### closure

```text
你是 spec-driver closure reviewer。先读 Previous-Review 及其 ledger，再只读检查当前输入与两次 revision 的 diff。
Review-Mode: closure。范围仅限上轮 open blocker、当前 diff 影响和全局安全/数据不变量回归；不得重扫未变更后续 Phase。
保留全部旧 finding 的 ID 和记录，只更新 open/closed/deferred；新 blocker 仅可由当前 diff 引入/暴露或不变量回归。
记录当前输入 hash 和 Previous-Review 路径@hash。
prior blockers closed + current scope no blocker + invariants no regression 时必须 PASS 并停止。
```

### incremental-phase

```text
你是 spec-driver incremental Phase reviewer。只读检查新增/变更的 Phase、其直接依赖和全局安全/数据不变量。
Review-Mode: incremental-phase。不得重审已通过且未变化的 Phase。
沿用 Previous-Review 的稳定 finding ID；后续 Phase 细节记 deferred，指定 target phase、evidence、required fix 和正式文档回写位置。
Blocking 仅限允许的三类。记录当前输入 hash；当前 Phase 无 blocker且不变量无回归时 PASS 并停止。
```

### tasks-phase

```text
你是 spec-driver tasks-phase reviewer。第一步只校验 requirements-design-review 是否对当前 requirements/design hash 为 PASS 且 Blocking: false。
若上游缺失、失败、格式不完整或 revision 不匹配，只返回 UPSTREAM-GATE-BLOCKED；不要扫描 tasks 输入，不要写/覆盖 review 文件。
上游有效后，按指定 Review-Mode 只读检查当前 Phase 的 design、task-execution-plan、tasks-phase-{N}、TC、直接依赖和验证方式。
Blocking 仅限需求直接冲突、跨 Phase 不可逆安全/数据不变量、当前 Phase 或直接依赖无法实现验收；后续 Phase 事项 deferred 并指定回写位置。
维护稳定 finding ledger 和完整 revision 元数据；满足停止条件即 PASS。
输出：docs/dev/sdd/reviews/tasks-phase-{N}-review.md
```

### task code review

```text
你是 spec-driver task code reviewer。只读检查当前 TASK 的任务定义、代码 diff、测试结果和交付凭证，只写该 TASK 的 review 文件。
按指定 Review-Mode 和 Scope 检查当前任务验收、直接依赖、diff 影响及全局安全/数据不变量；不扫描无关任务或后续 Phase。
维护稳定 finding ledger、逐输入 hash 和 Previous-Review。Blocking 仅限允许的三类。
先验证已有授权状态转换链：对上一 TASK 的状态单元格做反向恢复并复算 pre-hash；链合法后再审当前任务。
prior blockers closed + current scope no blocker + invariants no regression 时 PASS；Continue 必须写明当前 TASK ID、当前 tasks 文件 pre-hash、状态单元格精确 before/after，并只授权该一次状态变更后停止 review。
```
