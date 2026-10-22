# hal-service-api-finder Remove-Recovery Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Remove the task-recovery state machine from `SKILL.md`, replace it with a fresh-output-directory workflow and four simple debug snapshots, and keep the analysis behavior unchanged.

**Architecture:** Each run backs up the existing output directory to `<out>_bak`, creates a new empty `<out>`, analyzes from scratch, writes `01_entry.json` / `02_bind.json` / `03_trace.json` at phase boundaries, and writes `error.json` only on failure. `progress.json`, `entry_state.json`, `trace_state.json`, phase/status transitions, counters, and resume logic are removed.

**Tech Stack:** Markdown only. Verification is static: grep checks, JSON parsing, a shell test of the backup logic, and a before/after line count. No runtime tests.

---

## Testing Strategy

The deliverable is a prompt document. Each task ends with static checks:

1. **Removal checks:** no old state/resume tokens in the edited section.
2. **Snapshot checks:** new file names appear in the correct phase.
3. **Final checks:** no `progress.json` / `entry_state.json` / `trace_state.json` / `tmp/` / `任务恢复` / `断点续跑` / `state_conflict` / `state_io_error` anywhere in `SKILL.md`.
4. **Backup behavior:** run the backup snippet in a temporary directory and verify `_bak` contents and the new empty `<out>`.
5. **Structure checks:** 12 top-level headings; every fenced `json` block parses.

Use this command from the repository root:

```bash
cd /home/admin/cc/wksp/siakam_security_skills/hal_service_api_finder
```

## File Structure

- **Modify:** `hal-service-api-finder/SKILL.md` — replace section 四, simplify state references in sections 二、三、五、六、七、八、九、十、十一、十二.
- **Modify:** `docs/superpowers/plans/2026-10-22-hal-service-api-finder.md` — add a historical note at the top.
- **Modify:** `docs/superpowers/specs/2026-10-22-hal-service-api-finder-design.md` — add a historical note at the top.
- **Create:** `docs/superpowers/plans/2026-10-22-hal-service-api-finder-remove-recovery.md` — this plan.
- **Do not modify:** `rr1.txt`, analysis rules, search methods, final JSON schema.

---

### Task 1: Replace section 四 with the debug-snapshot section

**Files:**
- Modify: `hal-service-api-finder/SKILL.md` (replace `## 四、任务恢复机制` through just before `## 五、阶段一：定位服务注册点`)

- [ ] **Step 1: Replace the entire old section 四**

Replace the current section 四 (from `## 四、任务恢复机制` up to, but not including, `## 五、阶段一：定位服务注册点`) with exactly:

````markdown
## 四、中间结果记录

本 skill 不做任务恢复，也不读取历史结果。每次执行开始时，把旧输出目录整体备份，再新建空目录，从零开始分析。中间结果只用于调试。

### 4.1 输出目录与启动清理

`<out>` = `<project_dir>/.ethunter_out/hal-service-api-finder`
`<out>_bak` = `<project_dir>/.ethunter_out/hal-service-api-finder_bak`

启动时依次执行：

```bash
if [ -d "<out>_bak" ]; then rm -rf "<out>_bak"; fi
if [ -d "<out>" ]; then mv "<out>" "<out>_bak"; fi
mkdir -p "<out>"
```

- 已有 `<out>_bak` → 先删除，只保留最近一次备份。
- 已有 `<out>` → 整体 `mv` 为 `<out>_bak`，旧最终 JSON、阶段快照、scope 文件、error.json 一并保留在备份中。
- 新目录创建后重新生成 `scope_files.txt` / `scope_roots.txt`。
- 不读取旧结果，不判断 project/service 匹配，不做续跑。
- 备份、mv 或 mkdir 失败 → 报 `output_dir_failed`，停止；此时若 `<out>` 无法创建，只在对话中报告，不写 error.json。

输出目录结构：

```text
<out>/
  scope_files.txt
  scope_roots.txt
  01_entry.json
  02_bind.json
  03_trace.json
  <service_name>_api.json
  error.json          # 仅失败时
```

### 4.2 阶段快照

中间快照直接写入 `<out>`，不要求原子写入；写入失败 → 报 `snapshot_write_failed`，停止。最终 `<service_name>_api.json` 仍采用临时文件 + `mv`。

#### 01_entry.json

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

#### 02_bind.json

```json
{
  "service_name": "demo_service",
  "entry": {
    "variable": "g_DemoDriverEntry",
    "file": "/abs/path/...",
    "module_name": "demo_service"
  },
  "bind": {
    "function": "HdfDemoDriverBind",
    "file": "/abs/path/...",
    "line": 72,
    "host_type": "HdfDemoHost",
    "dispatch_functions": [
      {
        "name": "DemoDriverDispatch",
        "file": "/abs/path/...",
        "line": 129,
        "via": "hdfDemoHost->ioService.Dispatch"
      }
    ],
    "stub": {
      "member": "stub",
      "interface_type": "IDemoInterface",
      "impl_class": "DemoService",
      "impl_file": "/abs/path/...",
      "stub_class": "DemoInterfaceStub",
      "on_remote_request": {
        "name": "OHOS::HDI::Demo::V1_0::DemoInterfaceStub::OnRemoteRequest",
        "file": "/abs/path/...",
        "line": 160
      },
      "evidence": "hdfDemoHost->stub = ...GetOrNewObject(serviceImpl, IDemoInterface::GetDescriptor())"
    },
    "service_registration": {
      "file": "/abs/path/...",
      "line": 110,
      "evidence": "deviceObject->service = &hdfDemoHost->ioService"
    }
  }
}
```

#### 03_trace.json

```json
{
  "service_name": "demo_service",
  "entry_dispatchers": [],
  "cases": [],
  "handlers": [],
  "default_cases": [],
  "unresolved": []
}
```

- 队列、`visited`、`dispatch_cache` 只存在于内存，阶段三结束后不写入文件。
- 不写 `status`、`phase`、`counters`、`queue`、`visited`、`dispatch_cache`。
- `cases`、`handlers`、`default_cases`、`unresolved` 的元素结构沿用原定义，但去掉 `status` 等恢复字段。

#### error.json

仅在失败时写入：

```json
{
  "phase": "locate_entry|analyze_bind|trace_dispatch|output",
  "code": "<错误码>",
  "message": "<说明>",
  "evidence": "<证据或已尝试的搜索方式>"
}
```

`phase` 只是失败位置标签，不参与恢复。写入 error.json 后停止；不写最终 JSON。
````

- [ ] **Step 2: Verify Task 1**

```bash
grep -n '^## 四、中间结果记录' hal-service-api-finder/SKILL.md
sed -n '/^## 四、/,/^## 五、/p' hal-service-api-finder/SKILL.md | grep -n 'progress.json\|entry_state.json\|trace_state.json\|恢复流程\|state_conflict\|state_io_error' || echo 'no old state tokens in section 4'
python3 - <<'PY'
import json, re, pathlib
text = pathlib.Path("hal-service-api-finder/SKILL.md").read_text(encoding="utf-8")
section = text.split("## 四、中间结果记录", 1)[1].split("## 五、", 1)[0]
blocks = re.findall(r"```json\n(.*?)\n```", section, re.S)
for block in blocks:
    json.loads(block)
print(f"{len(blocks)} JSON blocks valid in section 4")
PY
```

Expected: heading present, no old state tokens in section 4, 4 JSON blocks valid.

- [ ] **Step 3: Commit**

```bash
git add hal-service-api-finder/SKILL.md
git -c user.name=siakam43 -c user.email=429872468@qq.com commit -m "docs(hal-service-api-finder): replace recovery with debug snapshots"
```

---

### Task 2: Update section 二 and section 三 (output list, scope paths, remove resume rule)

**Files:**
- Modify: `hal-service-api-finder/SKILL.md` (sections 二 and 三)

- [ ] **Step 1: Replace the intermediate-results list in section 二**

Replace:

```text
中间结果：
- progress.json
- tmp/scope_files.txt
- tmp/scope_roots.txt（仅 clean_code.txt 分支存在）
- tmp/entry_state.json
- tmp/trace_state.json
```

with:

```text
中间结果（均位于 <out>）：
- scope_files.txt
- scope_roots.txt（仅 clean_code.txt 分支存在）
- 01_entry.json
- 02_bind.json
- 03_trace.json
- error.json（仅失败时）
```

- [ ] **Step 2: Remove the resume-precedence bullet in section 三**

Delete:

```text
- 若存在与当前 `project_dir`/`service_name` 匹配的 `progress.json`，先执行第四章的恢复检查；只有全新分析才重新生成 scope 和 scope 文件，恢复流程明确要求重建的除外（如 scope_files.txt 缺失）。
```

Replace it with:

```text
- 每次执行都重新生成 `scope_files.txt` / `scope_roots.txt`，不读取旧 scope。
```

- [ ] **Step 3: Update all scope-file paths in section 三**

Apply these literal replacements throughout section 三:

```text
<out>/tmp/scope_files.txt  →  <out>/scope_files.txt
<out>/tmp/scope_roots.txt  →  <out>/scope_roots.txt
tmp/scope_files.txt        →  <out>/scope_files.txt
tmp/scope_roots.txt        →  <out>/scope_roots.txt
```

Then replace the sentence:

```text
无 clean_code.txt 时，`scope_roots = null`；不使用 `scope_roots.txt`。若存在旧的 `scope_roots.txt`，删除或忽略，避免陈旧状态被误用。
```

with:

```text
无 clean_code.txt 时，`scope_roots = null`，不使用 `scope_roots.txt`。
```

- [ ] **Step 4: Verify Task 2**

```bash
sed -n '/^## 二、/,/^## 四、/p' hal-service-api-finder/SKILL.md | grep -n 'tmp/' || echo 'no tmp paths in sections 2-3'
sed -n '/^## 二、/,/^## 四、/p' hal-service-api-finder/SKILL.md | grep -n 'progress.json\|恢复检查' || echo 'no recovery references in sections 2-3'
grep -c '^## ' hal-service-api-finder/SKILL.md
```

Expected: no `tmp/`, no recovery references, heading count 12.

- [ ] **Step 5: Commit**

```bash
git add hal-service-api-finder/SKILL.md
git -c user.name=siakam43 -c user.email=429872468@qq.com commit -m "docs(hal-service-api-finder): update output list and scope paths"
```

---

### Task 3: Update section 五 (Phase 1) to write 01_entry.json

**Files:**
- Modify: `hal-service-api-finder/SKILL.md` (section 五)

- [ ] **Step 1: Replace the phase intro**

Replace:

```text
阶段一只做一件事：在 scope 内找到 `.moduleName` 与 `<service_name>` 精确相等的 `HdfDriverEntry` 初始化器，确认注册形式，写出 `entry_state.json`。本阶段不分析 Bind 内部逻辑，不进入 Dispatch。
```

with:

```text
阶段一只做一件事：在 scope 内找到 `.moduleName` 与 `<service_name>` 精确相等的 `HdfDriverEntry` 初始化器，确认注册形式，写出 `01_entry.json`。本阶段不分析 Bind 内部逻辑，不进入 Dispatch。
```

- [ ] **Step 2: Replace the multi-hit error and failure paragraphs**

In the multi-hit rule, replace:

```text
写入 `progress.error` 并停止
```

with:

```text
写 `error.json` 并停止
```

Replace the failure paragraph:

```text
失败一律按第四章写入 `progress.error = {code, message, evidence}`，保留 `phase = locate_entry`，不写 `entry_state.json`，不进入阶段二，停止。
```

with:

```text
失败一律写 `error.json = {phase: "locate_entry", code, message, evidence}`，不写 `01_entry.json`，不进入阶段二，停止。
```

In the `bind_not_found` bullet, replace:

```text
该检查在多命中消歧之后执行；失败时不写 `entry_state.json`，下次运行因无有效状态会重跑阶段一并重新校验。
```

with:

```text
该检查在多命中消歧之后执行；失败时不写 `01_entry.json`，停止。
```

- [ ] **Step 3: Replace the phase-output steps**

Replace the old `### 阶段产物` content (from `定位成功后：` through the `progress.json` update) with:

````markdown
### 阶段产物

定位成功后：

1. 组装 `01_entry.json`：
   - `service_name`：当前 `<service_name>`；
   - `scope`：`file_count`、`clean_code`、`scope_files`、`scope_roots`；
   - `entry`：本阶段提取的 `variable`、`file`、`line`、`module_name`、`bind`、`init`、`release`、`registration`。
2. 直接写 `<out>/01_entry.json`；写入失败 → 写 `error.json`（`phase = "locate_entry"`，`code = "snapshot_write_failed"`），停止。
3. 进入 `## 六、阶段二：分析 Bind`。
````

- [ ] **Step 4: Verify Task 3**

```bash
sed -n '/^## 五、/,/^## 六、/p' hal-service-api-finder/SKILL.md | grep -n 'progress\|entry_state\|phase\b' || echo 'no old state references in section 5'
grep -n '01_entry.json' hal-service-api-finder/SKILL.md
grep -n 'error.json' hal-service-api-finder/SKILL.md
```

Expected: no old state references in section 5; `01_entry.json` and `error.json` present.

- [ ] **Step 5: Commit**

```bash
git add hal-service-api-finder/SKILL.md
git -c user.name=siakam43 -c user.email=429872468@qq.com commit -m "docs(hal-service-api-finder): write phase 1 debug snapshot"
```

---

### Task 4: Update section 六 (Phase 2) to write 02_bind.json

**Files:**
- Modify: `hal-service-api-finder/SKILL.md` (section 六)

- [ ] **Step 1: Update the phase intro and input**

Replace:

```text
提取 Host 类型、全部 Dispatch 函数、可选的 stub/接口/实现类信息与服务注册证据，写出 `entry_state.bind` 后进入阶段三。
```

with:

```text
提取 Host 类型、全部 Dispatch 函数、可选的 stub/接口/实现类信息与服务注册证据，写出 `02_bind.json` 后进入阶段三。
```

Replace:

```text
输入：`entry_state.entry.bind`、`entry.file`（阶段一已确认 `.Bind` 非空）。
```

with:

```text
输入：`01_entry.json` 的 `entry.bind`、`entry.file`（阶段一已确认 `.Bind` 非空）。
```

- [ ] **Step 2: Update the null-Dispatch reference**

Replace:

```text
阶段三初始化队列时跳过 file 为 null 的 Dispatch 条目并记入 `trace_state.unresolved`（`kind = "definition"`）
```

with:

```text
阶段三初始化队列时跳过 file 为 null 的 Dispatch 条目并记入 `03_trace.json` 的 `unresolved`（`kind = "definition"`）
```

- [ ] **Step 3: Replace the phase-output block**

Replace the old `### 阶段产物` content (from `提取成功后，按第四章 ...` through the `progress.json` update) with:

````markdown
### 阶段产物

提取成功后，组装并写 `02_bind.json`：

```json
{
  "service_name": "demo_service",
  "entry": {
    "variable": "g_DemoDriverEntry",
    "file": "/abs/path/...",
    "module_name": "demo_service"
  },
  "bind": {
    "function": "HdfDemoDriverBind",
    "file": "/abs/path/...",
    "line": 72,
    "host_type": "HdfDemoHost",
    "dispatch_functions": [
      {"name": "DemoDriverDispatch", "file": "/abs/path/...", "line": 129, "via": "hdfDemoHost->ioService.Dispatch"}
    ],
    "stub": {
      "member": "stub",
      "interface_type": "IDemoInterface",
      "impl_class": "DemoService",
      "impl_file": "/abs/path/...",
      "stub_class": "DemoInterfaceStub",
      "on_remote_request": {"name": "OHOS::HDI::Demo::V1_0::DemoInterfaceStub::OnRemoteRequest", "file": "/abs/path/...", "line": 160},
      "evidence": "hdfDemoHost->stub = ...GetOrNewObject(serviceImpl, IDemoInterface::GetDescriptor())"
    },
    "service_registration": {"file": "/abs/path/...", "line": 110, "evidence": "deviceObject->service = &hdfDemoHost->ioService"}
  }
}
```

- `bind` 对象字段沿用原定义（`function` / `file` / `line` / `host_type` / `dispatch_functions` / `stub` / `service_registration`），不写 `status`。
- 直接写 `<out>/02_bind.json`；写入失败 → 写 `error.json`（`phase = "analyze_bind"`，`code = "snapshot_write_failed"`），停止。
- 进入 `## 七、阶段三：cmdId 分发追踪`。
````

- [ ] **Step 4: Replace the failure paragraph**

Replace:

```text
- 任一失败码一律按第四章写 `progress.error = {code, message, evidence}`，保持 `phase = analyze_bind`，不写 `bind.status = completed`，不进入阶段三；下次运行从本阶段开头重跑（本阶段幂等）。
```

with:

```text
- 任一失败码一律写 `error.json`（`phase = "analyze_bind"`），不写 `02_bind.json`，不进入阶段三，停止。
```

- [ ] **Step 5: Verify Task 4**

```bash
sed -n '/^## 六、/,/^## 七、/p' hal-service-api-finder/SKILL.md | grep -n 'progress\|entry_state\|trace_state\|bind.status\|phase = analyze_bind' || echo 'no old state references in section 6'
grep -n '02_bind.json' hal-service-api-finder/SKILL.md
```

Expected: no old state references except the new `error.json` phase label; `02_bind.json` present.

- [ ] **Step 6: Commit**

```bash
git add hal-service-api-finder/SKILL.md
git -c user.name=siakam43 -c user.email=429872468@qq.com commit -m "docs(hal-service-api-finder): write phase 2 debug snapshot"
```

---

### Task 5: Update section 七 (Phase 3) to in-memory tracing + 03_trace.json

**Files:**
- Modify: `hal-service-api-finder/SKILL.md` (section 七)

- [ ] **Step 1: Update the phase intro and input**

Replace `写入 `trace_state.json`` with `写出 `03_trace.json`` in the section 七 intro.

Replace:

```text
输入取自 `entry_state.json` 的 `bind`：
```

with:

```text
输入取自 `02_bind.json` 的 `bind`：
```

- [ ] **Step 2: Simplify 3.1 initialization**

Replace `按第四章的 `trace_state.json` 字段形状初始化：` with `在内存中初始化：`.

Remove the line:

```text
1. `status = in_progress`，`service_name` 取当前服务名。
```

Renumber the remaining list. Replace:

```text
3. `queue`、`cases`、`handlers`、`default_cases`、`unresolved`、`dispatch_cache`、`visited` 初始化为空数组；`counters` 按实际数组长度重算。
```

with:

```text
2. `queue`、`cases`、`handlers`、`default_cases`、`unresolved`、`dispatch_cache`、`visited` 初始化为空数组。
```

Remove all `status = pending` text from work-item creation, and delete the `status` field from the work-item field list. The queue is a plain FIFO; no item status is persisted.

- [ ] **Step 3: Remove per-work-item state saves in 3.2**

Replace:

```text
1. 取出第一个 `pending` 工作项，置为 `in_progress`，保存 `trace_state`。
```

with:

```text
1. 取出队列中的第一个工作项。
```

Replace:

```text
7. 该工作项的全部分支处理完后，从 `queue` 移除该项，把环检测键追加到 `visited`，重算 counters，保存 `trace_state`（见 3.9）。
```

with:

```text
7. 该工作项的全部分支处理完后，从 `queue` 移除该项，把环检测键追加到 `visited`（见 3.9）。
```

- [ ] **Step 4: Remove case status fields**

In 3.3, remove `status` from the `cases` field list and delete:

```text
- `status` 先记 `pending`，该分支完整分析后改为 `analyzed`；
```

Replace the `cases` field-list sentence with:

```text
每个识别出的分支先写入 `cases`（字段：`id`、`dispatcher`、`dispatcher_file`、`dispatcher_line`、`dispatch_site`、`level`、`case_expr`、`case_value`、`outer_cmd_expr`、`outer_cmd_value`、`parent_case_id`、`handlers`、`evidence`）：
```

- [ ] **Step 5: Remove work-item status and use in-memory cache**

In 3.8, remove `status = pending` from the work-item field list. In 3.7, add “仅在内存中” to the `dispatch_cache` bullet:

```text
- 每次解析调用目标后写/更新内存中的 `dispatch_cache`（`site`、`target`、`candidates`、`resolved`）；同一 site+target 复用缓存结果，不重复搜索。
```

- [ ] **Step 6: Replace 3.9 state saving and resume rules**

Replace the paragraph beginning `处理期间 `trace_state.status` ...` and the entire `去重与幂等规则` block through `unresolved.kind` with:

```text
追踪期间不写状态文件。队列为空后，一次性写 `03_trace.json`。

单次运行内的去重规则：

- case 以 `(dispatcher_file, dispatcher_line, dispatch_site.file, dispatch_site.line, case_expr, outer_cmd_expr, outer_cmd_value)` 去重；重复命中时复用已有 case id，合并 evidence 与 handlers。
- handler 以 `(cmd_id_macro, cmd_id_value, interface_function, file, definition_line)` 去重；同一业务函数在不同 cmdId 下分别保留多条记录。
- `default_cases` 以 `(dispatcher_file, dispatcher_line, dispatch_site.file, dispatch_site.line)` 去重。
- `unresolved` 以 `(kind, detail)` 去重，`tried` 合并后去重。
- `dispatch_cache` 以 `(site.file, site.line, target)` 去重，条目包含 `site`、`target`、`candidates`、`resolved`。
- `visited` 条目记录函数定义信息与 cmd 上下文；以环检测键去重。
- `unresolved.kind` 取值：`cmd_value` / `target` / `definition` / `data_deferred` / `cycle`。
```

Replace the `收尾与状态切换` block with:

````text
收尾：

- 队列为空后，写 `<out>/03_trace.json`：

```json
{
  "service_name": "<service_name>",
  "entry_dispatchers": [],
  "cases": [],
  "handlers": [],
  "default_cases": [],
  "unresolved": []
}
```

- 写入失败 → 写 `error.json`（`phase = "trace_dispatch"`，`code = "snapshot_write_failed"`），停止。
- 写入成功后进入 `## 八、输出与自检`。
- 全局失败（工具/文件 I/O 或无法继续读取源码等不可恢复错误）→ 若已有部分追踪结果，先写 `03_trace.json` 的部分快照，再写 `error.json`（`phase = "trace_dispatch"`），停止；单个分支的解析失败只写 `unresolved`，不中止整体追踪。
````

- [ ] **Step 7: Verify Task 5**

```bash
sed -n '/^## 七、/,/^## 八、/p' hal-service-api-finder/SKILL.md | grep -n 'trace_state\|progress\|counters\|queue_remaining\|status = pending\|status = in_progress\|status = analyzed' || echo 'no old state references in section 7'
grep -n '03_trace.json' hal-service-api-finder/SKILL.md
```

Expected: no old state references in section 7; `03_trace.json` present.

- [ ] **Step 8: Commit**

```bash
git add hal-service-api-finder/SKILL.md
git -c user.name=siakam43 -c user.email=429872468@qq.com commit -m "docs(hal-service-api-finder): simplify phase 3 tracing state"
```

---

### Task 6: Update section 八 (output and self-check)

**Files:**
- Modify: `hal-service-api-finder/SKILL.md` (section 八)

- [ ] **Step 1: Replace trace_state references in 8.1**

Replace `trace_state.handlers` with `03_trace.json` 的 `handlers` everywhere in section 8.1 (intro, field notes, assembly step 1).

Replace the sentence:

```text
5. 输出规则中的 `tmp/...` 以 `<out>` 为基准，即临时文件为 `<out>/tmp/<service_name>_api.json.tmp`，最终文件为 `<out>/<service_name>_api.json`。
```

with:

```text
5. 临时文件为 `<out>/<service_name>_api.json.tmp`，最终文件为 `<out>/<service_name>_api.json`。
```

Then replace any remaining `tmp/<service_name>_api.json.tmp` with `<out>/<service_name>_api.json.tmp` (for output rule 8 and any other occurrence).

- [ ] **Step 2: Remove status/phase transitions from 8.2**

Replace the intro paragraph:

```text
自检开始前将 `trace_state.status` 置为 `self_check`（阶段三收尾应已置为该值；从恢复流程进入时先确认，按以下映射纠正：`in_progress` → 回到 `trace_dispatch` 继续；`completed` → 跳过自检，直接进入输出组装；`self_check` → 执行清单），先以 `<out>/tmp/trace_state.json.tmp` + `mv` 原子保存，再执行以下清单：
```

with:

```text
自检直接基于内存中的追踪结果和 `03_trace.json`；不再有 `trace_state.status` 或 phase 转换。执行以下清单：
```

- [ ] **Step 3: Update the self-check checklist**

Replace the checklist with:

```text
1. 队列已处理完（所有工作项已从内存队列移除）。
2. 每个 `handlers` 条目都能在 scope_files 内定位到函数定义，且函数名为限定名。
3. `default_cases` 没有混入 `handlers`。
4. handlers 已按四元组去重，排序规则已执行。
5. `unresolved` 已记录，数量与报告一致。
6. 最终 JSON 恰好包含四个字段，无额外字段。
7. 写临时文件后 Read 回读，核对数组长度、字段名、类型。
8. `mv` 后双方法确认最终文件存在。
```

Update the execution notes to exactly:

```text
执行说明：

- 第 1-5 项检查内存中的追踪结果和 `03_trace.json`；第 6 项针对 8.1 组装后的最终数组核对。
- 第 2 项中 lambda 的 `interface_function` 以 `<lambda@绝对路径:行号>` 形式为合规；其余函数必须是定义处作用域限定名。每个条目都要按 `file` 与定义位置 Read 确认完整函数体，且 `file` 必须在 `scope_files.txt` 内。
- 第 3 项检查没有任何 handler 的 `cmd_id_macro == "default"`，且不存在仅由 `default_cases` 站点支撑的 handler。
- 第 5 项 unresolved 数量以 `03_trace.json` 的 `unresolved` 长度为准，并与完成报告一致。
- 第 7 项回读后核对：数组长度等于去重后的记录数；每条键集合恰为四字段；`cmd_id_macro` / `interface_function` / `file` 为字符串，`cmd_id_value` 为整数或 null。必须读到文件末尾；截断视为回读失败。
- 第 8 项双方法确认最终文件存在（`ls` + `Read`），禁止 `test -f` / `[ -f ]`。
```

Replace the `执行顺序与状态切换` block with:

```text
执行顺序：

1. 执行清单第 1-5 项。
2. 按 8.1 组装最终数组，执行清单第 6 项。
3. 写 `<out>/<service_name>_api.json.tmp`，执行清单第 7 项。
4. `mv` 为最终文件，执行清单第 8 项。
5. 按 8.3 输出完成报告。
```

- [ ] **Step 4: Replace failure handling**

Replace:

```text
- 清单第 1-7 项任一失败 → 按第四章写 `progress.error`（`code = "self_check_failed"`，message/evidence 记录失败项编号与证据），`trace_state.status` 保持 `self_check`，`progress.phase` 保持 `trace_dispatch`（若已进入 `output` 则保持 `output`），停止；不得写最终 JSON。
- 清单第 8-9 项（写临时文件、回读校验、`mv`、最终文件存在性确认）任一失败 → 写 `progress.error`（`code = "output_write_failed"`，evidence 记录临时/最终路径与失败动作），保持 `phase = output` 以便重试，停止；不得把 `phase` 置为 `done`。
- 重试 `output_write_failed` 时从 `trace_state.handlers` 重新组装（结果幂等），覆盖临时文件后重新执行回读、`mv` 与存在性确认。
- 最终文件必须存在；无法确认存在即按 `output_write_failed` 处理，本次任务失败。
```

with:

```text
- 自检任一失败 → 写 `error.json`（`phase = "output"`，`code = "self_check_failed"`，message/evidence 记录失败项编号与证据），停止；不得写最终 JSON。
- 写临时文件、回读校验、`mv` 或最终文件存在性确认失败 → 写 `error.json`（`phase = "output"`，`code = "output_write_failed"`，evidence 记录临时/最终路径与失败动作），停止。
- 最终文件必须存在；无法确认存在即按 `output_write_failed` 处理，本次任务失败。
```

- [ ] **Step 5: Update 8.3 counts**

Replace `trace_state.cases.length` with `03_trace.json` 的 `cases` 长度，and `trace_state.unresolved` with `03_trace.json` 的 `unresolved`.

- [ ] **Step 6: Verify Task 6**

```bash
sed -n '/^## 八、/,/^## 九、/p' hal-service-api-finder/SKILL.md | grep -n 'trace_state\|progress\|phase\|status' || echo 'no old state references in section 8'
sed -n '/^## 八、/,/^## 九、/p' hal-service-api-finder/SKILL.md | grep -n 'tmp/' || echo 'no tmp paths in section 8'
grep -n '03_trace.json' hal-service-api-finder/SKILL.md
```

Expected: no old state references or `tmp/` paths in section 8; `03_trace.json` present.

- [ ] **Step 7: Commit**

```bash
git add hal-service-api-finder/SKILL.md
git -c user.name=siakam43 -c user.email=429872468@qq.com commit -m "docs(hal-service-api-finder): simplify output and self-check"
```

---

### Task 7: Update sections 九、十、十一、十二

**Files:**
- Modify: `hal-service-api-finder/SKILL.md` (sections 九-十二)

- [ ] **Step 1: Update permissions (section 九)**

Replace the `Bash` / `Read` / `Write` rows with:

```text
| `Bash` | find/grep/xargs/ls/mkdir/mv/rm，scope 内搜索、备份/清理输出目录 |
| `Read` | 读取源码、macro.json、阶段快照 |
| `Write` | 写入 01_entry.json、02_bind.json、03_trace.json、error.json、最终 JSON |
```

- [ ] **Step 2: Update constraints and error table (section 十)**

Replace constraint 8:

```text
8. 每个工作项处理完立即落盘；最终 JSON 最后一次性原子写出。
```

with:

```text
8. 每个阶段结束写一次快照；最终 JSON 最后一次性原子写出。
```

Remove the `state_conflict` and `state_io_error` rows from the error table. Update `output_dir_failed` to:

```text
| `output_dir_failed` | 备份旧输出目录或创建新输出目录失败 | 报告，结束 |
```

Add a row:

```text
| `snapshot_write_failed` | 01_entry.json / 02_bind.json / 03_trace.json 写入失败 | 报告，结束 |
```

Update `output_write_failed` to:

```text
| `output_write_failed` | 最终 JSON 写入、回读或 mv 失败 | 报告，结束 |
```

Update `self_check_failed` to:

```text
| `self_check_failed` | 自检清单失败 | 报告，结束，不写最终 JSON |
```

Update the terminology `中间结果` row to:

```text
| 中间结果 | `scope_files.txt`、`scope_roots.txt`、`01_entry.json`、`02_bind.json`、`03_trace.json`、`error.json` 等调试文件 |
```

- [ ] **Step 3: Update anti-rationalization (section 十一)**

Replace:

```text
| “状态最后一起存” | 每个工作项处理完立即落盘。 |
| “有中间结果也从头跑一遍” | 先走恢复检查，按 phase 续跑。 |
```

with:

```text
| “阶段结果最后一起写” | 每个阶段结束写一次快照；最终 JSON 最后写出。 |
| “想复用上次结果” | 每次执行先备份旧输出目录，再从零分析，不复用历史结果。 |
```

- [ ] **Step 4: Rewrite the examples (section 十二)**

Replace the entire content of section 十二 with:

````markdown
## 十二、使用示例

### 示例一：首次分析

```text
/hal-service-api-finder /srv/ohos/drivers demo_service
```

1. 初始化：校验参数；若 `/srv/ohos/drivers/.ethunter_out/hal-service-api-finder` 已存在，先备份为 `..._bak`，再新建空目录；确定分析范围并生成 `scope_files.txt`（存在 `clean_code.txt` 时另生成 `scope_roots.txt`）。
2. 阶段一：在 scope 内搜索 `"demo_service"`，定位 `struct HdfDriverEntry g_DemoDriverEntry = {...}` 与 `HDF_INIT(g_DemoDriverEntry)`；写 `<out>/01_entry.json`。
3. 阶段二：定位 `HdfDemoDriverBind` 定义，识别 Host 类型与 `ioService.Dispatch = DemoDriverDispatch`，提取 `IDemoInterface`、`DemoService`、`DemoInterfaceStub` 及其 `OnRemoteRequest`；写 `<out>/02_bind.json`。
4. 阶段三：以 `DemoDriverDispatch` 初始化内存队列，追踪 `SendRequest` 到 `DemoInterfaceStub::OnRemoteRequest`，枚举 8 个 cmdId 用例并解析宏值；队列为空后写 `<out>/03_trace.json`。
5. 输出：自检后组装最终数组，写 `<out>/demo_service_api.json.tmp`，回读校验后 `mv` 为 `<out>/demo_service_api.json`，双方法确认存在。
6. 完成报告：`127 files / 8 cases / 8 interfaces / 0 unresolved`；service_name = `demo_service`，入口变量 = `g_DemoDriverEntry`，Bind = `HdfDemoDriverBind`，Dispatch 数量 = 1；最终文件 `/srv/ohos/drivers/.ethunter_out/hal-service-api-finder/demo_service_api.json`。

### 示例二：重复执行（备份旧结果）

```text
/hal-service-api-finder /srv/ohos/drivers demo_service
```

1. 检测到 `<out>` 已存在；若 `<out>_bak` 也存在，先删除旧 `_bak`。
2. 将 `<out>` 整体 `mv` 为 `<out>_bak`，新建空的 `<out>`。
3. 重新生成 scope 文件，按阶段一、二、三从零分析；不读取 `_bak` 中的任何内容。
4. 本次结果写入新的 `<out>`；旧结果保留在 `<out>_bak` 供人工查看。

### 示例三：service 未找到

```text
/hal-service-api-finder /srv/ohos/drivers demo_service_typo
```

1. 初始化与 scope 生成正常，`scope_files.txt` 共 127 个文件。
2. 阶段一按「精确字符串搜索」查找 `"demo_service_typo"`，0 命中。
3. 写 `<out>/error.json = {phase: "locate_entry", code: "service_not_found", message: ..., evidence: ...}`：`message` 记录 scope 文件数 127 与搜索模式 `"demo_service_typo"`，`evidence` 明确写“0 命中”；停止，不写 `01_entry.json`，不进入阶段二。
4. 报告：`service_not_found`，分析范围 127 个文件，已搜索模式 `"demo_service_typo"`；建议检查 service_name 拼写（如 `demo_service`），并确认 `clean_code.txt` 覆盖了该驱动的定义文件。
````

- [ ] **Step 5: Verify Task 7**

```bash
sed -n '/^## 九、/,$p' hal-service-api-finder/SKILL.md | grep -n 'progress\|entry_state\|trace_state\|state_conflict\|state_io_error\|断点续跑\|恢复' || echo 'no old state references in sections 9-12'
grep -n 'snapshot_write_failed\|01_entry.json\|02_bind.json\|03_trace.json' hal-service-api-finder/SKILL.md | tail -20
grep -c '^## ' hal-service-api-finder/SKILL.md
```

Expected: no old state references in sections 9-12; new snapshot names present; heading count 12.

- [ ] **Step 6: Commit**

```bash
git add hal-service-api-finder/SKILL.md
git -c user.name=siakam43 -c user.email=429872468@qq.com commit -m "docs(hal-service-api-finder): update constraints, errors, and examples"
```

---

### Task 8: Historical notes, final checks, and push

**Files:**
- Modify: `docs/superpowers/plans/2026-10-22-hal-service-api-finder.md`
- Modify: `docs/superpowers/specs/2026-10-22-hal-service-api-finder-design.md`

- [ ] **Step 1: Add historical notes**

At the top of the old plan (after its header block) and the old design spec (after its metadata list), insert:

```text
> 注：任务恢复机制已在后续重构中移除；本文中的 progress/state/resume 内容仅作历史记录。
```

- [ ] **Step 2: Run final removal and structure checks**

```bash
grep -n -E 'progress\.json|entry_state\.json|trace_state\.json|任务恢复|恢复流程|断点续跑|state_conflict|state_io_error|counters' hal-service-api-finder/SKILL.md || echo 'no old recovery tokens'
grep -n 'tmp/' hal-service-api-finder/SKILL.md || echo 'no tmp paths'
grep -c '^## ' hal-service-api-finder/SKILL.md
python3 - <<'PY'
import json, re, pathlib
text = pathlib.Path("hal-service-api-finder/SKILL.md").read_text(encoding="utf-8")
blocks = re.findall(r"```json\n(.*?)\n```", text, re.S)
for block in blocks:
    json.loads(block)
print(f"{len(blocks)} JSON blocks valid")
PY
wc -l hal-service-api-finder/SKILL.md
```

Expected:
- no old recovery tokens;
- no `tmp/` paths;
- heading count 12;
- all JSON blocks valid;
- line count below 1000 (record actual).

- [ ] **Step 3: Test the backup snippet in a temporary directory**

```bash
tmp=$(mktemp -d)
out="$tmp/out"; bak="${out}_bak"
mkdir -p "$out"; echo old > "$out/old.txt"
if [ -d "$bak" ]; then rm -rf "$bak"; fi
if [ -d "$out" ]; then mv "$out" "$bak"; fi
mkdir -p "$out"
test -f "$bak/old.txt" && test -z "$(ls -A "$out")" && echo 'first backup: OK'
echo older > "$bak/older.txt"
mkdir -p "$out"; echo new > "$out/new.txt"
if [ -d "$bak" ]; then rm -rf "$bak"; fi
if [ -d "$out" ]; then mv "$out" "$bak"; fi
mkdir -p "$out"
test ! -f "$bak/older.txt" && test -f "$bak/new.txt" && test -z "$(ls -A "$out")" && echo 'replacement backup: OK'
rm -rf "$tmp"
```

Expected: `first backup: OK` and `replacement backup: OK`.

- [ ] **Step 4: Commit and push**

```bash
git add hal-service-api-finder/SKILL.md docs/superpowers/plans/2026-10-22-hal-service-api-finder.md docs/superpowers/specs/2026-10-22-hal-service-api-finder-design.md
git -c user.name=siakam43 -c user.email=429872468@qq.com commit -m "refactor(hal-service-api-finder): remove task recovery mechanism"
git push origin main
git log -1 --oneline
git status --short --branch
```

Expected: commit and push succeed; working tree clean.

---

## Plan Self-Review

- **Spec coverage:** Task 1 covers the new output/backup/snapshot design; Task 2 covers output list and scope paths; Task 3 covers phase 1; Task 4 covers phase 2; Task 5 covers phase 3; Task 6 covers output/self-check; Task 7 covers permissions, constraints, errors, anti-rationalization, and examples; Task 8 covers notes, validation, and push.
- **Placeholder scan:** No TBD/TODO/fill-in instructions. Every edit has an exact old/new anchor; verification has expected output.
- **Type consistency:** Snapshot names are consistent: `01_entry.json`, `02_bind.json`, `03_trace.json`, `error.json`. Backup names are consistent: `<out>_bak`. No `progress.json` / `entry_state.json` / `trace_state.json` remains after Task 8.
