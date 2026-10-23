# hal-service-api-finder Per-Service Code File Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the shared `clean_code.txt` with a per-service `<service_name>_code.txt` under `.ethunter_out/`, with no fallback to the old file.

**Architecture:** Rename the scope input and update every reference. Scope determination order and all output paths stay the same; the `clean_code` JSON boolean keeps its name.

**Tech Stack:** Markdown only. Verification is static: grep checks, JSON parsing, heading count, and `wc -l`.

---

## File Structure

- **Modify:** `hal-service-api-finder/SKILL.md` — sections 一, 三, 四, 五, 九, 十, 十二.
- **Do not modify:** output paths, snapshot schemas, error codes, analysis rules, `clean_code` JSON field name.

---

### Task 1: Rename the scope input in section 三

**Files:**
- Modify: `hal-service-api-finder/SKILL.md` (section 三: 分析范围确定, clean_code headings and rules)

- [ ] **Step 1: Update the scope determination steps**

Replace:

```text
1. 先用双方法检查 `<project_dir>/.ethunter_out/clean_code.txt`：存在则按“clean_code.txt 优先，且是硬边界”建立 scope。
2. `clean_code.txt` 不存在时，检查 `<project_dir>/.ethunter_out/.etignore`：存在则全量 find C/C++ 文件并按 `.etignore` 规则排除。
3. `.etignore` 也不存在时，直接全量 find C/C++ 文件。
4. 无论走哪条分支，都必须生成 `<out>/scope_files.txt`；存在 `clean_code.txt` 时还必须生成 `<out>/scope_roots.txt`，否则 `scope_roots` 置为 null。
- 每次执行都重新生成 `<out>/scope_files.txt`；`<out>/scope_roots.txt` 仅在存在 `clean_code.txt` 时生成；不读取旧 scope。
```

with:

```text
1. 先用双方法检查 `<project_dir>/.ethunter_out/<service_name>_code.txt`：存在则按“`<service_name>_code.txt` 优先，且是硬边界”建立 scope。
2. `<service_name>_code.txt` 不存在时，检查 `<project_dir>/.ethunter_out/.etignore`：存在则全量 find C/C++ 文件并按 `.etignore` 规则排除。
3. `.etignore` 也不存在时，直接全量 find C/C++ 文件。
4. 无论走哪条分支，都必须生成 `<out>/scope_files.txt`；存在 `<service_name>_code.txt` 时还必须生成 `<out>/scope_roots.txt`，否则 `scope_roots` 置为 null。
- 每次执行都重新生成 `<out>/scope_files.txt`；`<out>/scope_roots.txt` 仅在存在 `<service_name>_code.txt` 时生成；不读取旧 scope。
```

- [ ] **Step 2: Rename the two section headings**

Replace:

```text
### clean_code.txt 优先，且是硬边界
```

with:

```text
### `<service_name>_code.txt` 优先，且是硬边界
```

Replace:

```text
### clean_code.txt 不存在
```

with:

```text
### `<service_name>_code.txt` 不存在
```

- [ ] **Step 3: Update the hard-boundary section body**

Replace:

```text
双方法检查 <project_dir>/.ethunter_out/clean_code.txt 是否存在。
```

with:

```text
双方法检查 <project_dir>/.ethunter_out/<service_name>_code.txt 是否存在。
```

Replace:

```text
- clean_code.txt 存在时，不得分析或搜索列表以外的源文件。
```

with:

```text
- `<service_name>_code.txt` 存在时，不得分析或搜索列表以外的源文件。
```

Replace:

```text
无 clean_code.txt 时，`scope_roots = null`，不使用 `scope_roots.txt`。
```

with:

```text
无 `<service_name>_code.txt` 时，`scope_roots = null`，不使用 `scope_roots.txt`。
```

- [ ] **Step 4: Verify Task 1**

```bash
sed -n '/^### 分析范围确定/,/^### 搜索方法与纪律/p' hal-service-api-finder/SKILL.md | grep -n 'clean_code.txt' || echo 'section 3 renamed'
grep -n '<service_name>_code.txt' hal-service-api-finder/SKILL.md
grep -c '^## ' hal-service-api-finder/SKILL.md
```

Expected: no old name in section 三; new name in steps, headings, and boundary rules; 12 headings.

- [ ] **Step 5: Commit**

```bash
git add hal-service-api-finder/SKILL.md
git -c user.name=siakam43 -c user.email=429872468@qq.com commit -m "docs(hal-service-api-finder): per-service scope file"
```

---

### Task 2: Update remaining references and verify

**Files:**
- Modify: `hal-service-api-finder/SKILL.md` (principle 2, chapter 四 list, chapter 五 messages/scope, chapter 九 Read row, chapter 十 constraint 3 and term, chapter 十二 examples)

- [ ] **Step 1: Update principle 2 and chapter 四 list**

Replace:

```text
2. **分析范围是硬边界。** 存在 `clean_code.txt` 时，只分析列表（含文件夹展开）内的文件；所有搜索都限制在 scope 内，禁止 `grep -rn <project_dir>`。
```

with:

```text
2. **分析范围是硬边界。** 存在 `<service_name>_code.txt` 时，只分析列表（含文件夹展开）内的文件；所有搜索都限制在 scope 内，禁止 `grep -rn <project_dir>`。
```

Replace:

```text
- scope_roots.txt（仅 clean_code.txt 分支存在）
```

with:

```text
- scope_roots.txt（仅 `<service_name>_code.txt` 分支存在）
```

- [ ] **Step 2: Update chapter 五 messages and scope fields**

Replace:

```text
- `service_not_found`：没有任何合法候选。`message` 必须包含 scope 文件数（`scope_files.txt` 行数）和已搜索模式（至少 `"<service_name>"`），并建议检查 service_name 拼写与 clean_code.txt 覆盖范围；`evidence` 列出被拒命中及其原因；零命中时明确写“0 命中”。
```

with:

```text
- `service_not_found`：没有任何合法候选。`message` 必须包含 scope 文件数（`scope_files.txt` 行数）和已搜索模式（至少 `"<service_name>"`），并建议检查 service_name 拼写与 `<service_name>_code.txt` 覆盖范围；`evidence` 列出被拒命中及其原因；零命中时明确写“0 命中”。
```

Replace:

```text
   - `scope`：`file_count` 为 `<out>/scope_files.txt` 的行数；`clean_code` 为本次是否使用 clean_code.txt（布尔值）；`scope_files` 固定为 `"scope_files.txt"`；`scope_roots` 在使用 clean_code.txt 时为 `"scope_roots.txt"`，否则为 `null`；
```

with:

```text
   - `scope`：`file_count` 为 `<out>/scope_files.txt` 的行数；`clean_code` 为本次是否使用 `<service_name>_code.txt`（布尔值）；`scope_files` 固定为 `"scope_files.txt"`；`scope_roots` 在使用 `<service_name>_code.txt` 时为 `"scope_roots.txt"`，否则为 `null`；
```

- [ ] **Step 3: Update permissions, constraint, and terminology**

Replace:

```text
| `Read` | 读取源码、macro.json、clean_code.txt/.etignore、阶段快照、待回读的临时 JSON |
```

with:

```text
| `Read` | 读取源码、macro.json、`<service_name>_code.txt`/.etignore、阶段快照、待回读的临时 JSON |
```

Replace:

```text
3. clean_code.txt 是硬边界；grep 只用 scope_files/scope_roots；codegraph 结果按 scope 过滤；禁止全仓 grep。
```

with:

```text
3. `<service_name>_code.txt` 是硬边界；grep 只用 scope_files/scope_roots；codegraph 结果按 scope 过滤；禁止全仓 grep。
```

Replace:

```text
| scope_roots | clean_code.txt 中原始的文件/文件夹条目清单 |
```

with:

```text
| scope_roots | `<service_name>_code.txt` 中原始的文件/文件夹条目清单 |
```

- [ ] **Step 4: Update examples 1 and 3**

Replace:

```text
1. 初始化：校验参数；若 `/srv/ohos/drivers/.ethunter_out/hal-service-api-finder/demo_service` 已存在，先删除旧 `<out>_bak`（若存在），再把 `<out>` 备份为 `<out>_bak`，新建空目录；确定分析范围并生成 `<out>/scope_files.txt`（存在 `clean_code.txt` 时另生成 `<out>/scope_roots.txt`）。
```

with:

```text
1. 初始化：校验参数；若 `/srv/ohos/drivers/.ethunter_out/hal-service-api-finder/demo_service` 已存在，先删除旧 `<out>_bak`（若存在），再把 `<out>` 备份为 `<out>_bak`，新建空目录；确定分析范围并生成 `<out>/scope_files.txt`（存在 `demo_service_code.txt` 时另生成 `<out>/scope_roots.txt`）。
```

Replace:

```text
4. 报告：`service_not_found`，分析范围 127 个文件，已搜索模式 `"demo_service_typo"`；建议检查 service_name 拼写（如 `demo_service`），并确认 `clean_code.txt` 覆盖了该驱动的定义文件。
```

with:

```text
4. 报告：`service_not_found`，分析范围 127 个文件，已搜索模式 `"demo_service_typo"`；建议检查 service_name 拼写（如 `demo_service`），并确认 `demo_service_code.txt` 覆盖了该驱动的定义文件。
```

- [ ] **Step 5: Verify Task 2**

```bash
grep -n 'clean_code.txt' hal-service-api-finder/SKILL.md || echo 'old file name removed'
grep -n '<service_name>_code.txt\|demo_service_code.txt' hal-service-api-finder/SKILL.md
grep -n '"clean_code"' hal-service-api-finder/SKILL.md
python3 - <<'PY'
import json, re, pathlib
text = pathlib.Path("hal-service-api-finder/SKILL.md").read_text(encoding="utf-8")
blocks = re.findall(r"^[ \t]*```json\n(.*?)\n[ \t]*```", text, re.S | re.M)
for block in blocks:
    json.loads(block)
print(f"{len(blocks)} JSON blocks valid")
PY
grep -c '^## ' hal-service-api-finder/SKILL.md
wc -l hal-service-api-finder/SKILL.md
```

Expected: no `clean_code.txt`; new file name covers principle, section 三, permissions, constraint, term, examples; `"clean_code"` field remains in the 01_entry example; 5 JSON blocks valid; 12 headings; record actual line count.

- [ ] **Step 6: Commit**

```bash
git add hal-service-api-finder/SKILL.md
git -c user.name=siakam43 -c user.email=429872468@qq.com commit -m "docs(hal-service-api-finder): rename remaining code-file references"
```

---

## Plan Self-Review

- **Spec coverage:** §2 file location and no-fallback → Task 1; §3 all reference categories → Tasks 1-2; §4 examples → Task 2; §6 verification → Task 2.
- **Placeholder scan:** no TBD/TODO; every replacement is complete; every check has expected output.
- **Consistency:** the file path and all prose use `<service_name>_code.txt`; the `clean_code` JSON field name is intentionally unchanged; examples use `demo_service_code.txt`; no fallback to the old file is introduced.
