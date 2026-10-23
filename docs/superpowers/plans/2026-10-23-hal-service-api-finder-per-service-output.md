# hal-service-api-finder Per-Service Output Directory Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Isolate each service's output and backup under its own `<service_name>` directory so multiple services in one project do not overwrite each other.

**Architecture:** Change the single `<out>` definition and the final-output path; all other sections already use `<out>` and follow automatically. Update the example's absolute paths to the per-service directory.

**Tech Stack:** Markdown only. Verification is static: grep checks, JSON parsing, heading count, and `wc -l`.

---

## File Structure

- **Modify:** `hal-service-api-finder/SKILL.md` — sections 二, 三, 十二.
- **Do not modify:** snapshot schemas, error table, final JSON fields, backup/failure rules, analysis behavior.

---

### Task 1: Move output and backup under the service directory

**Files:**
- Modify: `hal-service-api-finder/SKILL.md` (section 二 final output; section 三 output-dir and search definitions; section 十二 example 1)

- [ ] **Step 1: Update the final output path in section 二**

Replace:

```text
最终输出：<project_dir>/.ethunter_out/hal-service-api-finder/<service_name>_api.json
```

with:

```text
最终输出：<project_dir>/.ethunter_out/hal-service-api-finder/<service_name>/<service_name>_api.json
```

- [ ] **Step 2: Update the `<out>` definition in 输出目录初始化**

Replace:

```text
- 输出根目录：`<out>` = `<project_dir>/.ethunter_out/hal-service-api-finder`。
```

with:

```text
- 输出根目录：`<out>` = `<project_dir>/.ethunter_out/hal-service-api-finder/<service_name>`。
```

- [ ] **Step 3: Update the `<out>` definition in 搜索方法与纪律**

Replace:

```text
所有阶段统一使用以下三种方法；后续只写方法名与参数，不再重复命令。其中 `<out>` = `<project_dir>/.ethunter_out/hal-service-api-finder`。
```

with:

```text
所有阶段统一使用以下三种方法；后续只写方法名与参数，不再重复命令。其中 `<out>` = `<project_dir>/.ethunter_out/hal-service-api-finder/<service_name>`。
```

- [ ] **Step 4: Update example 1 paths**

Replace:

```text
1. 初始化：校验参数；若 `/srv/ohos/drivers/.ethunter_out/hal-service-api-finder` 已存在，先删除旧 `<out>_bak`（若存在），再把 `<out>` 备份为 `<out>_bak`，新建空目录；确定分析范围并生成 `<out>/scope_files.txt`（存在 `clean_code.txt` 时另生成 `<out>/scope_roots.txt`）。
```

with:

```text
1. 初始化：校验参数；若 `/srv/ohos/drivers/.ethunter_out/hal-service-api-finder/demo_service` 已存在，先删除旧 `<out>_bak`（若存在），再把 `<out>` 备份为 `<out>_bak`，新建空目录；确定分析范围并生成 `<out>/scope_files.txt`（存在 `clean_code.txt` 时另生成 `<out>/scope_roots.txt`）。
```

Replace:

```text
6. 完成报告：`127 files / 8 cases / 8 interfaces / 0 unresolved`；service_name = `demo_service`，入口变量 = `g_DemoDriverEntry`，Bind = `HdfDemoDriverBind`，Dispatch 数量 = 1；最终文件 `/srv/ohos/drivers/.ethunter_out/hal-service-api-finder/demo_service_api.json`。
```

with:

```text
6. 完成报告：`127 files / 8 cases / 8 interfaces / 0 unresolved`；service_name = `demo_service`，入口变量 = `g_DemoDriverEntry`，Bind = `HdfDemoDriverBind`，Dispatch 数量 = 1；最终文件 `/srv/ohos/drivers/.ethunter_out/hal-service-api-finder/demo_service/demo_service_api.json`。
```

- [ ] **Step 5: Verify Task 1**

```bash
grep -n '最终输出：<project_dir>' hal-service-api-finder/SKILL.md
grep -n '输出根目录：`<out>` = ' hal-service-api-finder/SKILL.md
grep -n '其中 `<out>` = ' hal-service-api-finder/SKILL.md
grep -n 'hal-service-api-finder/<service_name>_api.json' hal-service-api-finder/SKILL.md || echo 'old final path removed'
grep -n 'hal-service-api-finder/demo_service' hal-service-api-finder/SKILL.md
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

Expected: final path includes `/<service_name>/`; both `<out>` definitions include `/<service_name>`; old final path gone; example uses `demo_service/demo_service_api.json`; 5 JSON blocks valid; 12 headings; record actual line count.

- [ ] **Step 6: Commit**

```bash
git add hal-service-api-finder/SKILL.md
git -c user.name=siakam43 -c user.email=429872468@qq.com commit -m "docs(hal-service-api-finder): isolate output by service"
```

---

## Plan Self-Review

- **Spec coverage:** §2 new paths → Task 1 Steps 1-3; §3.1 example → Step 4; §5 verification → Step 5.
- **Placeholder scan:** no TBD/TODO; every replacement is complete; every check has expected output.
- **Consistency:** all other chapters already reference `<out>`, so changing the two definitions and the final absolute path covers the whole skill; the backup remains `<out>_bak`, now per-service.
