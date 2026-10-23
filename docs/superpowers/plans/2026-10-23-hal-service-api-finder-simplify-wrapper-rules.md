# hal-service-api-finder Simplify Wrapper Rules Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the six-condition wrapper heuristic with a name-only rule based on distinct data downstream targets, path/scope, and identical or short prefix/suffix names.

**Architecture:** Two documentation edits in `hal-service-api-finder/SKILL.md`. Task 1 replaces the 3.6 wrapper block and 3.9 step 2. Task 2 aligns global wording, terminology, 3.10 condition, and final verification. Fallback machinery is unchanged.

**Tech Stack:** Markdown only. Verification is static: grep checks, JSON parsing, heading count, and `wc -l`.

---

## File Structure

- **Modify:** `hal-service-api-finder/SKILL.md` — sections 一, 七, 十, 十一, 十二.
- **Do not modify:** snapshot schemas, error table, final JSON schema, target_path definition, fallback mechanics, historical plans/specs, `rr1.txt`.

---

### Task 1: Replace the wrapper decision and 3.9 step 2

**Files:**
- Modify: `hal-service-api-finder/SKILL.md` (3.6 wrapper block; 3.9 step 2)

- [ ] **Step 1: Replace the 3.6 wrapper decision block**

Replace the block from:

```text
封装判定（仅对 `<target_path>` 下接收 data/派生参数、不构成分发、且不是排除项的业务函数）：
```

through:

```text
6. 任一条件不满足 → 当前函数记为接口，停止。
```

with:

```text
封装判定（仅对 `<target_path>` 下接收 data/派生参数、不构成分发、且不是排除项的业务函数）：

1. 找出函数体内所有“实参含 data 或 data 派生值”的调用，按目标定义 `(file, line)` 去重；同一目标被调用多次只算一个目标。
2. 不同目标数为 0 → 当前函数记为接口。
3. 不同目标数 ≥ 2 → 当前函数记为接口。
4. 只有一个不同目标时：
   - 目标定义不在 `<target_path>` 或不在 scope 内 → 当前函数记为接口；
   - 目标是 dispatcher/转发器 → 不按封装处理，当前函数不记录，按现有 dispatch 规则入队该目标（`current_cmd_expr` 取调用点 cmdId 实参，`parent_case_id = origin_case_id`）；若当前已有 `fallback_handler`，按 3.9 现有 dispatcher 规则处理；
   - 目标是普通业务函数：去掉命名空间/类限定后比较名称。名称完全相同，或一个是另一个的前缀/后缀扩展且多出的部分不超过 8 个字符 → 判定为封装；否则当前函数记为接口。
5. 判定为封装 → 不记录当前函数，按 3.8 入队 `data_chase`，目标为唯一业务下游，`fallback_handler` = 当前函数；`effective_outer` 取当前值，`origin_case_id` 取当前 case id，无条件处理时两者均为 null；把 `当前函数名(data)` 追加到 `evidence_chain`。
```

- [ ] **Step 2: Replace 3.9 step 2**

Replace:

```text
2. 若当前函数定义在 `<target_path>` 下且是接收 data 的业务函数（非 dispatcher、非排除项）：按 3.6 的封装判定处理：
   - 封装 → 按 3.8 入队 `data_chase`，目标为唯一业务下游，`fallback_handler` = 当前函数，并把 `当前函数名(data)` 追加到 `evidence_chain`；结束当前工作项；
   - 真正处理、0 个业务下游、多个业务下游、下游是 dispatcher、名字不相近或定义不可解析 → 按第 4 条记录当前函数，结束当前工作项。
```

with:

```text
2. 若当前函数定义在 `<target_path>` 下且是接收 data 的业务函数（非 dispatcher、非排除项）：按 3.6 的封装判定处理：
   - 封装 → 按 3.8 入队 `data_chase`，目标为唯一业务下游，`fallback_handler` = 当前函数，并把 `当前函数名(data)` 追加到 `evidence_chain`；结束当前工作项；
   - 唯一目标是 dispatcher/转发器 → 不记录当前函数；若已有 `fallback_handler`，按 3.9 现有 dispatcher 规则记录 fallback；否则按 3.8 入队 dispatch；结束当前工作项；
   - 其余（0 个/多个目标、目标在路径外或 scope 外、名字不匹配等）→ 按第 4 条记录当前函数，结束当前工作项。
```

- [ ] **Step 3: Verify Task 1**

```bash
sed -n '/^封装判定/,/^分支级规则/p' hal-service-api-finder/SKILL.md | grep -n '自身不处理 data\|函数体简单\|统计业务 data 下游调用\|排除纯校验' || echo 'old wrapper conditions removed'
sed -n '/^封装判定/,/^分支级规则/p' hal-service-api-finder/SKILL.md | grep -n '按目标定义\|名称完全相同\|前后缀扩展\|不同目标数为 0\|不同目标数 ≥ 2'
sed -n '/^### 3.9/,/^### 3.10/p' hal-service-api-finder/SKILL.md | grep -n '其余（0 个/多个目标\|唯一目标是 dispatcher'
grep -c '^## ' hal-service-api-finder/SKILL.md
```

Expected: old conditions gone; new 5-rule block present; 3.9 step 2 updated; 12 headings.

- [ ] **Step 4: Commit**

```bash
git add hal-service-api-finder/SKILL.md
git -c user.name=siakam43 -c user.email=429872468@qq.com commit -m "docs(hal-service-api-finder): simplify wrapper decision"
```

---

### Task 2: Align global wording, terms, and final checks

**Files:**
- Modify: `hal-service-api-finder/SKILL.md` (principle 3, chapter 七 intro, 3.5, 3.9 step 4 fallback annotation, 3.10 condition 1, constraint 4, terminology, anti-rationalization, example 1)

- [ ] **Step 1: Replace core principle 3**

Replace:

```text
3. **只识别目标路径内真正处理 data 的业务函数。** cmdId 用例分支中直接接收 data 且不构成分发的业务函数若位于 `<target_path>` 下，先判断是否为明显封装：是则沿唯一 data 下游继续，直到真正处理 data 的函数；否则当前函数即接口。找不到真正处理函数时，保留最深的 `<target_path>` 封装函数。仍不展开最终接口的内部实现。
```

with:

```text
3. **只识别目标路径内第一个非封装业务函数。** cmdId 用例分支中直接接收 data 且不构成分发的业务函数若位于 `<target_path>` 下，先按名字规则判断是否为单线封装：是则沿唯一 data 下游继续，直到第一个非封装业务函数；否则当前函数即接口。找不到非封装业务函数时，保留最深的 `<target_path>` 封装函数。仍不展开最终接口的内部实现。
```

- [ ] **Step 2: Replace the chapter 七 intro**

Replace:

```text
阶段三只做一件事：从阶段二的 Dispatch（或 stub 的 OnRemoteRequest）出发，沿全部分发路径追踪 cmdId 与 data 的流向。对不在 `<target_path>` 下的 data 业务函数，沿 data 实参继续追踪；到 `<target_path>` 内接收 data 且不构成分发的业务函数后，只对明显的单线封装继续下钻，停在真正处理 data 的函数；找不到时保留最深的 `<target_path>` 封装函数。不展开最终接口的内部实现，也不做完整污点传播。最终把接口函数候选写入 `<out>/03_trace.json`。
```

with:

```text
阶段三只做一件事：从阶段二的 Dispatch（或 stub 的 OnRemoteRequest）出发，沿全部分发路径追踪 cmdId 与 data 的流向。对不在 `<target_path>` 下的 data 业务函数，沿 data 实参继续追踪；到 `<target_path>` 内接收 data 且不构成分发的业务函数后，只对名字相近的单线封装继续下钻，停在第一个非封装业务函数；找不到时保留最深的 `<target_path>` 封装函数。不展开最终接口的内部实现，也不做完整污点传播。最终把接口函数候选写入 `<out>/03_trace.json`。
```

- [ ] **Step 3: Replace the 3.5 bullet**

Replace:

```text
- 只做分类所需的局部追踪，不要求完整污点传播路径；C++ 间接调用、虚函数、函数指针、重载等解析可参考 `taint-path-walker` skill，本 skill 只追踪到真正处理 data 的业务函数（明显单线封装继续按 3.6 下钻）或明确终止条件。
```

with:

```text
- 只做分类所需的局部追踪，不要求完整污点传播路径；C++ 间接调用、虚函数、函数指针、重载等解析可参考 `taint-path-walker` skill，本 skill 只追踪到第一个非封装业务函数（名字相近的单线封装继续按 3.6 下钻）或明确终止条件。
```

- [ ] **Step 4: Replace 3.9 step 4 fallback annotation and 3.10 condition 1**

Replace:

```text
4. 记录接口（真正处理函数或 fallback）：
```

with:

```text
4. 记录接口（非封装业务函数或 fallback）：
```

Replace:

```text
   - 记录 fallback 时，evidence 标注“最深封装函数（未找到真正处理函数）”。
```

with:

```text
   - 记录 fallback 时，evidence 标注“最深封装函数（未找到非封装业务函数）”。
```

Replace:

```text
1. 找到 `<target_path>` 下真正处理 data 的业务函数（已写入 `handlers`）；
```

with:

```text
1. 找到 `<target_path>` 下按名字规则判定为非封装的业务函数（已写入 `handlers`）；
```

- [ ] **Step 5: Replace constraint 4**

Replace:

```text
4. 只输出 `<target_path>` 下真正处理 data 的业务函数（含 3.6 的 `*` 无条件/内联处理记录）；明显单线封装继续下钻，找不到真正处理函数时输出最深的 `<target_path>` 封装函数。目标路径外只沿 data 实参继续追踪，不展开最终接口内部实现。
```

with:

```text
4. 只输出 `<target_path>` 下第一个非封装业务函数（含 3.6 的 `*` 无条件/内联处理记录）；名字相近的单线封装继续下钻，找不到非封装业务函数时输出最深的 `<target_path>` 封装函数。目标路径外只沿 data 实参继续追踪，不展开最终接口内部实现。
```

- [ ] **Step 6: Replace terminology rows**

Replace:

```text
| 接口函数 | cmdId 用例分支中最终记录为外部接口的业务函数：位于 `<target_path>` 下并真正处理 data，或找不到真正处理函数时的最深封装函数 |
```

with:

```text
| 接口函数 | cmdId 用例分支中最终记录为外部接口的业务函数：位于 `<target_path>` 下并按名字规则判定为非封装，或找不到非封装业务函数时的最深封装函数 |
```

Replace:

```text
| 封装函数 | target_path 内只做校验/日志/别名，并将 data 单线传给名称相近下游的函数 |
```

with:

```text
| 封装函数 | target_path 内只有一个 data 下游目标，目标在 target_path/scope 内，且名称相同或前后缀扩展 ≤8 字符的函数 |
```

Replace:

```text
| fallback_handler | 已发现的最深 target_path 封装函数；未找到真正处理函数时作为外部接口 |
```

with:

```text
| fallback_handler | 已发现的最深 target_path 封装函数；未找到非封装业务函数时作为外部接口 |
```

- [ ] **Step 7: Replace anti-rationalization and example wording**

Replace:

```text
| “业务函数内部还有处理，继续展开” | `<target_path>` 外只沿 data 实参追踪到目标路径；`<target_path>` 内只对明显单线封装继续下钻，不展开最终接口实现。 |
```

with:

```text
| “业务函数内部还有处理，继续展开” | `<target_path>` 外只沿 data 实参追踪到目标路径；`<target_path>` 内只对名字相近的单线封装继续下钻，不展开最终接口实现。 |
```

Replace:

```text
4. 阶段三：以 `DemoDriverDispatch` 初始化内存队列，追踪 `SendRequest` 到 `DemoInterfaceStub::OnRemoteRequest`，枚举 8 个 cmdId 用例并解析宏值；对不在 `<target_path>` 下的直接目标继续沿 data 实参追踪，跳过明显封装函数，直到 `DemoVendorInterface(data)` 真正处理 data；若只到达封装层，则记录最深的 `DemoVendorInterfaceWrapper`；队列为空后写 `<out>/03_trace.json`。
```

with:

```text
4. 阶段三：以 `DemoDriverDispatch` 初始化内存队列，追踪 `SendRequest` 到 `DemoInterfaceStub::OnRemoteRequest`，枚举 8 个 cmdId 用例并解析宏值；对不在 `<target_path>` 下的直接目标继续沿 data 实参追踪，跳过名字相近的单线封装函数，直到 `DemoVendorInterface(data)` 按名字规则不再被判为封装；若只到达封装层，则记录最深的 `DemoVendorInterfaceWrapper`；队列为空后写 `<out>/03_trace.json`。
```

- [ ] **Step 8: Verify Task 2**

```bash
grep -n '真正处理\|明显封装\|明显单线封装' hal-service-api-finder/SKILL.md || echo 'old wrapper wording removed'
grep -n '第一个非封装业务函数\|名字相近的单线封装\|只有一个 data 下游目标' hal-service-api-finder/SKILL.md
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

Expected: old wording gone; new wording present; 5 JSON blocks valid; 12 headings; line count recorded (actual 995; rules simplified but the block length stayed similar).

- [ ] **Step 9: Commit**

```bash
git add hal-service-api-finder/SKILL.md
git -c user.name=siakam43 -c user.email=429872468@qq.com commit -m "docs(hal-service-api-finder): align simplified wrapper wording"
```

---

## Plan Self-Review

- **Spec coverage:** §3 wrapper rules → Task 1 Step 1; §4 3.9 step 2 → Task 1 Step 2; §5 global wording → Task 2 Steps 1-7; §6 verification → Tasks 1-2.
- **Placeholder scan:** no TBD/TODO; every replacement is complete and every check has expected output.
- **Consistency:** the new 3.6 block is the single source of wrapper truth; 3.9 step 2 only references it; terminology, principle, intro, constraint, 3.10, and example all use “非封装/名字相近的单线封装” rather than “真正处理 data”.
