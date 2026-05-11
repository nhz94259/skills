---
name: spec-driver
description: Spec-Driven 开发流程技能。提供稳定的文件命名约束、目标语句要求、分阶段任务组织、测试用例编号冲突检查、状态校验与新对话接手清单，并内置 Bug 结果沉淀规范，减少返工和偏差。覆盖需求、设计、任务、归档、Bugfix 全流程。
---

# Spec-Driver 开发流程技能

Spec-Driven 开发主流程：需求 → 设计 → AI Review → 用户确认 → 任务拆解 → AI Review → 测试先行开发 → AI Review → 验收。

```
requirements.md → design.md → AI Review → 用户确认 → task-execution-plan.md → tasks-phase-{N}.md → AI Review → 测试先行开发 → AI Review → 验收
                                  │            ↑ 必须暂停                                           │                          │
                                  │                                                               PASS 自动继续              PASS 自动继续
                                  └─ FAIL → 停下治理 → 复审                                        FAIL → 停下治理 → 复审      FAIL → 停下治理 → 复审
                                                                                                                                 ├─ 达到里程碑/完成后 → 归档
                                                                                                                                 └─ 验收发现问题 → bugfix / 缺陷沉淀
```

**禁止**：跳过需求直接编码 | 跳过用户确认直接拆任务 | 跳过 AI Review Gate 进入下一阶段 | 先写实现后补验证

---

## ⚙️ 文档结构

```
docs/
├── dev/                    # 当前开发文档
│   ├── sdd/                # Spec-Driven 开发文档
│   │   ├── requirements.md
│   │   ├── design.md
│   │   ├── task-execution-plan.md
│   │   ├── tasks-phase-{N}.md
│   │   └── reviews/        # AI Review Gate 临时结果，归档时不保留
│   │       ├── requirements-design-review.md
│   │       ├── tasks-phase-{N}-review.md
│   │       └── TASK-P{N}-{NNN}-code-review.md
│   ├── ui/                 # UI 设计稿（单页 HTML mockup）
│   │   └── {功能名}_v1.html
│   └── test/               # E2E 测试报告（gitignore，不提交）
│       ├── e2e-report.md
│       ├── e2e-result.json
│       └── test_report.html
├── archive/                # 归档文档
├── bugfix/                 # 价值问题总计
├── knowledge/              # 适合长期跟着仓库的知识库
```


## 防返工护栏（8 条强制规则）

### 1. 文件命名

`docs/dev/sdd/` 下正式执行文档文件名是**固定的**，禁止随意改名或并行维护多套同类主文件：

| 文件类型 | 固定文件名 | 禁止 |
|----------|-----------|------|
| 需求 | `requirements.md` | ~~feature-requirements.md~~ |
| 设计 | `design.md` | ~~feature-design.md~~ |
| 执行计划 | `task-execution-plan.md` | ~~feature-plan.md~~ |
| 阶段任务 | `tasks-phase-{N}.md` | ~~tasks.md~~ / ~~feature-tasks.md~~ |
| Bugfix | `bugfix-plan{N}.md` / `bugfix-task{N}.md` | — |

同一活跃规格上下文只保留一套当前标准文件；新增阶段在原文件续写，并新增对应的 `tasks-phase-{N}.md`。
`docs/dev/research/{主题}.md` 用于记录讨论、探索、对比和阶段性备注；它可以作为背景输入，但**不是后续实现、验收和归档的真实依据来源**。凡是要进入执行面的结论，必须回写到 `requirements.md`、`design.md`、`task-execution-plan.md`、`tasks-phase-{N}.md` 或 `bugfix-*` 文档。
辅助调研文档统一放在 `docs/dev/research/` 目录下，命名使用 `{主题}.md`。

### 2. 目标语句

`requirements.md` 开头必须有 `## 目标`（2-3 句话）。后续设计、任务和实现都围绕此目标校准，避免范围漂移。

### 3. 多阶段整合

统一按分阶段模式组织任务。即使当前只有一个阶段，也使用 `task-execution-plan.md` + `tasks-phase-1.md`：

- `requirements.md` 追加新 US，已完成的标 ✅
- `design.md` 追加新章节，已完成的折叠为摘要
- `task-execution-plan.md` 追加或更新 Phase 行
- `tasks-phase-{N}.md` 新建，N = 已有最大 + 1

### 4. TC 编号冲突

分配前用仓库可用的搜索方式确认无冲突（如 `rg "TC-{前缀}-"`）。
按功能域命名前缀：如 `TC-AUTH` / `TC-SYNC` / `TC-EXPORT`。已有前缀不要在不同功能语义下复用。

### 5. 状态校验

标 ✅ 前必须：实现改动实际存在 + 对应验证已通过。
禁止未验证就标 ✅，也禁止只改文档状态不校准真实代码状态。
一次状态更新只允许落到一个目标项：单个 `TASK-P{N}-{NNN}` 或单个 `G{N}`。禁止在一次验证中顺手把其他任务或 Gap 一起标 ✅。

### 6. 新对话接手

按顺序：仓库指导文件（如 `AGENTS.md` / `CLAUDE.md`）→ `task-execution-plan.md` → 当前 `tasks-phase-{N}.md` / `bugfix-task{N}.md` → 检查文件命名 → 校准状态 → 确认目标语句 → 开始开发。

### 7. AI Review Gate

关键阶段必须由 Codex native subagent 以只读方式执行 AI Review Gate。Review 是自动判定点，不是默认人工暂停点：`PASS` 且 `Blocking: false` 时主流程自动继续；`FAIL` 或存在阻断项时必须停下治理，修复后重新 review。

只有以下情况需要暂停等待用户：
- 设计调用链分析按设计规范要求必须用户确认
- 用户明确要求“review 后暂停”或“等我确认”
- review 暴露出需求边界、产品取舍、破坏性变更等必须由用户决策的问题

强制 review 类型：
- `requirements-design-review`：检查 `requirements.md` 与 `design.md` 是否一致；未通过禁止进入用户确认和任务拆解
- `tasks-phase-{N}-review`：检查任务是否完整覆盖设计、TC、依赖和验证；未通过禁止进入 Phase 开发
- `TASK-P{N}-{NNN}-code-review`：检查任务代码、测试、交付凭证和坏味道；未通过禁止标记任务 ✅

> 📄 详细规则、结果模板与 subagent prompt：[references/ai-review-gate-guide.md](references/ai-review-gate-guide.md)

### 8. 质量优先的并发执行

并发只用于提升吞吐，不能降低门禁。主 agent 仍是唯一流程 owner，负责依赖判断、lane 分配、冲突处理、结果整合、状态更新和最终放行。

可并发：
- 只读分析：调用链探索、TC 查重、现状盘点、风险识别
- 独立实现：依赖已完成、写入范围不重叠、验证方式明确的任务
- 独立 review：不同输出文件的只读 AI Review Gate

必须串行：
- 共享文件改造、公共接口变更、迁移脚本、全局状态或配置变更
- `tasks-phase-{N}.md` / `bugfix-task{N}.md` / `task-execution-plan.md` 的状态更新
- Phase 级集成验证、验收结论和归档清理

并发前必须确认：依赖已完成、写入范围不重叠、验证方式明确、review 输出路径唯一。并发后必须由主 agent 汇总 lane 结果并执行集成验证；集成验证失败时不得更新 Phase 完成状态。

---

## 各阶段规范（按需查阅）

### 需求文档

核心：用户视角、可测量验收标准、US 编号全局递增。

> 📄 详细模板与编写要点：[references/requirements-spec-guide.md](references/requirements-spec-guide.md)

### 设计文档

核心：**必须有从入口到落地的调用链/流程分析并与用户确认**、数据模型和关键接口要有明确约束、每个修改点有 before/after 代码示意或伪 diff、TC 编号分配前先查重。若调研文档中有可执行结论，必须在 design 中重新落成正式方案，不能直接把 research 作为实现依据。
涉及新 UI、新功能面板、新页面布局或多区块组合设计时，先走 Design Mockup First：用单 HTML 低保真原型在 `docs/dev/ui/` 对齐布局与信息架构，用户确认后再写真实代码。

> 📄 详细结构与评审清单：[references/design-spec-guide.md](references/design-spec-guide.md)
> 📄 UI 原型先行规范：[references/design-mockup-first-guide.md](references/design-mockup-first-guide.md)

### 任务清单

核心：统一按 Phase 组织、任务粒度可验证、先固化验证再改实现、大需求按阶段推进。需要提速时，按 `Lane`、`写入范围`、`可并发` 标注可并行任务；共享写入和状态更新保持串行。

补充（可选交付）：在规划 `tasks-phase-{N}.md` 后，进入该 Phase 前应询问用户是否需要同步产出「测试故事文档」。
- 询问建议话术：`是否需要我同时产出本阶段测试故事（tasks-phase-{N}-test-story.md），用于界面验收与冒烟回归？`
- 用户确认需要时：创建 `docs/dev/sdd/tasks-phase-{N}-test-story.md`
- 用户不需要时：记录为可选项跳过，不影响主流程继续执行

> 📄 分阶段规则与验证闭环：[references/tasks-spec-guide.md](references/tasks-spec-guide.md)

### 功能归档

触发：用户说“归档”、`archive`、“清空工作区”或阶段完成需要沉淀时，进入功能归档流程。
核心：将 `docs/dev/` 当前需要长期保留的内容归档到 `docs/archive/YYYYMMDD-<主题名>/`，创建 `ARCHIVE-README.md`，更新 `docs/archive/README.md`，并按归档规范清空工作区。AI review 文件只服务当前阶段治理，归档时不复制，归档后不保留。

> 📄 归档流程与目录规范：[references/archive-spec-guide.md](references/archive-spec-guide.md)

### Bugfix 流程

核心：先盘后修、`bugfix-plan` → `bugfix-task`、P0 先修、需要沉淀时直接补 `docs/bugfix/` 结果文档和索引。
这里定义的是 **过程 + 结果规范**：如何盘点 Gap、拆分修复批次、执行验证，以及何时把修复结果沉淀成可检索缺陷记录。

> 📄 Gap 盘点与修复规范：[references/bugfix-spec-guide.md](references/bugfix-spec-guide.md)

### 接口文档

`docs/api/` 用于维护需要长期保存和跨需求复用的接口文档，例如 API 列表、请求/响应字段、IPC/HTTP/SDK/内部模块接口约定和接口变更说明。若需求或设计产生新的接口约定，应先写入正式设计文档；需要长期维护时，再同步沉淀到 `docs/api/`。

### 参考代码

`docs/code/` 用于放本地参考代码，已加入 `.gitignore`，不提交到仓库。参考代码只能作为背景输入，不能替代正式需求、设计和任务文档。

### 稳定知识

`docs/knowledge/` 用于沉淀适合长期保存、不易腐化、可跨需求复用的知识。不要把临时讨论、当前任务状态或易随版本变化的接口细节放入该目录。

---

## 速查清单

**开始新功能**：□ 目标语句 □ 约束事项 □ design 关联 US □ TC 无冲突 □ 用户确认后才拆任务 □ 文件命名规范

**设计完成**：□ requirements-design-review 为 PASS □ 无阻断项 □ 仅在调用链确认点暂停等用户

**开始新 Phase**：□ requirements 追加 US □ design 追加章节 □ plan 追加或更新行 □ tasks-phase-{N} □ Lane / 写入范围 / 可并发已标注 □ tasks-phase-{N}-review 为 PASS □ 旧 US 标 ✅

**新对话接手**：□ 指导文件 → plan / bugfix → tasks □ 文件命名 □ 状态 vs 代码校准 □ 目标语句

**任务完成**：□ 实现已落地 □ 局部验证通过 □ AI code review 为 PASS □ 多 lane 已集成验证 □ 单目标交付凭证 □ 更新状态
**Bugfix 完成**：□ P0 优先处理 □ 并发 Gap 无共享写入 □ bugfix-plan / bugfix-task 状态更新 □ 相关验证通过 □ 需要沉淀时更新 `docs/bugfix/_INDEX.md` 和 `bug-{YYYYMMDD}-{序号}-{描述}.md`
