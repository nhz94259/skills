## 项目与流程

- **开发流程**：需求 -> 设计 -> 用户确认 -> 任务拆解（`tasks-phase-{N}.md`）-> 单任务、测试先行 -> 验收归档。禁止跳过任务拆解直接编码。
- **工作区**：需求、设计、任务文档在 `docs/dev/sdd/`，由 Spec-Driven 规则与技能（保证本地从 aone 已经安装这个 skill  `spec-driver`）驱动。

## Spec-Driver 规则

- 新功能、需求、设计、任务、归档、Bugfix 默认使用 `$spec-driver`；主流程是 `requirements.md -> design.md -> AI Review -> 用户确认 -> task-execution-plan.md -> tasks-phase-{N}.md -> AI Review -> 测试先行开发 -> AI Review -> 验收`。
- 禁止跳过需求直接编码、跳过用户确认直接拆任务、跳过 AI Review Gate 进入下一阶段、先写实现后补验证。
- `docs/dev/sdd/` 正式执行文件名固定为 `requirements.md`、`design.md`、`task-execution-plan.md`、`tasks-phase-{N}.md`、`bugfix-plan{N}.md`、`bugfix-task{N}.md`；同一活跃规格只保留一套当前主文件。
- `docs/dev/research/{主题}.md` 只作为背景输入；进入执行面的结论必须回写到正式 SDD 或 bugfix 文档。
- `requirements.md` 开头必须有 `## 目标`（2-3 句话）；设计、任务和实现都围绕目标校准，避免范围漂移。
- 任务统一按 Phase 组织；即使只有一个阶段，也使用 `task-execution-plan.md` + `tasks-phase-1.md`。
- 分配 TC 编号前先查重（如 `rg "TC-{前缀}-"`）；已有前缀不能在不同功能语义下复用。
- 强制 AI Review Gate 必须由 Codex native subagent 以只读方式执行，主 agent 自评不能替代 review：`requirements-design-review` 不通过不能进入用户确认和任务拆解；`tasks-phase-{N}-review` 不通过不能进入 Phase 开发；`TASK-P{N}-{NNN}-code-review` 不通过不能标任务完成。
- Review `PASS` 且 `Blocking: false` 时自动继续；`FAIL` 或存在阻断项时必须停下治理并复审。仅在设计调用链确认、用户明确要求暂停、或暴露产品取舍/破坏性变更时等待用户。
- 标 ✅ 前必须确认实现已落地、验证已通过、对应 AI code review 已通过；一次状态更新只允许更新一个 `TASK-P{N}-{NNN}` 或一个 `G{N}`。
- 对应 code review `PASS / Blocking: false` 明确授权后的单任务“待办→完成”状态单元格变更，不反向使该 code review 或 Phase tasks review stale；review 必须记录 TASK、pre-hash 和 before/after，下一 reviewer 通过反向恢复单元格复算 pre-hash，连续转换逐项成链；此例外禁止同时修改任务语义、依赖、TC、验收、其他状态或其他 Gate 输入。
- 新对话接手顺序：`AGENTS.md` / `CLAUDE.md` -> `task-execution-plan.md` -> 当前 `tasks-phase-{N}.md` / `bugfix-task{N}.md` -> 检查命名 -> 校准状态 -> 确认目标语句 -> 开始开发。
- Bugfix 由 `$spec-driver` 管过程和结果：Phase 验收或复查发现缺口时，先用 `bugfix-plan{N}.md` 全面盘点 Gap，再按 P0 -> P1 -> P2 拆 `bugfix-task{N}.md` 修复和验证；有价值修复经验再沉淀到 `docs/bugfix/_INDEX.md` 与 `docs/bugfix/bug-{YYYYMMDD}-{序号}-{描述}.md`。
- 涉及新 UI、新功能面板、新页面布局或多区块组合设计时，先走 Design Mockup First；应用内组件遵循 `otia-component-style`。


## 编码行为准则

这些准则与项目规则同时生效，默认偏谨慎；明显的低风险小任务可自行裁量，但不要牺牲正确性。

### 1. 框架优先，禁止重复造轮子

- 任何功能在开发之初，**必须先调研项目已使用的框架/库是否已提供该能力，再查业界是否有成熟方案**。
- 如果已有框架能力可以覆盖需求，优先使用框架方案；禁止无故自己从零实现。
- 自建仅在以下情况允许：框架能力明确不满足、引入新依赖的成本远大于自建、或框架方案引入不可接受的复杂度。
- 决定自建时，必须在设计文档或代码注释中说明”为什么不用现有方案”。
- 教训：流式 Markdown 渲染曾自建 `cleanMarkdown` 手动补围栏，未调研 Streamdown 等成熟方案，导致表格半截渲染、代码块闪烁等问题长期存在。

### 2. 先想清楚再编码

- 不要想当然。实现前先明确关键假设，无法从代码、文档或上下文验证时，要显式说明不确定点。
- 如果存在多个合理解释，不要静默挑一个；说明取舍。只有在歧义会实质影响结果时才暂停提问。
- 如果有更简单的方案，优先采用更简单的方案；必要时明确指出过度设计风险。
- 遇到真正不清楚、不可逆、副作用明显或高风险分支的问题时再问；低风险、可逆的下一步直接推进。
- 修改代码前需要考虑修改范围，小修改，不随意触发大面积重构

### 3. 简单优先

- 只写完成当前需求所需的最少代码，不预埋未被请求的能力。
- 不为单次使用场景提前抽象，不增加”未来可能会用到”的配置、开关或扩展层。
- 不为不可能发生的场景堆错误处理。
- 如果 50 行能解决，就不要写 200 行；写完后主动检查是否还能再简化。
- 单文件默认不得超过 1000 行；接近上限时优先拆分文件，而不是继续堆叠实现。

### 4. 外科手术式改动

- 只改与当前任务直接相关的代码、注释和格式，不顺手重构邻近模块。
- 发现无关的坏味道、死代码或历史问题时，可以指出，但不要借机扩改。
- 保持现有代码风格，除非当前任务明确要求调整。
- 只清理由本次改动新增的无用 import、变量、函数或分支；不要清扫历史遗留物。
- 自检标准：每一处 diff 都应能直接追溯到当前需求。

### 5. 目标驱动执行

- 先把任务翻译成可验证目标，再开始实现。
- 修 bug 时优先先写复现测试；做重构时优先先确认前后行为有保护。
- 多步骤任务先写一个简短计划，并为每一步绑定验证方式。
- 完成前必须运行与改动匹配的测试、类型检查或其它验证；不要用”理论上应该没问题”收尾。
