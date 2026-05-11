# AI Review Gate 规范

AI Review Gate 是 Spec-Driven 主流程里的自动质量闸门。它的目标不是增加人工等待，而是在阶段切换前用 subagent 独立检查产物：没有阻断问题就继续，有阻断问题就停下治理并复审。

## 核心原则

- Review 必须由 Codex native subagent 执行，主 agent 不得自己给自己放行。
- Review subagent 只读检查，不直接修改文件。
- Review 结果必须写入 `docs/dev/sdd/reviews/`。
- Review 输出路径必须唯一；两个 review 会写同一路径时不得并发写入，应串行复审。
- Review 结果是当前开发阶段的治理凭证，不进入功能归档；归档完成后不保留。
- 主流程只在 `Decision: PASS` 且 `Blocking: false` 时继续。
- `Decision: FAIL`、`Blocking: true`、review 文件缺失或结果格式不完整，都视为未通过。
- 未通过时主 agent 必须停下治理：修复正式文档、任务或代码后重新发起 review。
- Review 不是默认暂停点；除非用户明确要求暂停、设计规范要求用户确认，或 review 暴露必须由用户决策的问题，否则 PASS 后自动继续。

## Review 类型与阻断点

| Review 类型 | 输入 | 输出 | 阻断点 |
| --- | --- | --- | --- |
| `requirements-design-review` | `requirements.md`, `design.md` | `reviews/requirements-design-review.md` | 未通过禁止进入用户确认和任务拆解 |
| `tasks-phase-{N}-review` | `design.md`, `task-execution-plan.md`, `tasks-phase-{N}.md` | `reviews/tasks-phase-{N}-review.md` | 未通过禁止进入 Phase 开发 |
| `TASK-P{N}-{NNN}-code-review` | 对应任务、代码 diff、测试结果、交付凭证 | `reviews/TASK-P{N}-{NNN}-code-review.md` | 未通过禁止标记任务完成 |

可选复核：大型 Phase、跨模块改造、用户明确要求或 Phase 验收风险较高时，可额外发起 `phase-{N}-code-review`，输出到 `reviews/phase-{N}-code-review.md`。它不是默认强制门禁；一旦主动发起，则必须按同样规则处理，PASS 自动继续，FAIL 停下治理并复审。

## 并发 Review 规则

只读 review 可以并发，但必须满足输出隔离和逐 gate 判定：

- `requirements-design-review` 固定写入 `docs/dev/sdd/reviews/requirements-design-review.md`，同一规格上下文只允许一个当前结果。
- `tasks-phase-{N}-review` 固定写入 `docs/dev/sdd/reviews/tasks-phase-{N}-review.md`，每个 Phase 一个当前结果。
- `TASK-P{N}-{NNN}-code-review` 固定写入 `docs/dev/sdd/reviews/TASK-P{N}-{NNN}-code-review.md`，每个任务一个当前结果。
- 多个 `TASK-P{N}-{NNN}-code-review` 可以并发，但每个 review 只检查一个任务、只写对应任务的 review 文件。
- 任一任务 review FAIL，只阻断该任务及依赖它的后续任务；不得因为其他 lane PASS 而批量放行失败任务。
- Phase 级或集成级 review 一旦发起，必须等结论 PASS 后继续。

冲突处理：
- 若目标 review 文件已存在且对应当前输入过期，review agent 可以覆盖该文件生成最新结论。
- 若两个 review 会写同一路径，说明它们检查的是同一 gate，不得并发写入；应串行复审。
- review 文件缺失、路径不符合固定规则或格式不完整，等同于 gate 未通过。

## 自动继续与停下治理

Review 后按以下规则处理：

```text
PASS + Blocking: false
  → 主流程自动继续到下一阶段

FAIL 或 Blocking: true
  → 停下治理
  → 修复阻断项
  → 重新发起同类型 review

需要用户决策
  → 暂停并向用户呈现具体决策点
```

“停下治理”不等于问用户是否继续。若问题可以由主 agent 根据现有需求、设计和仓库事实修复，应直接修复并复审。只有范围、取舍、破坏性变更或用户明确要求的暂停才需要询问用户。

## Review 结果模板

```markdown
# AI Review — {review-type}

Review-Type: {requirements-design-review|tasks-phase-{N}-review|TASK-P{N}-{NNN}-code-review|phase-{N}-code-review}
Reviewed-Input:
- docs/dev/sdd/{input}.md
Decision: PASS
Blocking: false
Reviewer: subagent
Reviewed-At: YYYY-MM-DD

## Blocking Findings
- None

## Non-Blocking Suggestions
- None

## Evidence
- {说明检查了哪些映射、测试、diff 或状态}

## Continue
主流程可继续。
```

FAIL 模板：

```markdown
# AI Review — {review-type}

Review-Type: {requirements-design-review|tasks-phase-{N}-review|TASK-P{N}-{NNN}-code-review|phase-{N}-code-review}
Reviewed-Input:
- docs/dev/sdd/{input}.md
Decision: FAIL
Blocking: true
Reviewer: subagent
Reviewed-At: YYYY-MM-DD

## Blocking Findings
- RD-001: `{文件}` 中 `{位置}` 未覆盖 `{依据}`。
- RD-002: `{文件}` 中 `{位置}` 与 `{依据}` 冲突。

## Non-Blocking Suggestions
- S-001: {非阻断优化建议}

## Required Fixes
- 修复 RD-001。
- 修复 RD-002。

## Continue
禁止继续下一阶段；修复后重新 review。
```

## requirements-design-review

检查目标：确认设计忠实覆盖需求，且没有把未确认范围带入实现面。

检查项：
- `design.md` 是否覆盖所有当前 US。
- 每条验收标准是否有设计落点、流程分支或测试设计。
- 是否存在 `requirements.md` 没有依据的功能扩张。
- 调用链、数据模型、接口、TC 是否支撑需求。
- 调研结论是否已正式回写到 `requirements.md` 或 `design.md`。
- 是否存在需要用户决策但被设计直接拍板的事项。

Subagent prompt：

```text
你是 spec-driver 的 requirements-design reviewer。
只读检查：
- docs/dev/sdd/requirements.md
- docs/dev/sdd/design.md

任务：
1. 检查 design 是否覆盖全部当前 US 和验收标准。
2. 检查 design 是否存在 requirements 没有依据的范围扩张。
3. 检查调用链、数据模型、接口、TC 是否足以支撑需求。
4. 识别必须由用户决策的问题。

输出到：
docs/dev/sdd/reviews/requirements-design-review.md

结论只能是 PASS 或 FAIL。
PASS 时 Blocking 必须为 false。
FAIL 时 Blocking 必须为 true，并列出阻断项、依据文件位置和 required fixes。
不要修改任何文件。
```

## tasks-phase-review

检查目标：确认任务拆解完整覆盖设计，并且每个任务都可验证、可执行、依赖清晰。

检查项：
- `tasks-phase-{N}.md` 是否覆盖 `design.md` 当前 Phase 的所有修改点。
- `task-execution-plan.md` 的 Phase 状态、依赖和任务文件是否一致。
- 每个 TC 是否映射到任务或验证方式。
- 任务是否粒度过大、不可验证、依赖缺失或顺序错误。
- 是否存在需要测试先行但没有测试任务的实现项。

Subagent prompt：

```text
你是 spec-driver 的 task coverage reviewer。
只读检查：
- docs/dev/sdd/design.md
- docs/dev/sdd/task-execution-plan.md
- docs/dev/sdd/tasks-phase-{N}.md

任务：
1. 检查任务是否覆盖设计中的全部修改点、TC、依赖和验证方式。
2. 找出遗漏任务、不可验证任务、过大任务、依赖错误。
3. 判断是否可以进入 Phase 开发。

输出到：
docs/dev/sdd/reviews/tasks-phase-{N}-review.md

结论只能是 PASS 或 FAIL。
PASS 后主流程自动继续。
FAIL 必须给出 required fixes。
不要修改任何文件。
```

## code-quality-review

检查目标：确认代码、测试和状态更新满足任务要求，且没有明显 bug 或坏味道。

检查项：
- 实现是否满足对应 `TASK-P{N}-{NNN}`、`design.md` 和相关 TC。
- 测试是否先行或至少能证明行为，并且结果已记录。
- 是否存在明显 bug、坏味道、重复逻辑、过度抽象、边界遗漏。
- 是否误改无关文件或引入未授权依赖。
- 交付凭证是否足以支撑标 ✅。

Subagent prompt：

```text
你是 spec-driver 的 code quality reviewer。
只读检查当前 TASK 的代码 diff、测试结果和交付凭证。

输入：
- docs/dev/sdd/design.md
- docs/dev/sdd/task-execution-plan.md
- docs/dev/sdd/tasks-phase-{N}.md
- 当前 git diff
- 本任务验证结果

任务：
1. 检查实现是否满足任务、设计和 TC。
2. 检查测试和交付凭证是否足以标记任务完成。
3. 找出 bug、坏味道、过度抽象、边界遗漏和无关改动。

输出到：
docs/dev/sdd/reviews/TASK-P{N}-{NNN}-code-review.md

结论只能是 PASS 或 FAIL。
PASS 后主流程可以标记该任务完成。
FAIL 必须给出 required fixes。
不要修改任何文件。
```
