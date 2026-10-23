# hal-service-api-finder target_path Interface Tracing Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Only record business functions defined under `<target_path>` as external interfaces, chasing data arguments through out-of-path functions until the target path is reached.

**Architecture:** Add a `<target_path>` variable, extend phase 3 classification with a path check, add a `data_chase` work-item kind, and insert a new 3.9 data-chase procedure. The existing dispatch queue, cycle detection, dedup, and final JSON schema are reused.

**Tech Stack:** Markdown only. Verification is static: grep checks, JSON parsing, heading count, and `wc -l`.

---

## File Structure

- **Modify:** `hal-service-api-finder/SKILL.md` — sections 一, 三, 七, 十, 十一, 十二.
- **Do not modify:** snapshot schemas, error table, final JSON schema, historical plans/specs, `rr1.txt`.

---

### Task 1: Add target_path variable and global documentation

**Files:**
- Modify: `hal-service-api-finder/SKILL.md` (section 一 principle 3; section 三 参数解析; section 七 intro and 3.5; section 十 constraint 4 and terminology; section 十一 anti-rationalization; section 十二 example 1)

- [ ] **Step 1: Replace core principle 3**

Replace:

```text
3. **只识别直接目标。** cmdId 用例分支中直接接收 data 的业务函数就是接口函数；不展开业务函数的内部实现。
```

with:

```text
3. **只识别目标路径内的业务函数。** cmdId 用例分支中直接接收 data 且不构成分发的业务函数若位于 `<target_path>` 下，即为接口；否则沿 data 实参继续追踪到 `<target_path>` 下的业务函数。仍不展开 `<target_path>` 内业务函数的内部实现。
```

- [ ] **Step 2: Add the target_path section to 参数解析**

After the two `service_name` bullets (`service_name` 字符集校验和搜索加引号说明):

```text
- 搜索时按字面量处理并正确加引号（优先 `grep -F`）。
```

insert:

```text
### 目标路径（非命令行参数）

`<target_path>` 默认为 `<project_dir>/vendor/hisi/`；归一化为绝对路径并补尾斜杠，相对路径按 `<project_dir>` 展开，可人工修改。函数定义文件的绝对路径以 `<target_path>` 开头（区分大小写）才算在目标路径下。`<target_path>` 不扩大 scope；目标路径不存在或不在 scope 内时不会有函数命中。
```

- [ ] **Step 3: Replace the chapter 七 intro**

Replace:

```text
阶段三只做一件事：从阶段二的 Dispatch（或 stub 的 OnRemoteRequest）出发，沿全部分发路径追踪 cmdId 与 data 的流向，把每个最外层 cmdId 分支直接消费 data 的业务函数记为接口函数候选，写出 `03_trace.json`。本阶段只追踪到 cmdId 处理位置，不展开接口函数内部实现，不做完整污点传播。
```

with:

```text
阶段三只做一件事：从阶段二的 Dispatch（或 stub 的 OnRemoteRequest）出发，沿全部分发路径追踪 cmdId 与 data 的流向。对不在 `<target_path>` 下的 data 业务函数，沿 data 实参继续追踪；到 `<target_path>` 内接收 data 且不构成分发的业务函数即停止，不展开其内部实现，也不做完整污点传播。最终把接口函数候选写入 `<out>/03_trace.json`。
```

- [ ] **Step 4: Replace constraint 4**

Replace:

```text
4. 只输出直接目标业务函数（含 3.6 的 `*` 无条件/内联处理记录），不展开业务函数内部实现。
```

with:

```text
4. 只输出 `<target_path>` 下接收 data 且不构成分发的业务函数（含 3.6 的 `*` 无条件/内联处理记录）；目标路径外只沿 data 实参继续追踪，不展开目标路径内业务实现。
```

- [ ] **Step 5: Replace the anti-rationalization row**

Replace:

```text
| “业务函数内部还有处理，继续展开” | 只记直接目标，不展开业务实现。 |
```

with:

```text
| “业务函数内部还有处理，继续展开” | `<target_path>` 外只沿 data 实参追踪到目标路径；`<target_path>` 内不展开业务实现。 |
```

- [ ] **Step 6: Update the interface-function terminology row**

Replace:

```text
| 接口函数 | cmdId 用例分支中直接接收客户端 data（或 data 派生值）的业务函数 |
```

with:

```text
| 接口函数 | cmdId 用例分支中直接接收客户端 data（或 data 派生值）且定义在 `<target_path>` 下的业务函数 |
```

- [ ] **Step 7: Update the 3.5 local-tracing bullet**

Replace:

```text
- 只做分类所需的局部追踪，不要求完整污点传播路径；C++ 间接调用、虚函数、函数指针、重载等解析可参考 `taint-path-walker` skill，本 skill 只追踪到 cmdId 处理位置。
```

with:

```text
- 只做分类所需的局部追踪，不要求完整污点传播路径；C++ 间接调用、虚函数、函数指针、重载等解析可参考 `taint-path-walker` skill，本 skill 只追踪到 `<target_path>` 内的业务函数或明确终止条件。
```

- [ ] **Step 8: Add the terminology row**

After:

```text
| scope_roots | clean_code.txt 中原始的文件/文件夹条目清单 |
```

insert:

```text
| target_path | 外部接口落地区域，默认 `<project_dir>/vendor/hisi/`，可人工修改 |
```

- [ ] **Step 9: Update example 1 phase 3**

Replace:

```text
4. 阶段三：以 `DemoDriverDispatch` 初始化内存队列，追踪 `SendRequest` 到 `DemoInterfaceStub::OnRemoteRequest`，枚举 8 个 cmdId 用例并解析宏值；队列为空后写 `<out>/03_trace.json`。
```

with:

```text
4. 阶段三：以 `DemoDriverDispatch` 初始化内存队列，追踪 `SendRequest` 到 `DemoInterfaceStub::OnRemoteRequest`，枚举 8 个 cmdId 用例并解析宏值；对不在 `<target_path>` 下的直接目标继续沿 data 实参追踪，直到 `DemoVendorInterface(data)` 落在 `<target_path>` 下；队列为空后写 `<out>/03_trace.json`。
```

- [ ] **Step 10: Verify Task 1**

```bash
grep -n '<target_path> 默认为' hal-service-api-finder/SKILL.md
grep -n '只识别目标路径内的业务函数' hal-service-api-finder/SKILL.md
grep -n '只输出 `<target_path>` 下接收 data 且不构成分发的业务函数' hal-service-api-finder/SKILL.md
grep -n '| target_path | 外部接口落地区域' hal-service-api-finder/SKILL.md
grep -n 'DemoVendorInterface(data)' hal-service-api-finder/SKILL.md
grep -c '^## ' hal-service-api-finder/SKILL.md
```

Expected: all five new lines present; heading count 12.

- [ ] **Step 11: Commit**

```bash
git add hal-service-api-finder/SKILL.md
git -c user.name=siakam43 -c user.email=429872468@qq.com commit -m "docs(hal-service-api-finder): add target_path variable"
```

---

### Task 2: Extend 3.8, add 3.9 data-chase, renumber 3.10

**Files:**
- Modify: `hal-service-api-finder/SKILL.md` (3.1 and 3.2 section-number references; 3.8; new 3.9; old 3.9 → 3.10)

- [ ] **Step 1: Replace the 3.8 opening and add the two work-item kinds**

Replace:

```text
### 3.8 入队规则

只有 3.6 判定为转发器或内层 dispatcher 的目标才入队。新工作项字段：

- `id` 递增；`dispatcher` 为目标定义处的限定名，`dispatcher_file` / `dispatcher_line` 为目标定义文件与签名行；
- `current_cmd_expr` 记调用点传入的 cmdId 实参表达式；`current_cmd_value` 按 3.4 解析，解析不出时为 null（变量实参本身不产生 `unresolved`，由目标函数内的具体 case 负责解析）；
- `outer_cmd_expr` / `outer_cmd_value` 取调用点的 `effective_outer`；没有外层时为 null；
- `data_vars` 记目标函数中与 data 派生实参对应的形参名（SendRequest 时为 data/parcel 形参）；调用点实参原文写入触发该入队的 case 的 `evidence`；
- `data_derived`：本工作项是否含 data 派生实参（布尔），由调用点实参是否派生决定；
- `parent_case_id` 取当前 case 的 id；无条件转发时为 null；
- `path = 父工作项.path + [目标限定名]`。
```

with:

```text
### 3.8 入队规则

3.6 或 3.9 判定为转发器、内层 dispatcher 或 `data_chase` 的目标才入队。

`kind = "dispatch"` 工作项（转发器/内层 dispatcher）字段：

- `kind = "dispatch"`；
- `id` 递增；`dispatcher` 为目标定义处的限定名，`dispatcher_file` / `dispatcher_line` 为目标定义文件与签名行；
- `current_cmd_expr` 记调用点传入的 cmdId 实参表达式；`current_cmd_value` 按 3.4 解析，解析不出时为 null（变量实参本身不产生 `unresolved`，由目标函数内的具体 case 负责解析）；
- `outer_cmd_expr` / `outer_cmd_value` 取调用点的 `effective_outer`；没有外层时为 null；
- `data_vars` 记目标函数中与 data 派生实参对应的形参名（SendRequest 时为 data/parcel 形参）；调用点实参原文写入触发该入队的 case 的 `evidence`；
- `data_derived`：本工作项是否含 data 派生实参（布尔），由调用点实参是否派生决定；
- `parent_case_id` 取当前 case 的 id；无条件转发时为 null；
- `path = 父工作项.path + [目标限定名]`。

`kind = "data_chase"` 工作项（不在 `<target_path>` 下的 data 业务函数）字段：

- `kind = "data_chase"`；
- `id` 沿用递增计数器；
- `function`：目标定义处限定名；
- `file` / `line`：目标定义文件与签名行；
- `data_vars`：目标函数中与 data 派生实参对应的形参名；
- `effective_outer`：触发追踪的最外层 cmd 表达式与值；
- `origin_case_id`：触发追踪的原 case id，无条件处理时为 null；
- `path = 父工作项.path + [目标限定名]`；
- `evidence_chain`：从原目标到当前函数的调用链文本。
```

- [ ] **Step 2: Add the data_chase dedup key**

Replace:

```text
- 环检测键：`(函数定义文件+行号, current_cmd_expr, current_cmd_value, outer_cmd_expr, outer_cmd_value, sorted(data_vars), data_derived)`。不同 data 上下文（形参集合或派生状态不同）产生不同键，必须分别入队。
```

with:

```text
- `dispatch` 环检测键：`(函数定义文件+行号, current_cmd_expr, current_cmd_value, outer_cmd_expr, outer_cmd_value, sorted(data_vars), data_derived)`。不同 data 上下文（形参集合或派生状态不同）产生不同键，必须分别入队。
- `data_chase` 环检测键：`(kind="data_chase", 定义文件+行号, effective_outer.expr, effective_outer.value, sorted(data_vars), origin_case_id)`。
```

- [ ] **Step 3: Insert the new 3.9 before the old 3.9**

Replace the heading:

```text
### 3.9 终止条件、去重与收尾
```

with the new section followed by the renumbered heading:

```text
### 3.9 target_path 深入追踪

处理 `kind = "data_chase"` 工作项：

1. 按 3.2 的同样方法读取函数完整函数体；定义不在 scope 内或无法唯一确定 → 记 `unresolved`（`kind = "definition"`），结束该工作项。
2. 按 3.5 找出函数体内所有“实参含 data 或 data 派生值”的调用。
3. 对每个调用按 3.7 解析目标定义，然后分类：
   - 定义不在 scope、无法唯一确定或函数体不完整 → 记 `unresolved`（`kind = "definition"`），继续下一个调用；
   - 接收 cmdId + data 且函数体内有分发逻辑 → 按 3.8 入队 `kind = "dispatch"` 工作项；
   - 接收 data 的业务函数，定义在 `<target_path>` 下 → 记为接口（见第 4 条）；
   - 接收 data 的业务函数，定义不在 `<target_path>` 下 → 按 3.8 入队新的 `kind = "data_chase"` 工作项；
   - 排除项 → 忽略，把调用原文与忽略原因追加到 `origin_case_id` 对应 case 的 `evidence`；`origin_case_id` 为 null 时写入当前追踪链；
   - 不接收 data/派生参数 → 不追踪。
4. 命中 `<target_path>` 业务函数时：
   - `interface_function`、`file`、`definition_line` 取该函数；
   - `cmd_id_macro` / `cmd_id_value` 取 `effective_outer`；无外层时 `cmd_id_macro = "*"`、`cmd_id_value = null`；
   - 把 `{name, file, line}` 追加到 `origin_case_id` 对应 case 的 `handlers`；`origin_case_id` 为 null 时不追加；
   - 在对应 case 的 `evidence` 追加追踪链（起点 → ... → 目标(data)）；
   - 按现有 handler 五元组去重。
5. 如果函数体内没有任何 data 实参调用，且函数自身直接处理 data：记 `unresolved`（`kind = "target_path"`），detail 记录起点函数与追踪链。
6. 每条未到达 `<target_path>` 的追踪路径各记一条 `unresolved`（`kind = "target_path"`），detail 包含起点函数、`evidence_chain` 和已尝试的 data 实参调用；已到达的路径正常记接口。
7. 真环 → `unresolved`（`kind = "cycle"`）；队列重复/菱形重复静默去重。
8. 不设深度/扇出上限；同一函数有多个 data 实参调用时全部处理。

### 3.10 终止条件、去重与收尾
```

- [ ] **Step 4: Add data_chase termination and dedup to 3.10, and update cross-references**

Replace the first termination condition:

```text
1. 找到直接业务处理函数（接口函数已写入 `handlers`）；
```

with:

```text
1. 找到 `<target_path>` 下接收 data 且不构成分发的业务函数（接口函数已写入 `handlers`）；
```

After the four termination conditions in 3.10, insert:

```text
`data_chase` 工作项在 3.9 的全部 data 实参目标分类完成后终止。
```

In the dedup rules, after the `visited` bullet, insert:

```text
- `data_chase` 工作项以 `(kind, 定义文件+行号, effective_outer.expr, effective_outer.value, sorted(data_vars), origin_case_id)` 去重。
```

Replace the unresolved kind bullet:

```text
- `unresolved.kind` 取值：`cmd_value` / `target` / `definition` / `data_deferred` / `cycle`。
```

with:

```text
- `unresolved.kind` 取值：`cmd_value` / `target` / `definition` / `data_deferred` / `cycle` / `target_path`。
```

Update the two 3.1/3.2 references to the old finalization section:

```text
5. 若全部条目被跳过且 `on_remote_request` 也为 null：写一条 `unresolved`（`kind = "definition"`，detail 说明没有可用入口），不创建队列项，直接执行 3.9 的收尾。
```

→ end with `直接执行 3.10 的收尾。`

```text
7. 该工作项的全部分支处理完后，从 `queue` 移除该项，把环检测键追加到 `visited`（见 3.9）。
8. 队列仍有项 → 继续下一项；队列为空 → 执行 3.9 收尾。
```

→ replace both `3.9` with `3.10`.

- [ ] **Step 5: Verify Task 2**

```bash
grep -n '### 3.9 target_path 深入追踪\|### 3.10 终止条件、去重与收尾' hal-service-api-finder/SKILL.md
grep -n 'data_chase' hal-service-api-finder/SKILL.md
grep -n 'unresolved.kind` 取值' hal-service-api-finder/SKILL.md
grep -n '见 3.9\|执行 3.9' hal-service-api-finder/SKILL.md || echo 'no old 3.9 references'
grep -c '^## ' hal-service-api-finder/SKILL.md
```

Expected: new 3.9 and 3.10 headings present; `data_chase` appears in 3.8/3.9/3.10; unresolved kind list includes `target_path`; no old 3.9 references; 12 headings.

- [ ] **Step 6: Commit**

```bash
git add hal-service-api-finder/SKILL.md
git -c user.name=siakam43 -c user.email=429872468@qq.com commit -m "docs(hal-service-api-finder): add data-chase queue and section"
```

---

### Task 3: Adjust 3.1/3.2/3.6 classification and finalize

**Files:**
- Modify: `hal-service-api-finder/SKILL.md` (3.1, 3.2, 3.6, chapter 七 收尾)

- [ ] **Step 1: Add kind to 3.1 initial work items**

Replace:

```text
   - `file` 非 null：创建一个工作项，字段为 `id`、`dispatcher`、`dispatcher_file`、`dispatcher_line`、`current_cmd_expr`、`current_cmd_value`、`outer_cmd_expr`、`outer_cmd_value`、`data_vars`、`parent_case_id`、`path`：
```

with:

```text
   - `file` 非 null：创建一个 `kind = "dispatch"` 的工作项，字段为 `id`、`dispatcher`、`dispatcher_file`、`dispatcher_line`、`current_cmd_expr`、`current_cmd_value`、`outer_cmd_expr`、`outer_cmd_value`、`data_vars`、`parent_case_id`、`path`：
```

Replace:

```text
4. 若所有 Dispatch 条目都被跳过（初始队列为空）且 `stub.on_remote_request` 非 null：用该 OnRemoteRequest 创建初始工作项：
   - `dispatcher` / `dispatcher_file` / `dispatcher_line` 取 `on_remote_request` 的值；
```

with:

```text
4. 若所有 Dispatch 条目都被跳过（初始队列为空）且 `stub.on_remote_request` 非 null：用该 OnRemoteRequest 创建初始工作项：
   - `kind = "dispatch"`；
   - `dispatcher` / `dispatcher_file` / `dispatcher_line` 取 `on_remote_request` 的值；
```

- [ ] **Step 2: Branch the 3.2 loop**

Replace:

```text
1. 读取队列中的第一个工作项（不移出队列；处理完在第 7 步移除）。
```

with:

```text
1. 读取队列中的第一个工作项（不移出队列；处理完在第 7 步移除）。若 `kind = "data_chase"`，执行 3.9 后跳到第 7 步；否则继续第 2 步。
```

- [ ] **Step 3: Add the path check to 3.6**

Replace:

```text
4. 接收 data/派生参数且不构成分发的业务函数 → 接口函数候选；
```

with:

```text
4. 接收 data/派生参数且不构成分发的业务函数 → 定义在 `<target_path>` 下记为接口候选，否则入队 `kind = "data_chase"` 工作项；
```

Replace the table row:

```text
| 接收 data/派生参数，且不构成分发 | 记为接口函数候选 |
```

with:

```text
| 接收 data/派生参数，且不构成分发 | 在 `<target_path>` 下记为接口候选；否则入队 `data_chase` |
```

Add after the table:

```text
目标是否在 `<target_path>` 下，按函数定义文件的绝对路径前缀判断（区分大小写）。
```

Replace the multiple-target branch rule:

```text
- 一个分支有多个 data 消费目标 → 多条记录，每个目标各写一条 `handlers`，共用同一 `effective_outer`；同时把 `{name, file, line}` 追加到当前 case 的 `handlers`。
```

with:

```text
- 一个分支有多个 data 消费目标 → 每个目标分别按 `<target_path>` 判断：在目标路径下的各写一条 `handlers`，共用同一 `effective_outer`，并把 `{name, file, line}` 追加到当前 case 的 `handlers`；不在目标路径下的逐个入队 `data_chase`。
```

Replace the inline rule:

```text
- 一个分支只有内联 data 处理、没有调用目标 → 把分发函数自身记为接口函数，`cmd_id_macro` / `cmd_id_value` 取 `effective_outer`；没有外层时 `cmd_id_macro = "*"`、`cmd_id_value = null`。在该 case 的 `evidence` 中记录内联处理代码原文。
```

with:

```text
- 一个分支只有内联 data 处理、没有调用目标 → 分发函数定义在 `<target_path>` 下时，把自身记为接口函数，`cmd_id_macro` / `cmd_id_value` 取 `effective_outer`；没有外层时 `cmd_id_macro = "*"`、`cmd_id_value = null`，并在该 case 的 `evidence` 中记录内联处理代码原文。不在 `<target_path>` 下时，记 `unresolved`（`kind = "target_path"`），detail 记录分发函数与内联处理位置。
```

Replace the unconditional rule:

```text
- 无 cmdId 分支、无条件直接调用 data 消费函数 → 记为 `cmd_id_macro = "*"`、`cmd_id_value = null`，避免漏接口；`evidence` 说明“无条件直接处理”。
```

with:

```text
- 无 cmdId 分支、无条件直接调用 data 消费函数 → 目标在 `<target_path>` 下时记为 `cmd_id_macro = "*"`、`cmd_id_value = null`；否则入队 `data_chase`。两种情况都在 evidence 中说明“无条件直接处理”。
```

- [ ] **Step 4: Update the 03_trace field bullet**

In chapter 七 收尾, replace the `unresolved` field bullet:

```text
- `unresolved` 元素字段：`kind`、`detail`、`tried`；`kind` 取 `cmd_value` / `target` / `definition` / `data_deferred` / `cycle`。
```

with:

```text
- `unresolved` 元素字段：`kind`、`detail`、`tried`；`kind` 取 `cmd_value` / `target` / `definition` / `data_deferred` / `cycle` / `target_path`。
```

- [ ] **Step 5: Verify Task 3**

```bash
sed -n '/^### 3.1/,/^### 3.3/p' hal-service-api-finder/SKILL.md | grep -n 'kind = "dispatch"'
sed -n '/^### 3.2/,/^### 3.3/p' hal-service-api-finder/SKILL.md | grep -n 'kind = "data_chase"'
sed -n '/^### 3.6/,/^### 3.7/p' hal-service-api-finder/SKILL.md | grep -n 'target_path\|data_chase'
grep -n 'kind.*取.*target_path' hal-service-api-finder/SKILL.md
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

Expected: kind fields present in 3.1/3.2; target_path/data_chase in 3.6; target_path in both unresolved kind lists; 5 JSON blocks valid; 12 headings; line count about 1000 (record actual).

- [ ] **Step 6: Commit**

```bash
git add hal-service-api-finder/SKILL.md
git -c user.name=siakam43 -c user.email=429872468@qq.com commit -m "docs(hal-service-api-finder): apply target_path classification"
```

---

## Plan Self-Review

- **Spec coverage:** §3 → Task 1 Step 2; §4 → Task 3 Step 3; §5 → Task 3 Steps 1-2 and Task 2 Steps 1-2; §6 → Task 2 Step 3; §7 → Task 2 Step 3 item 4; §8 → Task 2 Steps 3-4 and Task 3 Steps 3-5; §9 → Task 2 Steps 3-4; §10 → Task 1 Steps 1, 3-6, 8-9 and Task 3 Step 4.
- **Placeholder scan:** no TBD/TODO; every replacement is complete; every verification has expected output.
- **Consistency:** the data-chase machinery (3.8/3.9/3.10) is added in Task 2 before Task 3 starts enqueuing `data_chase` items, so no intermediate commit points at a missing section. `data_chase` field names in 3.8 match those in 3.9; `origin_case_id` is null exactly for unconditional `*`; both unresolved kind lists are updated.
