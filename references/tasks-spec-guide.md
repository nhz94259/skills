# 任务清单规范

## 分阶段判定

统一按分阶段模式组织任务。满足任一条件时，建议拆成多个 Phase；若不满足，也保留 `task-execution-plan.md` + `tasks-phase-1.md` 这一套结构：

| 条件 | 说明 |
|------|------|
| 预估超 5 天 | 单文件无法一周消化 |
| 任务超 10 个 | 依赖复杂、难追踪 |
| 涉及多层改造 | 底层+数据+业务+UI 联动 |
| 有明确里程碑 | 每阶段独立可验收 |

## 文件结构

```
docs/dev/sdd/
├── task-execution-plan.md     ← 总纲（必须）
├── tasks-phase-1.md           ← Phase 1
├── tasks-phase-2.md           ← Phase 2
├── reviews/                   ← AI Review Gate 结果
│   ├── tasks-phase-1-review.md
│   └── TASK-P1-001-code-review.md
└── ...
```

**多阶段整合**：新 Phase 在 plan 中追加或更新行，创建 `tasks-phase-{N}.md`（N = 已有最大 + 1），不创建新的 plan 文件。

## task-execution-plan.md 模板

```markdown
# 任务执行指导书 — {标题}
## 文档信息
| 属性 | 内容 |
| 关联需求 | `requirements.md`（US-NNN ~ US-NNN）|
| 关联设计 | `design.md` |

## Phase 依赖关系
[ASCII 图]

## Phase 状态
| Phase | 目标 | 关联 US | 完成后能力 | 验证方式 | 状态 |

## 各 Phase 任务文件
| Phase | 文件 | 任务 ID 前缀 |

## 执行顺序
```

## tasks-phase-{N}.md 模板

```markdown
# Phase {N} 任务清单 — {目标}
> 关联设计 / 关联需求 / Phase 依赖

## 任务清单
| ID | 任务名称 | 关联需求 | Lane | 写入范围 | 可并发 | Dev | Test | 依赖 | 优先级 | 验证方式 |

## 任务详情
### TASK-P{N}-001: {标题}
**文件**：{路径}
**位置**：{方法名/行号}
**改动**：{具体做什么}
**Lane**：{lane 名称，如 lane-ui / lane-core / serial-shared}
**写入范围**：{允许修改的文件或目录}
**可并发**：yes/no，原因：{依赖、共享文件、验证边界}
**TC**：{关联测试用例}

## 执行说明
```

## 可选文档：tasks-phase-{N}-test-story.md

在编写 `tasks-phase-{N}.md` 后、进入该 Phase 开发前，新增一个**可选确认步骤**：

1. 询问用户：是否需要同步产出该 Phase 的测试故事文档。  
2. 若用户确认需要：创建 `docs/dev/sdd/tasks-phase-{N}-test-story.md`。  
3. 若用户不需要：在当轮说明中记录“测试故事本阶段跳过”，继续主流程。

### 询问话术（建议）

`是否需要我同时产出本阶段测试故事（tasks-phase-{N}-test-story.md），用于界面验收与冒烟回归？`

### tasks-phase-{N}-test-story.md 模板（可选）

```markdown
# Task-P{N} Test Story（Phase {N}）

> 适用范围：`docs/dev/sdd/tasks-phase-{N}.md`
> 覆盖任务：`TASK-P{N}-001` ~ `TASK-P{N}-XXX`
> 目标：测试故事与功能点一一对应，可追溯验收

## 一、任务与测试故事映射（1:1）
| 任务 ID | 功能点 | 对应测试故事 | 通过标准（界面可观察） |

## 二、测试故事
### 故事 1（对应 TASK-P{N}-001）
**标题**：{一句话标题}
**步骤**：
1. ...
2. ...
**预期**：
- ...
- ...

## 三、10 分钟冒烟清单
- [ ] {关键链路 1}
- [ ] {关键链路 2}

## 四、补充说明
- 界面验证为主，文件验证为辅
```

### 编写约束（可选文档启用时）

- 测试故事必须与 `tasks-phase-{N}.md` 的任务功能点对齐，映射关系清晰。  
- 每个测试故事应可被非开发同学按步骤执行并观察结果。  
- 优先覆盖：依赖解锁、状态流转、关键产物可见性、失败阻断提示。  

## 任务要求

- **粒度**：每个 TASK 建议 2-8 小时
- **命名**：动词开头（"实现""添加""修复"）
- **编号**：`TASK-P{N}-{NNN}`
- **状态**：⬜ 待办 → 🔄 进行中 → ✅ 完成
- **依赖**：开始前所有依赖必须 ✅
- **验证**：每个 TASK 必须写清对应验证方式或命令
- **Lane**：每个 TASK 必须标注执行 lane，用于识别可并发的独立工作流
- **写入范围**：每个 TASK 必须列出允许修改的文件或目录；范围不清时不得并发
- **可并发**：只有依赖已完成、写入范围不重叠、验证方式明确时才能标 `yes`
- **单目标更新**：一次状态变更只允许更新一个 `TASK-P{N}-{NNN}`，禁止批量把多个任务一起标 ✅

## 并发执行规则

`可并发: yes` 只表示允许主 agent 调度，不表示必须并发。主 agent 必须在启动 lane 前检查依赖、写入范围和验证方式。

可并发：
- 只读分析、TC 查重、调用链探索、风险盘点
- 写入范围不重叠的独立实现任务
- 输出路径不同的逐任务 code review

必须串行：
- 共享文件、公共接口、迁移脚本、全局配置或状态表改造
- `tasks-phase-{N}.md`、`task-execution-plan.md` 的状态更新
- Phase 级集成验证和验收结论

失败回退：
- 依赖未完成：该任务不得进入并发队列
- 写入范围交叉：相关任务回退串行，并指定一个 owner
- 验证方式缺失：先补任务文档并重新发起 `tasks-phase-{N}-review`
- lane 局部验证失败：该 lane 停下治理；无依赖且无共享边界的其他 lane 可继续
- 集成验证失败：不得更新 Phase 完成状态，必须定位到相关 lane 或共享边界后复验

## TDD 执行闭环（强制）

```
0. Phase 开发前确认 tasks-phase-{N}-review 为 PASS；FAIL 时停下治理并复审
1. 确认当前 TASK 的依赖、Lane、写入范围和可并发资格
2. 找 design.md 中对应 TC
3. 创建或更新验证用例，按 TC 写测试/检查步骤
4. 运行 → 先看到失败或缺口（🔴 红灯，验证测试有效）
5. 编写实现代码
6. 运行 → 验证通过（🟢 绿灯）
7. 若有 UI 变更 → 同步稳定自动化标识 + 相关界面验证
8. 输出交付凭证
9. 发起 TASK-P{N}-{NNN}-code-review；PASS 自动继续，FAIL 停下治理并复审
10. 多 lane 场景由主 agent 运行集成验证
11. code review PASS 且必要集成验证通过后更新 tasks 状态
```

**禁止**：
- ❌ 先写实现后补验证
- ❌ 批量改实现后再补验证
- ❌ 测试未通过开始下一任务
- ❌ 写入范围冲突时强行并发
- ❌ 子 agent 批量更新任务状态
- ❌ 新增 UI 不加稳定自动化标识
- ❌ 验证 A 时顺手把 B 一起标 ✅
- ❌ 未输出交付凭证就标 ✅
- ❌ 未验证代码实际存在就标 ✅
- ❌ AI code review 未 PASS 就标 ✅

## 交付凭证

任务标 ✅ 前必须输出：

```markdown
## 交付凭证 — TASK-P{N}-{NNN}
| 验证项 | 依据来源 | 验证方式 | 验证载体 | 预期结果 | 实际结果 |
验证结果：`{仓库测试命令或检查命令}` → {关键结果}
验证结果：`{仓库验证命令、检查步骤或验收记录}` → {关键结果}
集成验证：`{多 lane 集成验证命令或说明}` → {关键结果，单 lane 可写 N/A}
AI Review：`docs/dev/sdd/reviews/TASK-P{N}-{NNN}-code-review.md` → PASS
```

填写要求：

- `依据来源` 只能引用正式执行文档，如 `requirements.md`、`design.md`、`task-execution-plan.md`、`tasks-phase-{N}.md`、`bugfix-*`
- `research` 文档可作为背景输入，但不能单独作为标 ✅ 的唯一依据
- `预期结果` 必须先写清“本次应验证什么”
- `实际结果` 必须明确写出结论；若只验证了 A，也要说明 B 仅回归检查、未改状态

## 分阶段执行流程

```
1. 设计完成 → 判断是否分阶段
2. 创建 task-execution-plan.md
3. 按 Phase 创建 tasks-phase-{N}.md（当前 Phase 详细拆解，标注 Lane / 写入范围 / 可并发，后续待前置完成再拆）
4. 发起 tasks-phase-{N}-review；PASS 自动继续，FAIL 停下治理并复审
5. 询问用户是否需要 `tasks-phase-{N}-test-story.md`（可选）
6. 每个 Phase 独立走 TDD 闭环；可并发任务按 lane 执行，状态更新和集成验证由主 agent 串行处理 → Phase 验收
7. Phase 验收通过 → 更新 plan 状态 → 开始下一 Phase
```
