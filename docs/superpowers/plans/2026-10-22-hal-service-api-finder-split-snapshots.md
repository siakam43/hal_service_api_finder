# hal-service-api-finder Split Snapshot Documentation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Keep chapter 四 as the intermediate-file list plus the shared `error.json`, and move the phase-specific snapshot formats into their phase chapters.

**Architecture:** Two edits in `hal-service-api-finder/SKILL.md`. Task 1 removes the duplicate list from chapter 二 and rewrites chapter 四. Task 2 moves the `01_entry.json` example to chapter 五, repoints the chapter 六 field reference, and moves the `03_trace.json` element-field bullets to chapter 七.

**Tech Stack:** Markdown only. Verification is static: grep checks, JSON parsing, heading count, and `wc -l`.

---

## File Structure

- **Modify:** `hal-service-api-finder/SKILL.md` — chapters 二, 四, 五, 六, 七.
- **Do not modify:** analysis rules, error table, final JSON schema, historical plans/specs, `rr1.txt`.

---

### Task 1: Remove the chapter 二 list and rewrite chapter 四

**Files:**
- Modify: `hal-service-api-finder/SKILL.md` (chapter 二 output overview; chapter 四)

- [ ] **Step 1: Delete the intermediate-results list from chapter 二**

Delete exactly:

```text
中间结果（均位于输出根目录 `<out>`，定义见第三章）：
- scope_files.txt
- scope_roots.txt（仅 clean_code.txt 分支存在）
- 01_entry.json
- 02_bind.json
- 03_trace.json
- error.json（仅失败时）
```

Keep the surrounding final-output path, four-field definition, and “不生成 summary 报告。”.

- [ ] **Step 2: Replace the entire chapter 四 body**

Run this script from the repository root:

````python
from pathlib import Path

path = Path("hal-service-api-finder/SKILL.md")
text = path.read_text(encoding="utf-8")
start = text.index("## 四、中间结果记录")
end = text.index("## 五、阶段一：定位服务注册点")
new_chapter = '''## 四、中间结果记录

中间文件（均位于 `<out>`）：
- scope_files.txt
- scope_roots.txt（仅 clean_code.txt 分支存在）
- 01_entry.json
- 02_bind.json
- 03_trace.json
- error.json（仅失败时）

`01_entry.json`、`02_bind.json`、`03_trace.json` 的格式分别在第五、六、七章说明。`error.json` 为所有阶段共用的失败产物，格式如下。

### 4.1 error.json

仅在失败时写入：

```json
{
  "phase": "locate_entry",
  "code": "service_not_found",
  "message": "未找到 moduleName 为 demo_service 的 HdfDriverEntry",
  "evidence": "已按 moduleName 精确搜索 scope_files.txt 中的全部文件"
}
```

`phase` 取值：`init` / `locate_entry` / `analyze_bind` / `trace_dispatch` / `output`，只作失败位置标签。写入 error.json 后停止；不写最终 JSON。
error.json 只在新 `<out>` 已成功创建后写入；备份/mv/mkdir 失败只在对话中报告。若 error.json 本身写入失败，改为在对话中报告，不再重试。

'''
path.write_text(text[:start] + new_chapter + text[end:], encoding="utf-8")
````

- [ ] **Step 3: Verify Task 1**

```bash
sed -n '/^## 二、/,/^## 三、/p' hal-service-api-finder/SKILL.md | grep -n '中间结果（均位于' || echo 'chapter 2 list removed'
sed -n '/^## 四、/,/^## 五、/p' hal-service-api-finder/SKILL.md | grep -n '#### 01_entry\|#### 02_bind\|#### 03_trace\|不参与恢复\|不做任务恢复' || echo 'chapter 4 is list + error only'
sed -n '/^## 四、/,/^## 五、/p' hal-service-api-finder/SKILL.md | grep -n '中间文件（均位于\|### 4.1 error.json'
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

Expected: chapter 2 list removed; chapter 4 contains only the list and `error.json`; 4 JSON blocks remain (chapter 4 error, chapter 6 02, chapter 7 03, chapter 8 final); 12 headings; line count about 879 (record actual).

- [ ] **Step 4: Commit**

```bash
git add hal-service-api-finder/SKILL.md
git -c user.name=siakam43 -c user.email=429872468@qq.com commit -m "docs(hal-service-api-finder): move intermediate-file list to chapter 4"
```

---

### Task 2: Move snapshot formats into their phase chapters

**Files:**
- Modify: `hal-service-api-finder/SKILL.md` (chapter 五 阶段产物; chapter 六 阶段产物; chapter 七 收尾)

- [ ] **Step 1: Add the 01_entry.json example to chapter 五**

In `### 阶段产物`, after the `entry` sub-bullet of step 1 and before step 2, insert:

```text

   示例：

   ```json
   {
     "service_name": "demo_service",
     "scope": {
       "file_count": 123,
       "clean_code": true,
       "scope_files": "scope_files.txt",
       "scope_roots": "scope_roots.txt"
     },
     "entry": {
       "variable": "g_DemoDriverEntry",
       "file": "/abs/path/...",
       "line": 39,
       "module_name": "demo_service",
       "bind": "HdfDemoDriverBind",
       "init": "HdfDemoDriverInit",
       "release": "HdfDemoDriverRelease",
       "registration": {
         "form": "HDF_INIT",
         "file": "/abs/path/...",
         "line": 50,
         "evidence": "HDF_INIT(g_DemoDriverEntry)"
       }
     }
   }
   ```

```

The three-space indentation keeps the example inside step 1’s list item.

- [ ] **Step 2: Repoint the chapter 六 field reference**

Replace:

```text
- `bind` 对象字段形状见第四章 4.1 的 `02_bind.json`（`function` / `file` / `line` / `host_type` / `dispatch_functions` / `stub` / `service_registration`），不写 `status`。
```

with:

```text
- `bind` 对象字段形状见上方示例（`function` / `file` / `line` / `host_type` / `dispatch_functions` / `stub` / `service_registration`），不写 `status`。
```

- [ ] **Step 3: Add the 03_trace.json element-field bullets to chapter 七**

In 3.9 `收尾`, after the `03_trace.json` JSON code block and before the write-failure bullet, insert:

```text
- `cases` 元素字段：`id`、`dispatcher`、`dispatcher_file`、`dispatcher_line`、`dispatch_site`、`level`、`case_expr`、`case_value`、`outer_cmd_expr`、`outer_cmd_value`、`parent_case_id`、`handlers`、`evidence`；其中 `handlers` 是 `{name, file, line}` 引用列表。
- 顶层 `handlers` 元素字段：`cmd_id_macro`、`cmd_id_value`、`interface_function`、`file`、`definition_line`；`definition_line` 仅用于定位与去重，最终 JSON 不包含。
- `default_cases` 元素字段：`dispatcher`、`dispatcher_file`、`dispatcher_line`、`dispatch_site`、`evidence`。
- `unresolved` 元素字段：`kind`、`detail`、`tried`；`kind` 取 `cmd_value` / `target` / `definition` / `data_deferred` / `cycle`。
```

- [ ] **Step 4: Verify Task 2**

```bash
grep -n '见第四章 4\.1' hal-service-api-finder/SKILL.md || echo 'no chapter-4 reference in chapter 6'
grep -n '见上方示例' hal-service-api-finder/SKILL.md
sed -n '/^## 五、/,/^## 六、/p' hal-service-api-finder/SKILL.md | grep -n '"service_name": "demo_service"' | head -1
sed -n '/^## 七、/,/^## 八、/p' hal-service-api-finder/SKILL.md | grep -n '顶层 `handlers` 元素字段\|`unresolved` 元素字段'
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

Expected: no `见第四章 4.1`; `见上方示例` present; 01 example in chapter 五; element-field bullets in chapter 七; 5 JSON blocks valid (error/01/02/03/final); 12 headings; line count about 910 (record actual).

- [ ] **Step 5: Commit**

```bash
git add hal-service-api-finder/SKILL.md
git -c user.name=siakam43 -c user.email=429872468@qq.com commit -m "docs(hal-service-api-finder): move snapshot formats to phase chapters"
```

---

## Plan Self-Review

- **Spec coverage:** §3 → Task 1 Step 1; §4 → Task 1 Step 2; §5 → Task 2 Step 1; §6 → Task 2 Step 2; §7 → Task 2 Step 3; §8 verification → both tasks. §2.2 non-changes are untouched.
- **Placeholder scan:** no TBD/TODO; all replacement text and JSON are complete; verification has expected outputs.
- **Consistency:** chapter 四’s new list uses the same six file names as chapter 二’s removed list; the chapter 五 example matches the `scope`/`entry` fields listed in its own step 1; chapter 七 receives the same four field bullets removed from chapter 四.
