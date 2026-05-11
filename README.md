# spec-driver

本说明仅介绍 `spec-driver` skill 约束的实际工作目录空间。

## 工作目录

`spec-driver` 主要围绕 `docs/dev/`、`docs/archive/`、`docs/api/`、`docs/bugfix/`、`docs/knowledge/` 和 `docs/code/` 工作。

```text
docs/
├── dev/       # 正式执行与阶段工作目录
│   ├── README.md
│   ├── sdd/
│   ├── research/
│   ├── ui/
│   └── test/
├── archive/
├── api/
├── bugfix/
├── knowledge/
└── code/       # 本地参考代码，git 忽略切记
```

## `docs/dev/sdd/`

正式执行文档目录。

这里放会直接作为实现、验收、任务推进依据的文档，例如：

- `requirements.md`
- `design.md`
- `task-execution-plan.md`
- `tasks-phase-{N}.md`
- `tasks-phase-{N}-test-story.md`
- `bugfix-plan{N}.md`
- `bugfix-task{N}.md`

这些文档属于正式依据，状态、结论、任务推进都应以这里的内容为准。

`docs/dev/sdd/` 默认应被 git 跟踪；不要把该目录加入 `.gitignore`。AI Review Gate 结果位于 `docs/dev/sdd/reviews/`，只服务当前阶段治理，归档时不保留。

## `docs/dev/research/`

调研与讨论目录。

这里放探索过程、候选方案、对比分析、阶段性备注，例如：

- `docs/dev/research/{主题}.md`

这些文档可以作为背景输入，但不能单独作为正式执行依据。凡是影响需求、设计、任务、Bugfix 的结论，都必须回写到 `docs/dev/sdd/` 中的正式文档。

## `docs/dev/ui/`

界面设计与展示材料目录。

这里通常放：

- 单页 HTML mockup
- 原型说明
- 界面展示稿

如果设计稿会影响正式实现方案，应在 `design.md` 中同步落地说明，不能只停留在 `ui/` 目录。

## `docs/dev/test/`

测试输出与报告目录。

这里通常放：

- 验收报告
- HTML 测试报告快照
- 临时验证产物

该目录更偏验证结果存放区，不替代 `tasks-phase-{N}.md` 或 `bugfix-task{N}.md` 中的正式验证结论。

提交前优先把测试结论写回任务交付凭证或归档报告。大体积、机器生成或仅本地排查用的测试产物不应作为正式依据提交。

## `docs/archive/`

归档目录。

当某一轮需求或阶段完成后，`docs/dev/` 中需要保留的正式材料会归档到：

- `docs/archive/YYYYMMDD-<主题名>/ARCHIVE-README.md`
- `docs/archive/YYYYMMDD-<主题名>/sdd/requirements.md`
- `docs/archive/YYYYMMDD-<主题名>/sdd/design.md`
- `docs/archive/YYYYMMDD-<主题名>/sdd/task-execution-plan.md`
- `docs/archive/YYYYMMDD-<主题名>/sdd/tasks-phase-{N}.md`
- `docs/archive/YYYYMMDD-<主题名>/ui/`（如有）
- `docs/archive/YYYYMMDD-<主题名>/test-report.html`（如有）

归档索引统一维护在 `docs/archive/README.md`。
如存在其他阶段性材料，也可按归档规范一并进入对应归档目录。清空 `docs/dev/` 前必须先完成归档验证。

## `docs/api/`

接口文档维护目录。

这里放需要长期维护和跨需求复用的接口说明，例如：

- API 列表和接口索引
- 请求 / 响应字段说明
- IPC、HTTP、SDK 或内部模块接口约定
- 接口变更说明和兼容性注意事项

若需求或设计产生新的接口约定，应在正式设计文档中说明，并在需要长期维护时同步沉淀到 `docs/api/`。

## `docs/bugfix/`

有价值修复经验沉淀目录。

这里放已经完成修复、值得后续复用的问题记录和排查经验，例如：

- 用户明确反馈过、未来可能再次出现的问题
- 根因复杂、排查路径有复用价值的问题
- 跨会话、跨成员需要复用的修复经验
- 已脱离当前 Phase、成为独立缺陷知识的问题

`docs/dev/sdd/bugfix-plan*.md` / `bugfix-task*.md` 是过程文档；`docs/bugfix/` 是结果沉淀和经验库。

## `docs/knowledge/`

稳定知识沉淀目录。

这里放适合长期保存、不易腐化、可跨需求复用的知识，例如：

- 长期有效的架构原则和边界说明
- 稳定的领域概念、术语和业务规则
- 不依赖具体版本的工程约定
- 经多次验证后仍适用的最佳实践

不要把临时讨论、阶段性判断、易随版本变化的接口细节或当前任务状态放入 `docs/knowledge/`。这些内容应留在 `docs/dev/research/`、`docs/dev/sdd/`、`docs/api/` 或对应任务文档中。

## `docs/code/`

本地参考代码目录。

这里放用于理解、对照、迁移或临时参考的外部代码、样例代码和历史代码片段。该目录只作为本地参考输入，不作为正式需求、设计、任务、归档或缺陷沉淀来源。

该目录已由 `.gitignore` 忽略；提交时不要用强制添加把本地参考代码纳入仓库。
