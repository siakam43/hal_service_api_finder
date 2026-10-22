# hal-service-api-finder Simplify Output Initialization Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Move the output-directory backup rule into section 三, delete the redundant 4.1 subsection, and renumber the snapshot section.

**Architecture:** Two documentation edits in `hal-service-api-finder/SKILL.md`. Task 1 consolidates the backup rule, shell-quoting constraint, and the 4.2 cross-reference. Task 2 deletes the old 4.1 block and renumbers `4.2 阶段快照` to `4.1 阶段快照`. No analysis behavior or snapshot schema changes.

**Tech Stack:** Markdown only. Verification is static: grep checks, heading count, JSON parsing, and `wc -l`.

---

## File Structure

- **Modify:** `hal-service-api-finder/SKILL.md` — sections 三, 四, 六, 十.
- **Do not modify:** snapshot schemas, error table, analysis rules, historical plans/specs, `rr1.txt`.

---

### Task 1: Consolidate backup rule, quoting constraint, and reference

**Files:**
- Modify: `hal-service-api-finder/SKILL.md` (section 三 `### 输出目录初始化`, section 六 `### 阶段产物`, section 十 `## 十、约束规则`)

- [ ] **Step 1: Replace the output-directory initialization block**

Replace:

```text
### 输出目录初始化

- 输出根目录：`<out>` = `<project_dir>/.ethunter_out/hal-service-api-finder`。
- 启动顺序：先按第四章 4.1 备份旧 `<out>` 并新建空目录，再执行本章的 scope 生成。
- 不创建 `tmp/` 子目录；所有快照、scope 文件均直接写入 `<out>`，最终 JSON 仍用临时文件 + `mv`。
- 备份/新建失败 → 报 `output_dir_failed` 并结束。
```

with:

```text
### 输出目录初始化

- 输出根目录：`<out>` = `<project_dir>/.ethunter_out/hal-service-api-finder`。
- 若 `<out>` 已存在，先整体备份为 `<out>_bak`（旧 `<out>_bak` 先删除，只保留最近一次备份），再新建空 `<out>`；不得直接删除旧结果。`<out>` 不存在时保留现有 `<out>_bak`。
```

- [ ] **Step 2: Add the global quoting constraint**

After constraint 10:

```text
10. 全中文提示词，同一概念用词前后一致。
```

insert:

```text
11. 所有 shell 命令中的路径变量使用双引号；`<project_dir>` 按 shell 规则转义。
```

- [ ] **Step 3: Update the 4.2 cross-reference in section 六**

Replace:

```text
- `bind` 对象字段形状见第四章 4.2 的 `02_bind.json`（`function` / `file` / `line` / `host_type` / `dispatch_functions` / `stub` / `service_registration`），不写 `status`。
```

with:

```text
- `bind` 对象字段形状见第四章 4.1 的 `02_bind.json`（`function` / `file` / `line` / `host_type` / `dispatch_functions` / `stub` / `service_registration`），不写 `status`。
```

- [ ] **Step 4: Verify Task 1**

```bash
sed -n '/^### 输出目录初始化/,/^### 分析范围确定/p' hal-service-api-finder/SKILL.md | grep -n 'tmp/\|output_dir_failed\|启动顺序' || echo 'section 3 output-init is clean'
grep -n '^11\. 所有 shell 命令中的路径变量' hal-service-api-finder/SKILL.md
grep -n '第四章 4\.2' hal-service-api-finder/SKILL.md || echo 'no 4.2 cross-reference'
grep -n '第四章 4\.1 的 `02_bind.json`' hal-service-api-finder/SKILL.md
grep -c '^## ' hal-service-api-finder/SKILL.md
```

Expected: section 3 clean; constraint 11 present; no `第四章 4.2`; new `第四章 4.1` reference present; heading count 12.

- [ ] **Step 5: Commit**

```bash
git add hal-service-api-finder/SKILL.md
git -c user.name=siakam43 -c user.email=429872468@qq.com commit -m "docs(hal-service-api-finder): consolidate output-dir backup rule"
```

---

### Task 2: Delete old 4.1 and renumber snapshots

**Files:**
- Modify: `hal-service-api-finder/SKILL.md` (section 四)

- [ ] **Step 1: Shorten the section 四 intro**

Replace:

```text
本 skill 不做任务恢复。每次执行开始时，把旧输出目录整体备份，再新建空目录，从零开始分析。中间结果用于阶段交接与调试。
```

with:

```text
本 skill 不做任务恢复。中间结果用于阶段交接与调试。
```

- [ ] **Step 2: Delete the entire old 4.1 subsection**

Delete from:

```text
### 4.1 输出目录与启动清理
```

through the output directory tree code block, stopping immediately before:

```text
### 4.2 阶段快照
```

The deleted block includes the `<out>`/`<out>_bak` definitions, the bash snippet, the 7 edge-case bullets, the `[ -d ]` note, and the `<out>/` directory tree.

- [ ] **Step 3: Renumber the snapshot subsection**

Replace:

```text
### 4.2 阶段快照
```

with:

```text
### 4.1 阶段快照
```

Do not change the JSON examples or field bullets below the heading.

- [ ] **Step 4: Verify Task 2**

```bash
grep -n '4\.2' hal-service-api-finder/SKILL.md || echo 'no 4.2 references'
grep -n '4\.1' hal-service-api-finder/SKILL.md
grep -n '输出目录与启动清理\|输出目录结构' hal-service-api-finder/SKILL.md || echo 'old 4.1 content removed'
sed -n '/^## 四、/,/^### 4\.1 阶段快照/p' hal-service-api-finder/SKILL.md
python3 - <<'PY'
import json, re, pathlib
text = pathlib.Path("hal-service-api-finder/SKILL.md").read_text(encoding="utf-8")
blocks = re.findall(r"```json\n(.*?)\n```", text, re.S)
for block in blocks:
    json.loads(block)
print(f"{len(blocks)} JSON blocks valid")
PY
grep -c '^## ' hal-service-api-finder/SKILL.md
wc -l hal-service-api-finder/SKILL.md
```

Expected: no `4.2`; `4.1` appears only as the snapshot heading and the section 六 reference; old 4.1 content gone; section 四 intro shortened; 7 JSON blocks valid; 12 headings; line count about 981 (record actual).

- [ ] **Step 5: Commit**

```bash
git add hal-service-api-finder/SKILL.md
git -c user.name=siakam43 -c user.email=429872468@qq.com commit -m "docs(hal-service-api-finder): remove duplicate output-dir section"
```

---

## Plan Self-Review

- **Spec coverage:** spec §3 → Task 1 Step 1; §5 → Task 1 Step 2; §6 → Task 1 Step 3; §4 → Task 2 Steps 1-3; §7 → Tasks 1-2 verification. §2.2 non-changes are untouched.
- **Placeholder scan:** no TBD/TODO; every edit shows exact old/new text; every verification has an expected outcome.
- **Consistency:** `4.1` is used only for the snapshot subsection after Task 2; the section 六 reference is updated in Task 1 before the renumber in Task 2, so no step points to a missing heading.
