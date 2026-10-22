# hal-service-api-finder Search Refactor Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Consolidate all search command templates into one named-method subsection in `SKILL.md` and replace repeated command blocks in later phases with short method references.

**Architecture:** Add three named search methods (`精确字符串搜索`, `正则/词边界搜索`, `指定文件搜索`) in section 三, then rewrite sections 五、六、七、十二 so they reference those names instead of repeating `xargs`/`grep` commands. No analysis behavior, state schema, or output format changes.

**Tech Stack:** Markdown only. Verification is static: grep checks, heading count, JSON-block parsing, and a before/after line count. No runtime tests.

---

## Testing Strategy

The deliverable is a prompt document, not executable code. Each task ends with static checks:

1. **Command-location checks:** verify complete `xargs`/`grep` templates appear only in section 三.
2. **Reference checks:** verify later sections use the three named methods and do not contain old `方式一`/`方式二` or repeated scope paragraphs.
3. **Structure checks:** 12 top-level headings, 4 fenced `json` blocks parse, no placeholders.
4. **Line-count check:** record `wc -l` before and after; the refactor should reduce or at least not increase the line count.

Use this command from the repository root:

```bash
cd /home/admin/cc/wksp/siakam_security_skills/hal_service_api_finder
```

## File Structure

- **Modify:** `hal-service-api-finder/SKILL.md` — section 三 (search methods), sections 五、六、七、十二 (method references).
- **Modify:** `docs/superpowers/plans/2026-10-22-hal-service-api-finder.md` — add a one-line historical note at the top.
- **Modify:** `docs/superpowers/specs/2026-10-22-hal-service-api-finder-design.md` — add a one-line historical note at the top.
- **Create:** `docs/superpowers/plans/2026-10-22-hal-service-api-finder-search-refactor.md` — this plan.
- **Do not modify:** `rr1.txt`, the skill's behavior rules, state schemas, or final JSON schema.

---

### Task 1: Consolidate search methods in section 三

**Files:**
- Modify: `hal-service-api-finder/SKILL.md` (section 三, subsection `### 搜索纪律（适用于所有范围分支）`)

- [ ] **Step 1: Replace the search-discipline subsection with named methods**

Replace the entire block from `### 搜索纪律（适用于所有范围分支）` through the codegraph bullet with:

````markdown
### 搜索方法与纪律

所有阶段统一使用以下三种方法；后续只写方法名与参数，不再重复命令。

1. 精确字符串搜索：在 scope 内查找字面量。
2. 正则/词边界搜索：需要正则时把 `-F` 换成 `-E`；需要词边界时加 `-w`。
3. 指定文件搜索：已知文件在 scope 内时使用。

```bash
# 1. 精确字符串搜索
xargs -r -a "<out>/tmp/scope_files.txt" -d '\n' grep -I -H -n -F "<pattern>"

# 2. 正则/词边界搜索
xargs -r -a "<out>/tmp/scope_files.txt" -d '\n' grep -I -H -n -E "<pattern>"

# 3. 指定文件搜索
grep -I -H -n -E "<pattern>" "<file>"
```

全局纪律：
- 只搜索 scope_files.txt 中的文件；scope_roots.txt 非 null 时才允许按根目录搜索（把文件清单换成 scope_roots.txt 并加 -r）。
- 禁止 grep -rn <project_dir>、rg <project_dir>、find <project_dir> -exec grep 等全仓搜索。
- grep -I 保留，跳过二进制文件；不得用 2>/dev/null 隐藏错误；退出码 1/123 表示无匹配，不是失败。
- codegraph 结果按 scope_files.txt 过滤；范围外搜索命中直接丢弃，已知范围外目标记 unresolved。
````

The existing `<out>` definition line immediately above the old code block must remain; place it immediately before the new method block if it is not already there.

- [ ] **Step 2: Update the cross-reference in the clean_code subsection**

Replace:

```text
- 搜索方式见下方 `### 搜索纪律（适用于所有范围分支）`。
```

with:

```text
- 搜索方式见下方 `### 搜索方法与纪律`。
```

- [ ] **Step 3: Verify Task 1**

Run:

```bash
grep -n '### 搜索方法与纪律' hal-service-api-finder/SKILL.md
sed -n '/^### 搜索方法与纪律/,/^### macro.json/p' hal-service-api-finder/SKILL.md | grep -c 'xargs -r -a'
sed -n '/^### 搜索方法与纪律/,/^### macro.json/p' hal-service-api-finder/SKILL.md | grep -c 'grep -I -H -n'
grep -n '搜索纪律（适用于所有范围分支）' hal-service-api-finder/SKILL.md || echo 'old heading removed'
```

Expected:
- `### 搜索方法与纪律` appears once.
- `xargs -r -a` count is `2`.
- `grep -I -H -n` count is `3`.
- `old heading removed`.

- [ ] **Step 4: Commit**

```bash
git add hal-service-api-finder/SKILL.md
git -c user.name=siakam43 -c user.email=429872468@qq.com commit -m "docs(hal-service-api-finder): consolidate search methods"
```

---

### Task 2: Simplify section 五 (Phase 1) to method references

**Files:**
- Modify: `hal-service-api-finder/SKILL.md` (section 五)

- [ ] **Step 1: Replace the service-name search command**

Replace the block:

````text
按第三章“方式一”在 scope 内搜索带双引号的完整字面量 `"<service_name>"`（`<out>` 定义同第三章）：

```bash
xargs -r -a "<out>/tmp/scope_files.txt" -d '\n' grep -I -H -n -F '"<service_name>"'
```

- 只允许搜索 `scope_files.txt` 中的文件；禁止 `grep -rn <project_dir>` 或任何等价的全仓搜索。`scope_roots.txt` 非 null 时可改用第三章“方式二”，但最终仍以 `scope_files.txt` 为硬边界。
- 每个命中都必须用 Read 读取至少 30 行上下文（`offset=max(1, 命中行-15)`，`limit` 不少于 31）；向上读到初始化器声明行，向下读到闭合 `};`。不得只凭 grep 行下结论。
````

with:

````text
按「精确字符串搜索」查找带双引号的完整字面量 `"<service_name>"`。

- 每个命中都必须用 Read 读取至少 30 行上下文（`offset=max(1, 命中行-15)`，`limit` 不少于 31）；向上读到初始化器声明行，向下读到闭合 `};`。不得只凭 grep 行下结论。
````

- [ ] **Step 2: Replace the entry-variable search command**

Replace:

````text
对每个候选的 `variable`，按第三章“方式一”在 scope 内搜索其变量名（用 `-w` 避免子串误命中）：

```bash
xargs -r -a "<out>/tmp/scope_files.txt" -d '\n' grep -I -H -n -E -w "<entry_var>"
```
````

with:

```text
对每个候选的 `variable`，按「正则/词边界搜索」查找其变量名（加 `-w` 避免子串误命中）。
```

- [ ] **Step 3: Verify Task 2**

Run:

```bash
sed -n '/^## 五、/,/^## 六、/p' hal-service-api-finder/SKILL.md | grep -n 'xargs -r -a' || echo 'no xargs in section 5'
sed -n '/^## 五、/,/^## 六、/p' hal-service-api-finder/SKILL.md | grep -n '方式一\|方式二' || echo 'no old method references in section 5'
grep -n '按「精确字符串搜索」\|按「正则/词边界搜索」' hal-service-api-finder/SKILL.md
```

Expected:
- `no xargs in section 5`
- `no old method references in section 5`
- both method references appear at least once.

- [ ] **Step 4: Commit**

```bash
git add hal-service-api-finder/SKILL.md
git -c user.name=siakam43 -c user.email=429872468@qq.com commit -m "docs(hal-service-api-finder): use named search methods in phase 1"
```

---

### Task 3: Simplify section 六 (Phase 2) to method references

**Files:**
- Modify: `hal-service-api-finder/SKILL.md` (section 六)

- [ ] **Step 1: Remove the repeated scope-search paragraph**

Delete this bullet entirely:

```text
本阶段所有搜索沿用第三章“搜索纪律”：只允许搜索 `scope_files.txt` 中的文件；`scope_roots.txt` 非 null 时可按第三章“方式二”搜索，但最终仍以 `scope_files.txt` 为硬边界；禁止任何等价的全仓搜索。
```

- [ ] **Step 2: Replace the macro-normalization search reference**

Replace:

```text
- 标识符是宏时：按第三章“方式一”在 scope 内搜索 `#define <宏名>`，必要时查 `macro.json`，逐层展开到函数名；无法展开或展开后找不到定义 → 报 `bind_definition_not_found`。
```

with:

```text
- 标识符是宏时：按「精确字符串搜索」查找 `#define <宏名>`，必要时查 `macro.json`，逐层展开到函数名；无法展开或展开后找不到定义 → 报 `bind_definition_not_found`。
```

- [ ] **Step 3: Replace the Bind definition search commands**

Replace the block:

````text
   - 优先在 `entry.file` 内搜索（该文件必在 scope 内）：

     ```bash
     grep -I -H -n -E -w "<bind_name>" "<entry_file>"
     ```

   - 未找到再按第三章“方式一”在全 scope 搜索：

     ```bash
     xargs -r -a "<out>/tmp/scope_files.txt" -d '\n' grep -I -H -n -E -w "<bind_name>"
     ```
````

with:

````text
   - 优先按「指定文件搜索」在 `entry.file` 内查找 `<bind_name>`（该文件必在 scope 内）。
   - 未找到再按「正则/词边界搜索」在全 scope 查找 `<bind_name>`。
````

- [ ] **Step 4: Replace the Dispatch search commands**

Replace:

````text
3. 按第三章“方式一”在 scope 内搜索 Dispatch 赋值与 host 标识：

   ```bash
   xargs -r -a "<out>/tmp/scope_files.txt" -d '\n' grep -I -H -n -E '(\.|->)Dispatch[[:space:]]*='
   xargs -r -a "<out>/tmp/scope_files.txt" -d '\n' grep -I -H -n -E -w "<host_var>"
   ```

   其中 `<host_var>` 替换为实际变量名；可再用 host 类型名补充搜索。
````

with:

```text
3. 按「正则/词边界搜索」在 scope 内查找 Dispatch 赋值 `(\.|->)Dispatch[[:space:]]*=` 与 host 标识 `<host_var>`；其中 `<host_var>` 替换为实际变量名，可再用 host 类型名补充搜索。
```

- [ ] **Step 5: Replace the OnRemoteRequest search reference**

Replace:

```text
4. 定位 `OnRemoteRequest`：按第三章“方式一”优先搜索 `<stub_class>::OnRemoteRequest`；
```

with:

```text
4. 定位 `OnRemoteRequest`：按「正则/词边界搜索」优先搜索 `<stub_class>::OnRemoteRequest`；
```

Keep the rest of that paragraph unchanged.

- [ ] **Step 6: Verify Task 3**

Run:

```bash
sed -n '/^## 六、/,/^## 七、/p' hal-service-api-finder/SKILL.md | grep -n 'xargs -r -a\|grep -I -H' || echo 'no full commands in section 6'
sed -n '/^## 六、/,/^## 七、/p' hal-service-api-finder/SKILL.md | grep -n '方式一\|方式二\|本阶段所有搜索沿用' || echo 'no old references in section 6'
sed -n '/^## 六、/,/^## 七、/p' hal-service-api-finder/SKILL.md | grep -n '精确字符串搜索\|正则/词边界搜索\|指定文件搜索'
```

Expected:
- `no full commands in section 6`
- `no old references in section 6`
- all three method names appear.

- [ ] **Step 7: Commit**

```bash
git add hal-service-api-finder/SKILL.md
git -c user.name=siakam43 -c user.email=429872468@qq.com commit -m "docs(hal-service-api-finder): use named search methods in phase 2"
```

---

### Task 4: Simplify section 七 and section 十二

**Files:**
- Modify: `hal-service-api-finder/SKILL.md` (sections 七 and 十二)

- [ ] **Step 1: Remove the repeated scope-search bullet in section 七**

Delete this bullet entirely:

```text
- 所有搜索沿用第三章“搜索纪律”：只读 `scope_files.txt` 列出的文件；`scope_roots.txt` 非 null 时可按第三章“方式二”搜索，但最终仍以 `scope_files.txt` 为硬边界；禁止任何等价的全仓搜索。
```

- [ ] **Step 2: Replace the dispatcher-definition search reference**

In section 3.2 step 2, replace:

```text
否则按第三章“方式一”在 scope 内搜索精确名称
```

with:

```text
否则按「正则/词边界搜索」在 scope 内搜索精确名称
```

- [ ] **Step 3: Replace the example search command**

In section 十二 示例三, replace:

```text
2. 阶段一在 scope 内搜索 `"demo_service_typo"`（`xargs -r -a "<out>/tmp/scope_files.txt" -d '\n' grep -I -H -n -F '"demo_service_typo"'`），0 命中。
```

with:

```text
2. 阶段一按「精确字符串搜索」查找 `"demo_service_typo"`，0 命中。
```

- [ ] **Step 4: Verify Task 4**

Run:

```bash
sed -n '/^## 七、/,/^## 八、/p' hal-service-api-finder/SKILL.md | grep -n '方式一\|方式二\|所有搜索沿用' || echo 'no old references in section 7'
grep -n 'xargs -r -a' hal-service-api-finder/SKILL.md
grep -n '按「精确字符串搜索」查找 `"demo_service_typo"`' hal-service-api-finder/SKILL.md
```

Expected:
- `no old references in section 7`
- `xargs -r -a` appears exactly twice (both in section 三).
- the demo_service_typo method reference appears once.

- [ ] **Step 5: Commit**

```bash
git add hal-service-api-finder/SKILL.md
git -c user.name=siakam43 -c user.email=429872468@qq.com commit -m "docs(hal-service-api-finder): use named search methods in phase 3 and examples"
```

---

### Task 5: Add historical notes, run final checks, and push

**Files:**
- Modify: `docs/superpowers/plans/2026-10-22-hal-service-api-finder.md`
- Modify: `docs/superpowers/specs/2026-10-22-hal-service-api-finder-design.md`

- [ ] **Step 1: Add the historical note to the old plan**

Immediately after the plan's header block (after the `---` that follows `**Tech Stack:**`), insert:

```text
> 注：搜索命令已统一到 SKILL.md 第三章「搜索方法与纪律」，本文中的完整命令仅作历史记录。
```

- [ ] **Step 2: Add the historical note to the old design spec**

Immediately after the metadata list at the top of the spec (after the `- 交付物：...` line), insert the same note:

```text
> 注：搜索命令已统一到 SKILL.md 第三章「搜索方法与纪律」，本文中的完整命令仅作历史记录。
```

- [ ] **Step 3: Run the final structural and reference checks**

```bash
grep -c '^## ' hal-service-api-finder/SKILL.md
grep -c 'xargs -r -a' hal-service-api-finder/SKILL.md
grep -c 'grep -I -H -n' hal-service-api-finder/SKILL.md
grep -n -E '按第三章“方式一”|按第三章“方式二”|本阶段所有搜索沿用|所有搜索沿用第三章' hal-service-api-finder/SKILL.md || echo 'no old references'
grep -n -E '按「精确字符串搜索」|按「正则/词边界搜索」|按「指定文件搜索」' hal-service-api-finder/SKILL.md
python3 - <<'PY'
import json, re, pathlib
text = pathlib.Path("hal-service-api-finder/SKILL.md").read_text(encoding="utf-8")
blocks = re.findall(r"```json\n(.*?)\n```", text, re.S)
for block in blocks:
    json.loads(block)
print(f"{len(blocks)} JSON blocks valid")
PY
wc -l hal-service-api-finder/SKILL.md
git diff --stat
```

Expected:
- heading count `12`;
- `xargs -r -a` count `2`;
- `grep -I -H -n` count `3`;
- `no old references`;
- named method references appear multiple times;
- `4 JSON blocks valid`;
- `wc -l` is less than or equal to the baseline `1124`;
- `git diff --stat` lists only `SKILL.md`, the old plan, and the old spec.

- [ ] **Step 4: Commit and push**

```bash
git add hal-service-api-finder/SKILL.md docs/superpowers/plans/2026-10-22-hal-service-api-finder.md docs/superpowers/specs/2026-10-22-hal-service-api-finder-design.md
git -c user.name=siakam43 -c user.email=429872468@qq.com commit -m "refactor(hal-service-api-finder): deduplicate search instructions"
git push origin main
git log -1 --oneline
git status --short --branch
```

Expected: commit succeeds, push succeeds, working tree clean.

---

## Plan Self-Review

- **Spec coverage:** Task 1 covers the named-method block; Task 2 covers section 五; Task 3 covers section 六; Task 4 covers section 七 and the example; Task 5 covers historical notes, validation, and push. Every spec section maps to a task.
- **Placeholder scan:** No TBD/TODO/fill-in instructions; every edit shows the exact old/new text and every verification has an expected result.
- **Type consistency:** Method names are used consistently: `精确字符串搜索`, `正则/词边界搜索`, `指定文件搜索`. No `方式一`/`方式二` remains after Task 4.
