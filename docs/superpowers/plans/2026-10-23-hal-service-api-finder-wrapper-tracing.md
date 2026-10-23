# hal-service-api-finder Wrapper Tracing Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Skip obvious single-line data wrappers under `<target_path>` and record the first function that truly processes data, falling back to the deepest wrapper when none is found.

**Architecture:** Add a conservative wrapper check to 3.6, carry `fallback_handler` in `data_chase` items, and let 3.9 continue through wrappers or record processors/fallbacks. Dispatchers and the final JSON schema are unchanged.

**Tech Stack:** Markdown only. Verification is static: grep checks, JSON parsing, heading count, and `wc -l`.

---

## File Structure

- **Modify:** `hal-service-api-finder/SKILL.md` — sections 一, 七, 十, 十二.
- **Do not modify:** snapshot schemas, error table, final JSON schema, target_path definition, historical plans/specs, `rr1.txt`.

---

### Task 1: Update global wording, terminology, and example

**Files:**
- Modify: `hal-service-api-finder/SKILL.md` (section 一 principle 3; section 七 intro; section 十 constraint 4 and terminology; section 十二 example 1)

- [ ] **Step 1: Replace core principle 3**

Replace:

```text
3. **只识别目标路径内的业务函数。** cmdId 用例分支中直接接收 data 且不构成分发的业务函数若位于 `<target_path>` 下，即为接口；否则沿 data 实参继续追踪到 `<target_path>` 下的业务函数。仍不展开 `<target_path>` 内业务函数的内部实现。
```

with:

```text
3. **只识别目标路径内真正处理 data 的业务函数。** cmdId 用例分支中直接接收 data 且不构成分发的业务函数若位于 `<target_path>` 下，先判断是否为明显封装：是则沿唯一 data 下游继续，直到真正处理 data 的函数；否则当前函数即接口。找不到真正处理函数时，保留最深的 `<target_path>` 封装函数。仍不展开最终接口的内部实现。
```

- [ ] **Step 2: Replace the chapter 七 intro**

Replace:

```text
阶段三只做一件事：从阶段二的 Dispatch（或 stub 的 OnRemoteRequest）出发，沿全部分发路径追踪 cmdId 与 data 的流向。对不在 `<target_path>` 下的 data 业务函数，沿 data 实参继续追踪；到 `<target_path>` 内接收 data 且不构成分发的业务函数即停止，不展开其内部实现，也不做完整污点传播。最终把接口函数候选写入 `<out>/03_trace.json`。
```

with:

```text
阶段三只做一件事：从阶段二的 Dispatch（或 stub 的 OnRemoteRequest）出发，沿全部分发路径追踪 cmdId 与 data 的流向。对不在 `<target_path>` 下的 data 业务函数，沿 data 实参继续追踪；到 `<target_path>` 内接收 data 且不构成分发的业务函数后，只对明显的单线封装继续下钻，停在真正处理 data 的函数；找不到时保留最深封装函数。不展开最终接口的内部实现，也不做完整污点传播。最终把接口函数候选写入 `<out>/03_trace.json`。
```

- [ ] **Step 3: Replace constraint 4**

Replace:

```text
4. 只输出 `<target_path>` 下接收 data 且不构成分发的业务函数（含 3.6 的 `*` 无条件/内联处理记录）；目标路径外只沿 data 实参继续追踪，不展开目标路径内业务实现。
```

with:

```text
4. 只输出 `<target_path>` 下真正处理 data 的业务函数；明显单线封装继续下钻，找不到真正处理函数时输出最深封装函数（含 3.6 的 `*` 无条件/内联处理记录）。目标路径外只沿 data 实参继续追踪，不展开最终接口内部实现。
```

- [ ] **Step 4: Add terminology rows**

After:

```text
| 接口函数 | cmdId 用例分支中直接接收客户端 data（或 data 派生值）且定义在 `<target_path>` 下的业务函数 |
```

insert:

```text
| 封装函数 | target_path 内只做校验/日志/别名，并将 data 单线传给名称相近下游的函数 |
| fallback_handler | 已发现的最深 target_path 封装函数；未找到真正处理函数时作为外部接口 |
```

- [ ] **Step 5: Update example 1 phase 3**

Replace:

```text
4. 阶段三：以 `DemoDriverDispatch` 初始化内存队列，追踪 `SendRequest` 到 `DemoInterfaceStub::OnRemoteRequest`，枚举 8 个 cmdId 用例并解析宏值；对不在 `<target_path>` 下的直接目标继续沿 data 实参追踪，直到 `DemoVendorInterface(data)` 落在 `<target_path>` 下；队列为空后写 `<out>/03_trace.json`。
```

with:

```text
4. 阶段三：以 `DemoDriverDispatch` 初始化内存队列，追踪 `SendRequest` 到 `DemoInterfaceStub::OnRemoteRequest`，枚举 8 个 cmdId 用例并解析宏值；对不在 `<target_path>` 下的直接目标继续沿 data 实参追踪，跳过明显封装函数，直到 `DemoVendorInterface(data)` 真正处理 data；若只到达封装层，则记录最深的 `DemoVendorWrapper`；队列为空后写 `<out>/03_trace.json`。
```

- [ ] **Step 6: Verify Task 1**

```bash
grep -n '只识别目标路径内真正处理 data 的业务函数' hal-service-api-finder/SKILL.md
grep -n '只输出 `<target_path>` 下真正处理 data 的业务函数' hal-service-api-finder/SKILL.md
grep -n '| 封装函数 |' hal-service-api-finder/SKILL.md
grep -n '| fallback_handler |' hal-service-api-finder/SKILL.md
grep -n 'DemoVendorWrapper' hal-service-api-finder/SKILL.md
grep -c '^## ' hal-service-api-finder/SKILL.md
```

Expected: all five new lines present; 12 headings.

- [ ] **Step 7: Commit**

```bash
git add hal-service-api-finder/SKILL.md
git -c user.name=siakam43 -c user.email=429872468@qq.com commit -m "docs(hal-service-api-finder): document wrapper tracing"
```

---

### Task 2: Add wrapper decision to 3.6 and fallback to 3.8/3.10

**Files:**
- Modify: `hal-service-api-finder/SKILL.md` (3.6 branch rules; 3.8 data_chase fields/key; 3.10 termination/key)

- [ ] **Step 1: Update 3.6 step 4 and table row**

Replace:

```text
4. 接收 data/派生参数且不构成分发的业务函数 → 定义在 `<target_path>` 下记为接口候选，否则入队 `kind = "data_chase"` 工作项；
```

with:

```text
4. 接收 data/派生参数且不构成分发的业务函数 → 定义在 `<target_path>` 下执行下方封装判定；否则入队 `kind = "data_chase"` 工作项；
```

Replace:

```text
| 接收 data/派生参数，且不构成分发 | 在 `<target_path>` 下记为接口候选；否则入队 `data_chase` |
```

with:

```text
| 接收 data/派生参数，且不构成分发 | `<target_path>` 下执行封装判定：封装入队 `data_chase`，否则记为接口；路径外入队 `data_chase` |
```

- [ ] **Step 2: Add the wrapper decision block after the path-prefix sentence**

After:

```text
目标是否在 `<target_path>` 下，按函数定义文件的绝对路径前缀判断（区分大小写）。
```

insert:

```text
封装判定（仅对 `<target_path>` 下接收 data/派生参数、不构成分发、且不是排除项的业务函数）：

1. 自身不处理 data：不读取/解析 data 字段，不基于 data 做业务分支或计算，不写入成员/全局，不执行业务动作；只允许参数校验、空指针/长度/魔数/范围检查、日志、局部别名和类型转换。
2. 函数体简单；出现复杂控制流或难以判断的逻辑 → 按真正处理处理，停止下钻。
3. 统计业务 data 下游调用（实参含 data/派生值，排除纯校验/纯日志/框架转换调用）：
   - 0 个 → 当前函数记为接口；
   - ≥ 2 个 → 当前函数记为接口；
   - 1 个 → 继续第 4 条。
4. 该下游调用满足以下全部条件才判定为封装：
   - 下游是普通业务函数，不是 dispatcher；
   - 下游定义可解析且在 scope 内；
   - 去掉命名空间/类限定后，当前函数名与下游函数名中，一个是另一个的前缀或后缀扩展，多出的部分不超过 8 个字符，且两者不完全相同。
5. 判定为封装 → 不记录当前函数，按 3.8 入队 `data_chase`，目标为唯一业务下游，`fallback_handler` = 当前函数。
6. 任一条件不满足 → 当前函数记为接口，停止。
```

- [ ] **Step 3: Update the multi-target and unconditional-call rules**

Replace:

```text
- 一个分支有多个 data 消费目标 → 每个目标分别按 `<target_path>` 判断：在目标路径下的各写一条 `handlers`，共用同一 `effective_outer`，并把 `{name, file, line}` 追加到当前 case 的 `handlers`；不在目标路径下的逐个入队 `data_chase`。
```

with:

```text
- 一个分支有多个 data 消费目标 → 每个目标分别按 `<target_path>` 判断：在目标路径下的逐个执行封装判定（封装入队 `data_chase`，否则写 `handlers`，共用同一 `effective_outer` 并把 `{name, file, line}` 追加到当前 case）；不在目标路径下的逐个入队 `data_chase`。
```

Replace:

```text
- 无 cmdId 分支、无条件调用 data 消费函数 → 目标在 `<target_path>` 下时记为 `cmd_id_macro = "*"`、`cmd_id_value = null`；否则入队 `data_chase`。两种情况都在 evidence 中说明“无条件直接处理”；无当前 case 时写入追踪链。
```

with:

```text
- 无 cmdId 分支、无条件调用 data 消费函数 → 目标在 `<target_path>` 下时执行封装判定（封装入队 `data_chase`，否则记为 `cmd_id_macro = "*"`、`cmd_id_value = null`）；否则入队 `data_chase`。两种情况都在 evidence 中说明“无条件直接处理”；无当前 case 时写入追踪链。
```

- [ ] **Step 4: Update 3.8 data_chase description, fields, and key**

Replace:

```text
`kind = "data_chase"` 工作项（不在 `<target_path>` 下的 data 业务函数）字段：
```

with:

```text
`kind = "data_chase"` 工作项（路径外 data 业务函数，或 target_path 封装下钻目标）字段：
```

After:

```text
- `evidence_chain`：从触发追踪的原目标到当前函数的调用链文本。
```

insert:

```text
- `fallback_handler`：已发现的最深 `<target_path>` 封装函数 `{function, file, line}`；没有时为 null。
```

Replace:

```text
- `data_chase` 环检测键：`(kind="data_chase", 定义文件+行号, effective_outer.expr, effective_outer.value, sorted(data_vars), origin_case_id)`。
```

with:

```text
- `data_chase` 环检测键：`(kind="data_chase", 定义文件+行号, effective_outer.expr, effective_outer.value, sorted(data_vars), origin_case_id, fallback_handler 的 function/file/line 或 null)`。
```

- [ ] **Step 5: Update 3.10 condition 1 and dedup key**

Replace:

```text
1. 找到 `<target_path>` 下接收 data 且不构成分发的业务函数（接口函数已写入 `handlers`）；
```

with:

```text
1. 找到 `<target_path>` 下真正处理 data 的业务函数（已写入 `handlers`），或已记录 fallback；
```

Replace:

```text
- `data_chase` 工作项以 `(kind, 定义文件+行号, effective_outer.expr, effective_outer.value, sorted(data_vars), origin_case_id)` 去重。
```

with:

```text
- `data_chase` 工作项以 `(kind, 定义文件+行号, effective_outer.expr, effective_outer.value, sorted(data_vars), origin_case_id, fallback_handler 的 function/file/line 或 null)` 去重。
```

- [ ] **Step 6: Verify Task 2**

```bash
sed -n '/^### 3.6/,/^### 3.7/p' hal-service-api-finder/SKILL.md | grep -n '封装判定\|前缀或后缀\|fallback_handler'
sed -n '/^### 3.8/,/^### 3.9/p' hal-service-api-finder/SKILL.md | grep -n 'fallback_handler\|封装下钻'
sed -n '/^### 3.10/,/^收尾：/p' hal-service-api-finder/SKILL.md | grep -n '真正处理 data\|fallback_handler'
grep -c '^## ' hal-service-api-finder/SKILL.md
```

Expected: wrapper rules in 3.6; fallback in 3.8/3.10; 12 headings.

- [ ] **Step 7: Commit**

```bash
git add hal-service-api-finder/SKILL.md
git -c user.name=siakam43 -c user.email=429872468@qq.com commit -m "docs(hal-service-api-finder): add wrapper decision and fallback"
```

---

### Task 3: Rewrite 3.9 wrapper tracing and finalize

**Files:**
- Modify: `hal-service-api-finder/SKILL.md` (3.9)

- [ ] **Step 1: Replace 3.9 steps 1-6**

Replace the block from:

```text
1. 按 3.2 的同样方法读取函数完整函数体；定义不在 scope 内或无法唯一确定 → 记 `unresolved`（`kind = "definition"`），结束该工作项。
```

through:

```text
6. 由 `definition` 或 `cycle` 终止的路径不再重复记 `target_path`；已入队的路径由后续工作项处理。
```

with:

```text
1. 按 3.2 的同样方法读取函数完整函数体；定义不在 scope 内或无法唯一确定 → 记 `unresolved`（`kind = "definition"`），结束该工作项。
2. 若当前函数定义在 `<target_path>` 下且是接收 data 的业务函数（非 dispatcher、非排除项）：按 3.6 的封装判定处理：
   - 封装 → 按 3.8 入队 `data_chase`，目标为唯一业务下游，`fallback_handler` = 当前函数；结束当前工作项；
   - 真正处理、0 个业务下游、多个业务下游、下游是 dispatcher、名字不相近或定义不可解析 → 按第 4 条记录当前函数，结束当前工作项。
3. 当前函数在 `<target_path>` 外：按 3.5 找出所有“实参含 data 或 data 派生值”的调用，对每个调用按 3.7 解析目标：
   - 定义不在 scope、无法唯一确定或函数体不完整 → 记 `unresolved`（`kind = "definition"`），继续下一个调用；
   - 已知转发器或接收 cmdId（含 data 派生 cmdId）且有分发逻辑 → 按 3.8 入队 `kind = "dispatch"`，`current_cmd_expr` 取调用点 cmdId 实参，`parent_case_id = origin_case_id`；
   - 排除项 → 忽略，把调用原文与忽略原因追加到 `origin_case_id` 对应 case 的 `evidence`；`origin_case_id` 为 null 时写入当前追踪链；
   - 接收 data 的业务函数，定义在 `<target_path>` 下 → 按 3.6 封装判定：封装则入队 `data_chase`（`fallback_handler` = 该函数），否则按第 4 条记录该函数；
   - 接收 data 的业务函数，定义在 `<target_path>` 外 → 按 3.8 入队 `data_chase`，`fallback_handler` 保持当前值；
   - 不接收 data/派生参数 → 不追踪。
4. 记录接口（真正处理函数或 fallback）：
   - `interface_function`、`file`、`definition_line` 取该函数；
   - `cmd_id_macro` / `cmd_id_value` 取 `effective_outer`；无外层时 `cmd_id_macro = "*"`、`cmd_id_value = null`；
   - 写入顶层 `handlers`，按现有五元组去重；
   - 若 `origin_case_id` 非 null，追加 `{name, file, line}` 到对应 case 的 `handlers`，并在该 case 的 `evidence` 追加追踪链；否则只写追踪链；
   - 记录 fallback 时，evidence 标注“最深封装函数（未找到真正处理函数）”。
5. 当前函数在 `<target_path>` 外且没有任何业务 data 下游调用：
   - `fallback_handler` 非 null → 按第 4 条记录 fallback；
   - 否则 → `unresolved`（`kind = "target_path"`），detail 记录起点函数、`evidence_chain` 和已尝试的 data 实参调用。
6. 由 `definition` 或 `cycle` 终止的路径不再重复记 `target_path`；已入队的路径由后续工作项处理。
```

Keep steps 7-8 unchanged.

- [ ] **Step 2: Verify Task 3**

```bash
sed -n '/^### 3.9/,/^### 3.10/p' hal-service-api-finder/SKILL.md | grep -n '封装判定\|fallback_handler\|真正处理\|最深封装'
grep -n 'fallback_handler' hal-service-api-finder/SKILL.md
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

Expected: wrapper/fallback rules in 3.9; 5 JSON blocks valid; 12 headings; line count about 1020 (record actual).

- [ ] **Step 3: Commit**

```bash
git add hal-service-api-finder/SKILL.md
git -c user.name=siakam43 -c user.email=429872468@qq.com commit -m "docs(hal-service-api-finder): trace through data wrappers"
```

---

## Plan Self-Review

- **Spec coverage:** §3 wrapper criteria → Task 2 Step 2; §4.1 fallback → Task 2 Step 4; §4.2 flow → Task 3 Step 1; §5 recording/evidence → Task 3 Step 1 item 4; §6 document changes → Tasks 2-3; §7 global docs → Task 1.
- **Placeholder scan:** no TBD/TODO; every replacement is complete; every verification has expected output.
- **Consistency:** the wrapper decision is defined once in 3.6 and referenced by 3.9; `fallback_handler` appears in 3.8 fields, both dedup keys, 3.9, and the terminology; 0/≥2 downstream calls always record the current function, so the single-line wrapper path is unambiguous.
