# 归档 docs/dev（Archive dev workspace）

执行本规范时，按以下步骤将 **docs/dev/** 当前内容归档，并清空工作区。

---

## 1. 确认归档名称与目录

- 向用户确认本轮归档的**英文主题名**（如 `code-structure-refactor`、`feature-xxx`），或由用户直接给出。
- 归档目录：`docs/archive/YYYYMMDD-<主题名>/`（日期为当天，如 20250305）。

## 归档目录结构

归档目录必须覆盖 `docs/dev/` 下需要长期保留的内容，包括根目录 Markdown、`sdd/`（不含 `sdd/reviews/`）、`ui/` 和测试报告快照：

```text
docs/archive/YYYYMMDD-<主题名>/
├── ARCHIVE-README.md
├── *.md
├── sdd/  不含 reviews/
├── ui/  如果有
└── test/ 如果有
```

其中 `ui/` 和 `test-report.html` 按实际存在情况归档；`docs/dev/sdd/reviews/` 只服务当前阶段治理，不归档，归档后也不保留；`docs/dev/test/` 只归档 HTML 报告快照，不归档 JSON 数据文件和大体积产物。

## 2. 执行归档

1. **创建归档目录**
   `mkdir -p docs/archive/YYYYMMDD-<主题名>`

2. **复制 docs/dev 下所有 .md 文件**
   `cp docs/dev/*.md docs/archive/YYYYMMDD-<主题名>/`

3. **复制 docs/dev/sdd/ 子目录**（若存在，排除 review 文件）
   推荐使用 `rsync` 排除 `reviews/`：
   `rsync -a --exclude 'reviews/' docs/dev/sdd/ docs/archive/YYYYMMDD-<主题名>/sdd/`
   若只能使用 `cp -r`，复制后必须删除归档目录中的 review 文件：
   `rm -rf docs/archive/YYYYMMDD-<主题名>/sdd/reviews/`

4. **复制 docs/dev/ui/ 子目录**（若存在）
   UI 设计稿（单页 HTML mockup）一并归档：
   `cp -r docs/dev/ui/ docs/archive/YYYYMMDD-<主题名>/ui/`

5. **复制 docs/dev/test/ 测试报告**（若存在）
   `docs/dev/test/` 已被 gitignore，仅归档 HTML 报告快照：
   `cp docs/dev/test/index.html docs/archive/YYYYMMDD-<主题名>/test-report.html`
   （不归档 JSON 数据文件和大体积产物。）

6. **在归档目录添加 ARCHIVE-README.md**
   - 说明：归档日期、内容来源（docs/dev 快照）、本批主题简述。
   - 列出包含的所有文件及用途（README、sdd/ 文档、ui/ 设计稿、test-report 等）。
   - 明确说明 AI review 文件为临时治理产物，未纳入归档。

7. **更新归档索引**
   在 `docs/archive/README.md` 中：
   - 在「归档条目」中新增本条（归档目录、日期、功能说明）；
   - 在「按日期排序」表中增加一行；
   - 在「变更历史」中增加一行；
   - 将「归档总数」「最后归档日期」更新。

## 3. 清空 docs/dev

1. **删除** docs/dev/ 下已复制到归档的**所有 .md 文件**。
2. **删除已归档子目录**：`docs/dev/sdd/`、`docs/dev/ui/`（若存在）。其中 `docs/dev/sdd/reviews/` 虽未归档，也随当前工作区一起清理，归档后不保留。
3. **保留并重写** `docs/dev/README.md` 为简短说明：
   - 本目录为当前开发工作区；
   - 本次已归档至 `docs/archive/YYYYMMDD-<主题名>/`；
   - 新阶段在此新建 requirements/design/tasks 等，按 Spec-Driven 与 spec-driver 执行；
   - 归档索引见 docs/archive/README.md。

---

## 4. 可选：与用户确认后再清空

若需严格遵循「归档前用户确认」，可在步骤 2 完成后先向用户展示归档路径与文件列表，询问「是否清空 docs/dev？」；用户确认后再执行步骤 3。

---

## 注意：review 目录不归档，归档后不保留

- `docs/dev/sdd/reviews/` — AI Review Gate 文件只服务当前阶段治理，不归档，归档后清理
- `docs/dev/reviews/` — 兼容旧路径；若存在也不归档，归档后清理

## 注意：以下临时目录不归档，保持 .gitignore 控制

- `docs/dev/checkpoints/` — checkpoint 文件由 commit hook 自动生成，不归档

## 执行要求

- 若 `docs/archive/YYYYMMDD-<主题名>/` 已存在，停止并提示用户换主题名或处理已有目录。
- 清空 `docs/dev` 前，必须确认步骤 2 已完成。
- 回答归档位置时，必须以实际文件系统检查结果为准。
