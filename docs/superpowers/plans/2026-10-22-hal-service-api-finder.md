# hal-service-api-finder SKILL.md Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create `hal-service-api-finder/SKILL.md`, a pure-markdown skill that identifies all external interface functions of a specified OpenHarmony/HDF HAL service by tracing moduleName → HdfDriverEntry → Bind → Dispatch → cmdId dispatch.

**Architecture:** A single Chinese prompt document organized as a four-phase state machine: `locate_entry → analyze_bind → trace_dispatch → output`. State persists in `progress.json`, `tmp/entry_state.json`, and `tmp/trace_state.json` for resume. All analysis is done by the main agent; there are no scripts and no sub-agents.

**Tech Stack:** Markdown only. Verification is static: grep checks, JSON-block parsing, and a manual walkthrough of the rr1 `ar_service` sample. No runtime tests.

---

## Testing Strategy

The deliverable is a prompt document, not executable code. Each task ends with executable static checks:

1. **Structure checks:** `grep` for required headings, field names, error codes, and forbidden instructions.
2. **JSON checks:** extract every fenced `json` block from `SKILL.md` and parse it with Python's `json` module.
3. **Final walkthrough:** manually trace the `ar_service` sample from rr1.txt through the prompt text and confirm every step has an actionable instruction.

Use these commands from the repository root:

```bash
cd /home/admin/cc/wksp/siakam_security_skills/hal_service_api_finder
```

The plan deliberately contains no runtime test files. Adding test code to the skill folder would violate the "纯 markdown" requirement.

## File Structure

- **Create:** `hal-service-api-finder/SKILL.md` — the entire deliverable.
- **Create:** `docs/superpowers/plans/2026-10-22-hal-service-api-finder.md` — this plan.
- **Modify:** none in the skill folder after creation.
- **Existing:** `.gitignore` ignores `.omc/`, `.ethunter_out/`, and `*.tmp`.

`SKILL.md` has exactly 12 top-level sections:

```text
## 一、核心原则
## 二、使用方法与输出
## 三、初始化与分析范围
## 四、任务恢复机制
## 五、阶段一：定位服务注册点
## 六、阶段二：分析 Bind
## 七、阶段三：cmdId 分发追踪
## 八、输出与自检
## 九、权限申请
## 十、约束规则
## 十一、抗理性化检查
## 十二、使用示例
```

---

### Task 1: Create the skill skeleton and frontmatter

**Files:**
- Create: `hal-service-api-finder/SKILL.md`

- [ ] **Step 1: Create the directory and file with the frontmatter and section skeleton**

Run:

```bash
mkdir -p hal-service-api-finder
```

Write these exact contents to `hal-service-api-finder/SKILL.md`:

````markdown
---
name: hal-service-api-finder
description: 当用户调用 /hal-service-api-finder <project_dir> <service_name>，或要求在 OpenHarmony/HDF HAL 层 C/C++ 代码中识别指定 service 对外提供的全部接口函数时使用。按 HdfDriverEntry.moduleName 精确定位服务注册点，分析 Bind/Dispatch/cmdId 分发，输出 cmdId 到接口函数的 JSON 映射。
---

# hal-service-api-finder

根据 OpenHarmony/HDF HAL 层 service 的注册流程，识别指定 service 对外提供的全部接口函数，输出 cmdId 到接口函数的 JSON 映射，用于安全攻击面分析。

## 一、核心原则

## 二、使用方法与输出

## 三、初始化与分析范围

## 四、任务恢复机制

## 五、阶段一：定位服务注册点

## 六、阶段二：分析 Bind

## 七、阶段三：cmdId 分发追踪

## 八、输出与自检

## 九、权限申请

## 十、约束规则

## 十一、抗理性化检查

## 十二、使用示例
````

- [ ] **Step 2: Verify the frontmatter and section count**

Run:

```bash
head -5 hal-service-api-finder/SKILL.md
grep -c '^## ' hal-service-api-finder/SKILL.md
```

Expected:
- `name: hal-service-api-finder` appears in the first 5 lines.
- The `grep -c` output is exactly `12`.

- [ ] **Step 3: Commit**

```bash
git add hal-service-api-finder/SKILL.md
git commit -m "feat(hal-service-api-finder): add skill skeleton"
```

---

### Task 2: Write core principles, usage, and output overview

**Files:**
- Modify: `hal-service-api-finder/SKILL.md` (sections 一 and 二)

- [ ] **Step 1: Fill section 一、核心原则**

Write exactly these five principles, each as a numbered item:

1. **分析质量优先于分析效率。** 宁可慢，不可草率。每个分支都要追到接口函数或明确的终止条件，不得因任务长而跳读、合并或提前退出。不设深度上限、不设扇出上限，所有静态可确定的目标全部追踪。
2. **分析范围是硬边界。** 存在 `clean_code.txt` 时，只分析列表（含文件夹展开）内的文件；所有搜索都限制在 scope 内，禁止 `grep -rn <project_dir>`。
3. **只识别直接目标。** cmdId 用例分支中直接接收 data 的业务函数就是接口函数；不展开业务函数的内部实现。
4. **证据优先，不臆造。** 每个接口、每个 cmdId 值、每个调用目标都要有代码证据；无法确定的写入 `unresolved`，不得猜测。
5. **全部由主 agent 完成。** 不调用 sub-agent，不把分析拆给子代理。

- [ ] **Step 2: Fill section 二、使用方法与输出**

Include exactly these subsections and rules:

```text
### 使用方法

/hal-service-api-finder <project_dir> <service_name>

- 两个参数均必填，不能为空；缺失任意一个或参数多于两个，打印用法并结束。
- project_dir 必须是已存在的目录；相对路径转换为绝对路径后使用。
- service_name 是 HdfDriverEntry.moduleName 的精确字符串，区分大小写，不做模糊匹配。
```

```text
### 输出

最终输出：<project_dir>/.ethunter_out/hal-service-api-finder/<service_name>_api.json

最终 JSON 为扁平数组，每条记录四个字段：
- cmd_id_macro：最外层具体 case 的宏名或表达式原文。
- cmd_id_value：整数；无法解析或非单一取值时为 null。
- interface_function：定义处的作用域限定名。
- file：函数定义所在文件的绝对路径。

中间结果：
- progress.json
- tmp/scope_files.txt
- tmp/scope_roots.txt（仅 clean_code.txt 分支存在）
- tmp/entry_state.json
- tmp/trace_state.json

不生成 summary 报告。

### 完成报告

完成时报告：分析范围文件数、service_name、入口变量、Bind 函数、Dispatch 函数数量、cmdId 用例数、接口函数数量、unresolved 数量、最终输出文件绝对路径。
```

- [ ] **Step 3: Verify required phrases**

Run:

```bash
grep -n '/hal-service-api-finder <project_dir> <service_name>' hal-service-api-finder/SKILL.md
grep -n '<service_name>_api.json' hal-service-api-finder/SKILL.md
grep -n '不设深度' hal-service-api-finder/SKILL.md
grep -n '不设扇出' hal-service-api-finder/SKILL.md
grep -n 'sub-agent' hal-service-api-finder/SKILL.md
```

Expected: every command prints at least one matching line.

- [ ] **Step 4: Commit**

```bash
git add hal-service-api-finder/SKILL.md
git commit -m "docs(hal-service-api-finder): add principles, usage, and output"
```

---

### Task 3: Write initialization and analysis scope

**Files:**
- Modify: `hal-service-api-finder/SKILL.md` (section 三)

- [ ] **Step 1: Write the initialization steps**

Include these subsections in order:

1. **参数解析** — validate two non-empty args; resolve `project_dir` to an absolute path; reject missing/non-directory and service_name forbidden characters; use literal/quoted search.
2. **输出目录初始化** — `mkdir -p <project_dir>/.ethunter_out/hal-service-api-finder/tmp`; failure is `output_dir_failed`.
3. **分析范围确定** — clean_code.txt first, then `.etignore`, then full C/C++ find; no clean_code → scope_roots = null; matching progress.json → recovery before scope regeneration.
4. **macro.json** — on-demand only.
5. **codegraph** — prefer if `.codegraph` exists; fallback tools otherwise.
6. **文件存在性检查** — double-method rule.

- [ ] **Step 2: Write the clean_code.txt hard-boundary rules**

Include exactly:

````text
### clean_code.txt 优先，且是硬边界

双方法检查 <project_dir>/.ethunter_out/clean_code.txt 是否存在。

存在时：
1. 逐行读取，去掉首尾空白；忽略空行和以 # 开头的注释行。
2. 以 / 开头为绝对路径，否则拼接为 <project_dir>/<行内容>。
3. 路径去重、规范化（分隔符统一为 /，去掉前导 ./）。
4. 普通文件加入 scope；文件夹递归展开为其中全部常规文件；不存在的路径忽略。
5. 生成 tmp/scope_files.txt（展开后的绝对文件路径，去重、排序）和 tmp/scope_roots.txt（原始文件/文件夹条目，去重、排序）。
6. scope 为空 → 报错 empty_scope 并结束。

硬边界规则：
- clean_code.txt 存在时，不得分析或搜索列表以外的源文件。
- 入口、Bind、Dispatch、接口函数定义、宏定义的查找都只能在 scope 内进行。
- 范围外不搜索；若已知目标在范围外，记入 `unresolved`，不输出、不臆造。
- 搜索方式见下方 `### 搜索纪律（适用于所有范围分支）`。
````

- [ ] **Step 3: Write the fallback scope rules**

Include exactly:

````text
### clean_code.txt 不存在

1. `.ethunter_out/.etignore` 存在：按下方 find 命令收集 C/C++ 文件，再按 `.etignore` 规则排除；`.etignore` 语法与 `.gitignore` 一致。
2. `.etignore` 也不存在：直接使用下方 find 命令收集 C/C++ 文件。
3. 两种情况的 find 结果都规范化为绝对路径、去重、排序后写入 `scope_files.txt`；结果为空 → 报错 `empty_scope` 并结束。

C/C++ 扩展名：
.c .h .cpp .cxx .cc .hpp .hxx .hh .inl .ipp

```bash
find <project_dir> -type f \( -name "*.c" -o -name "*.h" -o -name "*.cpp" -o -name "*.cxx" -o -name "*.cc" -o -name "*.hpp" -o -name "*.hxx" -o -name "*.hh" -o -name "*.inl" -o -name "*.ipp" \)
```

无 clean_code.txt 时，`scope_roots = null`；不使用 `scope_roots.txt`。若存在旧的 `scope_roots.txt`，删除或忽略，避免陈旧状态被误用。
````

- [ ] **Step 4: Write the global search-discipline subsection**

Include exactly:

````text
### 搜索纪律（适用于所有范围分支）

- 所有代码搜索只允许在 `scope_files.txt` 列出的文件内进行；`scope_roots.txt` 非 null 时才允许按根目录搜索。
- `grep -I` 必须保留，跳过二进制文件。
- 两种搜索位置模板（文件参数必须来自 scope 清单；允许附加 `-w`、`-E`、`-F`、`-H`、`--` 等 grep 参数）：

其中 <out> = <project_dir>/.ethunter_out/hal-service-api-finder。

```bash
# 方式一：精确文件清单（-H 保证输出带文件名）
xargs -r -a "<out>/tmp/scope_files.txt" -d '\n' grep -I -H -n -F "<pattern>"

# 方式二：原始根目录，仅 scope_roots.txt 非 null 时使用
xargs -r -a "<out>/tmp/scope_roots.txt" -d '\n' grep -I -rn -F "<pattern>"
```

- 需要正则/词边界时，把 `-F` 换成 `-E` 并追加 `-w` 等参数，但文件参数仍必须来自上述清单。
- 禁止 `grep -rn <project_dir>`、`rg <project_dir>`、`find <project_dir> -exec grep` 等任何等价的全仓搜索。
- `xargs`/`grep` 退出码 1 或 123 表示“没有匹配”，不是工具失败；权限/读取错误应如实报告。
- codegraph 返回的每个文件都必须按 `scope_files.txt` 过滤；范围外的搜索命中直接丢弃，不读取、不记录；若分析中已明确知道某个目标定义在范围外，记入 `unresolved`。
````

- [ ] **Step 5: Write macro.json, codegraph, and existence-check rules**

Include exactly:

```text
### macro.json

- 路径：<project_dir>/.ethunter_out/macro.json。
- 按需查阅，不预先全量读取。
- 仅代表编译指令中 -D 等动态定义的宏，不是全部宏。
- 遇到 #ifdef / #if：先查代码，再查 macro.json；两处都没有定义，默认分析该分支。
- `macro.json` 不存在是正常情况，继续分析。
- `#ifdef` 无法判断时分析受保护分支；`#if/#else` 无法判断时两个分支都分析（宁可多分析，不可漏）。

### codegraph

- 双方法检查 <project_dir>/.codegraph 是否存在。
- 存在时优先使用 mcp__codegraph__codegraph_explore。
- 未安装/未配置 MCP → 使用 grep、find、Read、ast-grep，不中断分析。
- codegraph 结果必须按 scope_files.txt 过滤。

### 文件存在性检查

关键文件/目录的存在性判断一律双方法交叉确认，禁止 test -f / [ -f ]：
- 方法一：ls <path>
- 方法二：Read <path>（目录返回目录类错误也视为存在）

两次都确认存在 → 存在。
两次都确认不存在 → 不存在，按规则跳过/新建。
两次不一致 → 用 find 等第三种方法确认，取多数。
两次都无法得出确定结论 → 立即停止并报告，不得假设缺失后继续。
```

- [ ] **Step 6: Verify scope rules**

Run:

```bash
grep -n 'clean_code.txt' hal-service-api-finder/SKILL.md
grep -n 'scope_files.txt' hal-service-api-finder/SKILL.md
grep -n 'scope_roots.txt' hal-service-api-finder/SKILL.md
grep -n 'xargs -r -a' hal-service-api-finder/SKILL.md
grep -n '禁止.*grep -rn <project_dir>' hal-service-api-finder/SKILL.md
grep -n '.etignore' hal-service-api-finder/SKILL.md
grep -n '.codegraph' hal-service-api-finder/SKILL.md
grep -n 'macro.json' hal-service-api-finder/SKILL.md
grep -n '双方法' hal-service-api-finder/SKILL.md
grep -n '搜索纪律' hal-service-api-finder/SKILL.md
grep -n '退出码 1 或 123' hal-service-api-finder/SKILL.md
grep -n '\-H' hal-service-api-finder/SKILL.md
grep -n 'empty_scope' hal-service-api-finder/SKILL.md
grep -n 'scope_roots = null' hal-service-api-finder/SKILL.md
grep -n 'service_name.*不得包含' hal-service-api-finder/SKILL.md
grep -n 'unresolved' hal-service-api-finder/SKILL.md
grep -n '范围外的搜索命中' hal-service-api-finder/SKILL.md
```

Expected: every command prints at least one matching line.

- [ ] **Step 7: Commit**

```bash
git add hal-service-api-finder/SKILL.md
git commit -m "docs(hal-service-api-finder): add initialization and scope rules"
```

---

### Task 4: Write the task-recovery mechanism

状态文件路径：
- `<out>/progress.json`
- `<out>/tmp/entry_state.json`
- `<out>/tmp/trace_state.json`

以下 JSON 为字段形状示例，不要求彼此处于同一时刻。

**Files:**
- Modify: `hal-service-api-finder/SKILL.md` (section 四)

- [ ] **Step 1: Write the progress.json schema and phase rules**

Include this exact JSON block under a `### progress.json` heading:

```json
{
  "phase": "locate_entry",
  "service_name": "ar_service",
  "project_dir": "/abs/path",
  "scope": {
    "file_count": 0,
    "clean_code": true,
    "scope_files": "tmp/scope_files.txt",
    "scope_roots": "tmp/scope_roots.txt"
  },
  "locate_entry": {"status": "in_progress"},
  "analyze_bind": {"status": "pending"},
  "trace_dispatch": {"status": "pending"},
  "output": {"status": "pending"},
  "error": null
}
```

Add these rules:

- `phase` 取值：`locate_entry` / `analyze_bind` / `trace_dispatch` / `output` / `done`。
- 阶段状态：`pending` / `in_progress` / `completed`。
- 每次阶段切换立即保存。
- 全新分析开始时，先创建初始 progress.json：写入 service_name/project_dir/scope，phase = locate_entry，locate_entry.status = in_progress，其余阶段 status = pending，error = null；写入方式为 .tmp + mv。
- 失败时不新增 failed 阶段；保留当前 phase，写入 `error: {code, message, evidence}`，下次运行从当前阶段重试。
- 重试前清空 error。
- phase 是恢复的唯一权威；各阶段 `status` 仅作诊断，必须与 `phase` 同步更新。
- `scope.file_count` 在 scope 生成后写入；恢复时与 `scope_files.txt` 的行数核对，不一致以实际文件清单为准并更新。
- 只有 `trace_state.status == completed` 后，`phase` 才能设为 `output`；`output` 完成后才能设为 `done`。

- [ ] **Step 2: Write the entry_state.json schema**

Include this exact JSON block under a `### entry_state.json` heading:

```json
{
  "service_name": "ar_service",
  "scope": {
    "file_count": 123,
    "clean_code": true,
    "scope_files": "tmp/scope_files.txt",
    "scope_roots": "tmp/scope_roots.txt"
  },
  "entry": {
    "variable": "g_ArDriverEntry",
    "file": "/abs/path/...",
    "line": 39,
    "module_name": "ar_service",
    "bind": "HdfArDriverBind",
    "init": "HdfArDriverInit",
    "release": "HdfArDriverRelease",
    "registration": {
      "form": "HDF_INIT",
      "file": "/abs/path/...",
      "line": 50,
      "evidence": "HDF_INIT(g_ArDriverEntry)"
    }
  },
  "bind": {
    "status": "completed",
    "function": "HdfArDriverBind",
    "file": "/abs/path/...",
    "line": 72,
    "host_type": "HdfArHost",
    "dispatch_functions": [
      {
        "name": "ArDriverDispatch",
        "file": "/abs/path/...",
        "line": 129,
        "via": "hdfArHost->ioService.Dispatch"
      }
    ],
    "stub": {
      "member": "stub",
      "interface_type": "IActivityInterface",
      "impl_class": "ArService",
      "impl_file": "/abs/path/...",
      "stub_class": "ActivityInterfaceStub",
      "on_remote_request": {
        "name": "OHOS::HDI::ActivityRecognition::V1_0::ActivityInterfaceStub::OnRemoteRequest",
        "file": "/abs/path/...",
        "line": 160
      },
      "evidence": "hdfArHost->stub = ...GetOrNewObject(serviceImpl, IActivityInterface::GetDescriptor())"
    },
    "service_registration": {
      "file": "/abs/path/...",
      "line": 110,
      "evidence": "deviceObject->service = &hdfArHost->ioService"
    }
  }
}
```

Add these rules:

- `registration.form`：`HDF_INIT` / `other` / `unverified`。
- `bind.status`：`pending` / `in_progress` / `completed`；阶段二开始时若已有 bind 对象，必须先将 `bind.status` 置为 `in_progress`，失败时保持该值。
- `stub` 可以为 `null`；`host_type` 可以为 `null`；`service_registration` 的 `file`/`line`/`evidence` 可以为 `null`；`stub.on_remote_request` 可以为 `null`。
- 阶段二可重复执行（幂等）：中断后重跑并覆盖。
- 恢复时校验 `entry_state.service_name` 与当前 `service_name` 一致；不一致视为身份不一致，报 `state_conflict` 并停止，不混用旧状态。

- [ ] **Step 3: Write the trace_state.json schema**

Include this exact JSON block under a `### trace_state.json` heading:

```json
{
  "status": "in_progress",
  "service_name": "ar_service",
  "entry_dispatchers": [
    {"name": "ArDriverDispatch", "file": "/abs/path/...", "line": 129}
  ],
  "queue": [
    {
      "id": 1,
      "dispatcher": "ArDriverDispatch",
      "dispatcher_file": "/abs/path/...",
      "dispatcher_line": 129,
      "current_cmd_expr": "cmdId",
      "current_cmd_value": null,
      "outer_cmd_expr": null,
      "outer_cmd_value": null,
      "data_vars": ["data"],
      "data_derived": true,
      "parent_case_id": null,
      "path": ["ArDriverDispatch"],
      "status": "pending"
    }
  ],
  "cases": [
    {
      "id": 1,
      "dispatcher": "OHOS::HDI::ActivityRecognition::V1_0::ActivityInterfaceStub::OnRemoteRequest",
      "dispatcher_file": "/abs/path/...",
      "dispatcher_line": 160,
      "dispatch_site": {"file": "/abs/path/...", "line": 163},
      "level": 2,
      "case_expr": "CMD_ACTIVITY_INTERFACE_GET_VERSION",
      "case_value": 0,
      "outer_cmd_expr": "CMD_ACTIVITY_INTERFACE_GET_VERSION",
      "outer_cmd_value": 0,
      "parent_case_id": null,
      "status": "analyzed",
      "handlers": [
        {
          "name": "ActivityInterfaceStubGetVersion",
          "file": "/abs/path/...",
          "line": 1200
        }
      ],
      "evidence": "case CMD_ACTIVITY_INTERFACE_GET_VERSION: return ActivityInterfaceStubGetVersion(data, reply, option);"
    }
  ],
  "handlers": [
    {
      "cmd_id_macro": "CMD_ACTIVITY_INTERFACE_GET_VERSION",
      "cmd_id_value": 0,
      "interface_function": "ActivityInterfaceStubGetVersion",
      "file": "/abs/path/...",
      "definition_line": 1200
    }
  ],
  "default_cases": [],
  "unresolved": [],
  "dispatch_cache": [],
  "visited": [],
  "counters": {
    "queue_remaining": 1,
    "cases_total": 1,
    "handlers_total": 1,
    "unresolved_total": 0
  }
}
```

Add these rules:

- `trace_state.status`：`in_progress` / `self_check` / `completed`。
- `handlers` 是最终候选池；`definition_line` 仅用于调试，最终 JSON 不输出。
- `default_cases` 不进入最终 JSON。
- `unresolved` 条目包含 `kind`、`detail`、`tried`。
- `visited` 条目包含函数定义信息和 cmd 上下文，并带 `data_vars`、`data_derived` 字段，用于重建环检测键。
- `dispatch_cache` 条目包含 `site`、`target`、`candidates`、`resolved`。
- `counters` 用于恢复时的自洽检查。
- `queue` 只保存在处理中的工作项：处理前将该项置为 `in_progress`，处理完从 `queue` 移除；`queue_remaining` 必须等于 `queue` 的长度。
- cmdId 用例状态为 `pending` / `analyzed`；`default` 分支单独记入 `default_cases`。
- `trace_state.status` 含义：`in_progress` → 继续追踪；`self_check` → 完成或重做自检；`completed` → 可组装输出。
- `trace_state 完整` 的定义：`status == completed` 且 `queue` 为空，且 `counters` 与实际数组长度一致。
- 恢复时校验 `trace_state.service_name` 与当前 `service_name` 一致；不一致视为身份不一致，报 `state_conflict` 并停止。

- [ ] **Step 4: Write the resume flow**

Include these exact blocks under the following headings:

```text
### 状态文件问题分类

- 缺失：文件不存在 → 按恢复流程重建对应阶段。
- 不可解析：文件存在但 JSON 解析失败 → 报 `state_io_error`，停止，不自动清空。
- 身份不一致：JSON 可解析，但 service_name/project_dir 与当前调用不一致 → 报 `state_conflict`，停止，不混用旧状态。
- 内容不完整：JSON 可解析、身份一致，但缺少该阶段必需字段 → 视为该阶段未完成，按恢复流程重跑对应阶段。

“有效 entry_state”：JSON 可解析、service_name 匹配、entry 存在且 module_name 匹配。
“有效 trace_state”：JSON 可解析、service_name 匹配、status 为 in_progress/self_check/completed、queue/cases/handlers/default_cases/unresolved/dispatch_cache/visited 均为数组。

### 恢复流程

1. 双方法检查 progress.json：不存在 → 全新分析；存在 → 继续。
2. 校验 project_dir/service_name 与当前调用一致；不一致 → 报 state_conflict；progress.json 不可解析 → 报 state_io_error；均停止，不删除旧状态。
3. phase = done → 双方法确认最终输出存在：
   - 存在 → 报告完成，停止；
   - 不存在且 trace_state 完整（status == completed、queue 为空、counters 与实际数组一致）→ 从 trace_state 重新组装输出；
   - 不存在且 trace_state 不完整 → 回到 trace_dispatch，按 trace_state.status 继续。
4. 按 phase 恢复（先按“状态文件问题分类”处理当前阶段依赖的状态文件）：
   - locate_entry：entry_state 有效且 entry 存在 → 跳到 analyze_bind；entry_state 缺失或内容不完整 → 重跑阶段一；不可解析 → state_io_error；身份不一致 → state_conflict。
   - analyze_bind：entry_state 有效且 bind.status == completed 且 dispatch_functions 非空 → 跳到 trace_dispatch；entry_state 有效但 bind 未完成 → 重跑阶段二；entry_state 缺失或内容不完整（无 entry）→ 回到 locate_entry；不可解析 → state_io_error；身份不一致 → state_conflict。
   - trace_dispatch：trace_state 有效 → 按 status 处理：
       in_progress → 将 in_progress 工作项置回 pending，从第一个待处理项继续；重做该工作项的全部分支，已 analyzed 的 case 通过去重合并 handlers；queue 空 → 进入自检；
       self_check → 完成或重做自检（见第八章）；
       completed → 进入 output；
     trace_state 缺失或内容不完整 → 若 entry_state 有效且 bind.status == completed 且 dispatch_functions 非空，从 entry_state 重新初始化队列；否则回到 locate_entry；不可解析 → state_io_error；身份不一致 → state_conflict。
   - output：trace_state 有效且 status == completed 才组装输出；status == self_check → 先完成自检；status == in_progress → 回到 trace_dispatch；trace_state 缺失或内容不完整 → 回到 trace_dispatch；不可解析 → state_io_error；身份不一致 → state_conflict。成功后置 done。
5. 加载 trace_state 后重新计算 counters（queue_remaining = queue.length；cases_total = cases.length；handlers_total = handlers.length；unresolved_total = unresolved.length）；与实际不一致时以实际数组为准，并将重算后的 counters 写回 state；用重算结果判断 trace_state 完整。
6. 关键文件存在性一律双方法确认；结果无法判断 → 停止并报告。
7. 状态文件采用“先写 .tmp 再 mv”的方式覆盖，避免中断产生半截 JSON。
8. 存在与当前 project_dir/service_name 匹配的 progress.json 时，按恢复流程继续，不重新生成 scope 或 scope 文件；只有全新分析才重建 scope，除非恢复步骤明确要求重建。若 progress.json 匹配但 scope_files.txt 缺失，重建 scope 并更新 progress.scope。无 clean_code.txt 时 scope_roots = null，旧的 scope_roots.txt 删除或忽略。
```

Add a `### 全新分析与旧输出` subsection with:

- progress.json 不存在 → 全新分析；开始时若同 service 的旧输出存在，先重命名为 `<service_name>_api.json.bak`，完成时再原子覆盖最终文件。
- progress.json 存在但 phase 未完成 → 严格按恢复流程继续，不从头重跑。
- 强制全新分析：删除 progress.json 与 tmp/。
- 完成态 state 保留，不删除。
- 同一 project_dir 同时只保留一个 service 的状态；检测到不同 service 的 progress.json → 报 `state_conflict` 并停止，提示用户归档或删除旧 `progress.json` 与 `tmp/` 后重跑；skill 不自动删除、不继续分析。

- [ ] **Step 5: Verify state fields and JSON blocks**

Run:

```bash
grep -n 'progress.json' hal-service-api-finder/SKILL.md
grep -n 'entry_state.json' hal-service-api-finder/SKILL.md
grep -n 'trace_state.json' hal-service-api-finder/SKILL.md
grep -n 'state_conflict' hal-service-api-finder/SKILL.md
grep -n 'self_check' hal-service-api-finder/SKILL.md
grep -n 'queue' hal-service-api-finder/SKILL.md
grep -n 'unresolved' hal-service-api-finder/SKILL.md
grep -n 'trace_state 完整' hal-service-api-finder/SKILL.md
grep -n 'service_name 与当前' hal-service-api-finder/SKILL.md
```

Expected: every command prints at least one matching line.

Validate every fenced JSON block:

```bash
python3 - <<'PY'
import json, re, pathlib
text = pathlib.Path("hal-service-api-finder/SKILL.md").read_text(encoding="utf-8")
blocks = re.findall(r"```json\n(.*?)\n```", text, re.S)
for i, block in enumerate(blocks, 1):
    json.loads(block)
print(f"{len(blocks)} JSON blocks valid")
PY
```

Expected: prints `N JSON blocks valid` with no exception.

- [ ] **Step 6: Commit**

```bash
git add hal-service-api-finder/SKILL.md
git commit -m "docs(hal-service-api-finder): add state files and resume flow"
```

---

### Task 5: Write Phase 1 — locate the service registration point

**Files:**
- Modify: `hal-service-api-finder/SKILL.md` (section 五)

- [ ] **Step 1: Write the exact search procedure**

Include these subsections:

1. **精确搜索** — search `"<service_name>"` with `grep -F` in scope_files; read at least 30 lines of context; accept only `.moduleName` in a `HdfDriverEntry` initializer; extract variable, file, initializer declaration line, Bind/Init/Release.
2. **注册形式确认** — prefer `HDF_INIT(<entry_var>)`; also check `HDF_INIT(&<entry_var>)`; `HDF_INIT` must be a live executable statement, not a comment/string/log; fallback to other live references; record `registration.form` as `HDF_INIT`, `other`, or `unverified`; multiple evidence locations use the first in scope_files order.
3. **服务存在判定** — a moduleName-matched `HdfDriverEntry` means the service is found; registration reference is evidence, not a hard gate.
4. **多命中处理** — dedupe by `(file, initializer declaration line)`; unique candidate → use it; multiple candidates with exactly one registered → use it; multiple registered or none registered → `multiple_matches`, list candidates, exit.
5. **失败处理** — `service_not_found` with scope file count, searched patterns, and a spelling/clean_code suggestion; `bind_not_found` when `.Bind` is missing/empty/NULL.
6. **阶段产物** — write `entry` and `scope` to `tmp/entry_state.json`; proceed to `analyze_bind`.

Include the sample code block from rr1:

```c
struct HdfDriverEntry g_ArDriverEntry = {
    .moduleVersion = 1,
    .moduleName = "ar_service",
    .Bind = HdfArDriverBind,
    .Init = HdfArDriverInit,
    .Release = HdfArDriverRelease,
};

HDF_INIT(g_ArDriverEntry);
```

- [ ] **Step 2: Verify Phase 1 content**

Run:

```bash
grep -n 'moduleName' hal-service-api-finder/SKILL.md
grep -n 'HDF_INIT' hal-service-api-finder/SKILL.md
grep -n 'service_not_found' hal-service-api-finder/SKILL.md
grep -n 'multiple_matches' hal-service-api-finder/SKILL.md
grep -n 'bind_not_found' hal-service-api-finder/SKILL.md
grep -n 'unverified' hal-service-api-finder/SKILL.md
grep -n 'entry_state.json' hal-service-api-finder/SKILL.md
```

Expected: every command prints at least one matching line.

- [ ] **Step 3: Commit**

```bash
git add hal-service-api-finder/SKILL.md
git commit -m "docs(hal-service-api-finder): add registration-point phase"
```

---

### Task 6: Write Phase 2 — analyze Bind

**Files:**
- Modify: `hal-service-api-finder/SKILL.md` (section 六)

- [ ] **Step 1: Write the Bind analysis rules**

Include these subsections:

1. **定位 Bind 定义** — search scope; prefer same file as entry; require exact identifier equality; dedupe candidates by `(file, signature line)`; resolve macro/wrapper `.Bind` values to the real function with a visited set and cycle detection; if a qualified `Class::Method` is not found, search the class body for the unqualified method; verify a full body; handle multiple definitions:
   - signature matching `int32_t (*)(struct HdfDeviceObject *)` first;
   - still ambiguous → `bind_ambiguous`, list candidates, exit;
   - no body → `bind_definition_not_found`, exit.
2. **提取 Host 与 Dispatch** — identify `new <HostType>`, pointer/stack/static, `sptr`/`std::unique_ptr`/`std::make_unique`/factory instances; accept `host->ioService.Dispatch = X`, `host->ioService = { .Dispatch = X }`, `host.ioService.Dispatch = X`, `host->Dispatch = X`; if absent, follow initialization functions with a visited set and search scope for Dispatch assignments; keep all Dispatch functions; a Dispatch whose definition is outside scope gets `file`/`line = null` and is recorded as `unresolved` by phase 3 instead of generating a work item; if all Dispatch entries are skipped and `stub.on_remote_request` is non-null, phase 3 seeds from that OnRemoteRequest; none → `dispatch_not_found`, exit.
3. **提取 stub、接口与实现类（可选）** — find stub-like member assignment; trace `GetOrNewObject(serviceImpl, IActivityInterface::GetDescriptor())` or `new XxxStub`; record stub member, interface type, impl class (class body required), stub class; search both `<interface_type>Stub` and the leading-`I`-stripped name; locate `OnRemoteRequest` with exact-stub-first/nearest-base precedence; missing is not fatal.
4. **服务注册证据** — find `deviceObject->service = &host->ioService` or equivalent; record evidence only; missing fields may be null.
5. **阶段产物** — set `bind.status = in_progress` at phase start; write `bind` to `tmp/entry_state.json`; proceed to `trace_dispatch`.

Include the sample snippets from rr1:

```c
hdfArHost->ioService.Dispatch = ArDriverDispatch;
hdfArHost->ioService.Open = nullptr;
hdfArHost->ioService.Release = nullptr;

ArService *impl = new (std::nothrow) ArService();
...
sptr<IActivityInterface> serviceImpl = impl;
hdfArHost->stub = OHOS::HDI::ObjectCollector::GetInstance().GetOrNewObject(serviceImpl,
    IActivityInterface::GetDescriptor());
...
deviceObject->service = &hdfArHost->ioService;
```

- [ ] **Step 2: Verify Phase 2 content**

Run:

```bash
grep -n 'ioService.Dispatch' hal-service-api-finder/SKILL.md
grep -n 'GetOrNewObject' hal-service-api-finder/SKILL.md
grep -n 'OnRemoteRequest' hal-service-api-finder/SKILL.md
grep -n 'dispatch_not_found' hal-service-api-finder/SKILL.md
grep -n 'bind_ambiguous' hal-service-api-finder/SKILL.md
grep -n 'bind_definition_not_found' hal-service-api-finder/SKILL.md
grep -n 'deviceObject->service' hal-service-api-finder/SKILL.md
```

Expected: every command prints at least one matching line.

- [ ] **Step 3: Commit**

```bash
git add hal-service-api-finder/SKILL.md
git commit -m "docs(hal-service-api-finder): add Bind analysis phase"
```

---

### Task 7: Write Phase 3 — cmdId dispatch tracing

**Files:**
- Modify: `hal-service-api-finder/SKILL.md` (section 七)

- [ ] **Step 1: Write queue initialization and dispatch-structure recognition**

Include:

- Initial work item for each Dispatch function with a non-null definition file, with `current_cmd_expr`, `current_cmd_value = null`, `outer_cmd_expr/value = null`, `data_vars` (target parameter names), `data_derived = true`, `parent_case_id = null`, `path`, `status = pending`. Skip `dispatch_functions` entries with `file == null` and record them in `trace_state.unresolved` (`kind = "definition"`); if all entries are skipped and `bind.stub.on_remote_request` is non-null, seed a work item from that OnRemoteRequest.
- If Dispatch directly calls `stub->SendRequest(cmdId, ...)`, enqueue the corresponding stub's `OnRemoteRequest`, with outer still null.
- Recognize: switch/case, range/mask/if-else, array/map/function-pointer tables, table registration, macro-generated dispatch, `SendRequest`/`OnRemoteRequest` forwarding, unconditional direct data handling.

- [ ] **Step 2: Write the outer-cmd and value-resolution rules**

Include exactly:

```text
外层归属：
effective_outer = 当前分支已确定外层 ?? 工作项.outer ?? 本用例

- 内层 case 继承其外层 case 的 effective_outer，不得用内层 case 覆盖外层。

- 工作项外层为空 → 本用例成为外层，记录宏/表达式和值。
- 外层已存在 → 保留外层，本用例只作内层证据。
- Dispatch → SendRequest → OnRemoteRequest 时，OnRemoteRequest 的 switch case 成为外层。
- 嵌套分发继承 effective_outer。
```

Include the value-resolution table exactly:

| 形式 | 处理 |
|------|------|
| 整数字面量 | 直接取值 |
| `#define` 宏链 | 在 scope 内递归解析，支持括号和简单运算符 |
| 枚举常量 | 按显式值或隐式递增规则解析 |
| `const` / `constexpr` 变量 | 读取初始化值 |
| macro.json 中的 -D 宏 | 作为辅助来源 |
| 算术/移位/位运算 | 常量可计算则求值 |
| 范围 `A ... B` | 表达式原文，值记 `null` |
| 掩码/组合表达式 | 表达式原文；能算出单一值则记值，否则 `null` |
| 同名宏多定义/条件编译冲突 | 值记 `null`，写入 `unresolved` 并列出候选 |
| 无法解析 | 值记 `null`，写入 `unresolved` |

Add: 具名 cmdId 即使值无法解析，也必须保留在最终输出中（值为 `null`）。

- [ ] **Step 3: Write the data-tracking and callee-classification rules**

Include the data-derived table exactly:

| 形式 | 是否 data 派生 |
|------|----------------|
| 直接参数 `data` | 是 |
| `&data`、`*data`、cast | 是 |
| `data.member`、`data->member` | 是 |
| `data.ReadXxx()` 的返回值或输出参数 | 是 |
| `HdfSbufReadXxx(data, &out)` 等以 data 为输入并写出结果的框架读取/转换函数 | 是 |
| `SbufToParcel(data, &parcel)` 后的 `parcel` | 是 |
| 上述变量的简单赋值/别名 | 是 |
| 被非 data 值覆盖 | 否，移出派生集合 |
| `reply` 及其派生值 | 否 |
| 写入全局/成员后在另一个函数读取 | 记 `unresolved`（`data_deferred`），不臆造跨函数传播 |

Before the table, state the ordered evaluation (forwarder → dispatcher → exclusions including 无业务动作 → generic data consumer → ignore), so the rows are mutually exclusive. Include the table exactly:

| 目标特征 | 处理 |
|----------|------|
| 已知转发器（`SendRequest`、`OnRemoteRequest`、已知 dispatcher） | 入队为新工作项 |
| 接收 cmdId 且函数体内有分发逻辑 | 入队为内层 dispatcher |
| 接收 cmdId + data，且函数体内有分发 | 先作为分发函数入队，不直接记为接口 |
| 接收 data/派生参数，且不构成分发 | 记为接口函数候选 |
| 纯日志、断言、框架 sbuf/parcel 转换、纯校验辅助、无业务动作 | 忽略，写入 evidence |
| 只接收 cmdId、不接收 data、也不分发 | 忽略 |
| 只接收 reply、不读 data | 不记为接口函数 |

Add these branch rules:

- 一个分支有多个 data 消费目标 → 多条记录。
- 一个分支只有内联 data 处理、没有调用目标 → 把分发函数自身记为接口函数，附 evidence。
- `default` 分支只写入 `trace_state.default_cases`，不进入最终 JSON。
- 无 cmdId 分支、无条件直接调用 data 消费函数 → 记为 `cmd_id_macro = "*"`、`cmd_id_value = null`，避免漏接口；evidence 说明“无条件直接处理”。

- [ ] **Step 4: Write C/C++ target resolution and termination rules**

Include:

- C++ 间接调用、虚函数、函数指针、重载等解析细节可参考 taint-path-walker skill；本 skill 只追踪到 cmdId 处理位置，不要求完整污点传播路径。
- Support direct/member/`this->` calls, virtual functions, function pointers, table entries, registered callbacks.
- Statically determinable → exact target; otherwise enumerate all identifiable candidates in scope; no count limit; still unresolved → `unresolved` with attempted searches.
- `stub->SendRequest` special handling: use phase-2 interface/impl/stub info to locate `Stub::OnRemoteRequest`; map cmdId arg → `code`, data/parcel → data params; keep outer cmd; not found → `unresolved`.
- Qualified names; overloads by file+line; lambdas `<lambda@绝对路径:行号>`.
- Interface function definition must be in scope; otherwise `unresolved`, not output.
- 不设深度上限、不设扇出上限。
- Termination conditions: direct business handler found / no data target / unresolved / cycle.
- Cycle key: `(函数定义文件+行号, current_cmd_expr, current_cmd_value, outer_cmd_expr, outer_cmd_value, sorted(data_vars), data_derived)`. A queue hit or non-ancestor visited hit is silent dedup; only an ancestor visited hit is a `cycle`. Different data contexts produce different keys and are enqueued separately.
- Functions that both dispatch and process data → follow as dispatcher.
- Save `trace_state` after each work item; list all fields; idempotency/dedup rules.

- [ ] **Step 5: Verify Phase 3 content**

Run:

```bash
grep -n 'effective_outer' hal-service-api-finder/SKILL.md
grep -n 'SendRequest' hal-service-api-finder/SKILL.md
grep -n 'OnRemoteRequest' hal-service-api-finder/SKILL.md
grep -n 'data_deferred' hal-service-api-finder/SKILL.md
grep -n 'cycle' hal-service-api-finder/SKILL.md
grep -n 'default_cases' hal-service-api-finder/SKILL.md
grep -n '不设深度上限' hal-service-api-finder/SKILL.md
grep -n '不设扇出上限' hal-service-api-finder/SKILL.md
grep -n 'taint-path-walker' hal-service-api-finder/SKILL.md
```

Expected: every command prints at least one matching line.

- [ ] **Step 6: Commit**

```bash
git add hal-service-api-finder/SKILL.md
git commit -m "docs(hal-service-api-finder): add cmdId dispatch tracing"
```

---

### Task 8: Write output JSON and self-check

**Files:**
- Modify: `hal-service-api-finder/SKILL.md` (section 八)

- [ ] **Step 1: Write the final JSON schema and output rules**

Include this exact JSON block:

```json
[
  {
    "cmd_id_macro": "CMD_ACTIVITY_INTERFACE_GET_VERSION",
    "cmd_id_value": 0,
    "interface_function": "ActivityInterfaceStubGetVersion",
    "file": "/abs/path/activity_interface_stub.cpp"
  }
]
```

Include the field table exactly:

| 字段 | 类型 | 规则 |
|------|------|------|
| `cmd_id_macro` | string | 最外层具体 case 的宏名或表达式原文；范围/掩码/组合表达式照原样 |
| `cmd_id_value` | integer 或 null | 单一可解析值；范围/掩码/组合/无法解析时为 null |
| `interface_function` | string | 定义处的作用域限定名；lambda 用 `<lambda@绝对路径:行号>` |
| `file` | string | 函数定义所在文件的绝对路径，必须在 scope_files 内 |

Add output rules:

1. 同一 cmdId 对应多个接口函数 → 多条记录。
2. 完全相同的四元组去重。
3. 排序：`cmd_id_value` 升序（null 最后）→ `cmd_id_macro` → `interface_function` → `file`。
4. `default` 分支不进入最终 JSON。
5. 服务存在但未识别到接口 → 输出 `[]`，报告 0 个接口。
6. 不添加任何额外字段。
7. UTF-8 编码，JSON 缩进 2 空格。
8. 先写 `tmp/<service_name>_api.json.tmp`，Read 回读校验通过后 `mv` 为最终文件。
9. 最终文件必须存在，否则本次任务失败。

Add notes:
- `cmd_id_value` comes from `trace_state.handlers` without re-evaluation; computable masks/combinations keep integers, only ranges/uncomputable/unparseable are null.
- `cmd_id_macro` may be `"*"` for unconditional/inline handling and is output normally; only `default` is excluded.
- Four-tuple dedupe is the final normalization; overloads identical in all four fields collapse.

- [ ] **Step 2: Write the self-check checklist and completion report**

Include exactly these self-check items:

1. `queue` 为空，没有 `pending` / `in_progress` 工作项。
2. 所有 `cases` 已 `analyzed`，或明确记录无目标/unresolved。
3. 每个 `handlers` 条目都能在 scope_files 内定位到函数定义，且函数名为限定名。
4. `default_cases` 没有混入 `handlers`。
5. handlers 已按四元组去重，排序规则已执行。
6. `unresolved` 已记录，数量与报告一致。
7. 最终 JSON 恰好包含四个字段，无额外字段。
8. 写临时文件后 Read 回读，核对数组长度、字段名、类型。
9. `mv` 后双方法确认最终文件存在。
10. 写最终 JSON 前将 `progress.phase` 置为 `output`；写成功后置为 `done`，所有阶段状态为 `completed`。

Add operational notes:
- Item 2 requires every `cases[i].status == "analyzed"`; item 4 requires no `default` handler and no handler backed solely by `default_cases`.
- Read the final JSON to EOF; a truncated read is a failure.
- Self-check failure uses `code = "self_check_failed"`; cmdId case count is `cases.length` excluding `default_cases`.

Include the completion report fields:

- 分析范围文件数；
- service_name、入口变量、Bind 函数、Dispatch 函数数量；
- cmdId 用例数；
- 接口函数数量；
- unresolved 数量；
- 最终输出文件绝对路径。

Add: 自检开始前将 `trace_state.status` 置为 `self_check`；全部通过后置为 `completed`，并将 `progress.phase` 置为 `output`；最终 JSON 写成功后将 `progress.phase` 置为 `done`。

- [ ] **Step 3: Verify output content**

Run:

```bash
grep -n 'cmd_id_macro' hal-service-api-finder/SKILL.md
grep -n 'cmd_id_value' hal-service-api-finder/SKILL.md
grep -n 'interface_function' hal-service-api-finder/SKILL.md
grep -n '<service_name>_api.json' hal-service-api-finder/SKILL.md
grep -n 'self_check' hal-service-api-finder/SKILL.md
grep -n 'phase = done' hal-service-api-finder/SKILL.md
```

Expected: every command prints at least one matching line.

- [ ] **Step 4: Commit**

```bash
git add hal-service-api-finder/SKILL.md
git commit -m "docs(hal-service-api-finder): add output schema and self-check"
```

---

### Task 9: Write permissions, constraints, anti-rationalization, and examples

**Files:**
- Modify: `hal-service-api-finder/SKILL.md` (sections 九 through 十二)

- [ ] **Step 1: Write the permissions table**

Include exactly:

| 工具 | 用途 |
|------|------|
| `Bash` | find/grep/xargs/ls/mkdir/mv/rm，scope 内搜索与状态落盘 |
| `Read` | 读取源码、macro.json、状态文件 |
| `Write` | 写入 progress.json、entry_state.json、trace_state.json、最终 JSON |
| `mcp__codegraph__codegraph_explore` | （可选）.codegraph 存在时优先的代码探索工具 |
| `mcp__plugin_oh-my-claudecode_t__ast_grep_search` | （可选）AST 模式搜索 |

Add: codegraph 未安装/未配置 → 用常规工具代替，不中断分析。

- [ ] **Step 2: Write the ten constraints**

Include exactly these ten numbered constraints:

1. 全部由主 agent 完成，不调用 sub-agent。
2. 质量优先；不设深度/扇出上限；所有分支追到接口函数或明确终止条件。
3. clean_code.txt 是硬边界；grep 只用 scope_files/scope_roots；codegraph 结果按 scope 过滤；禁止全仓 grep。
4. 只输出直接目标业务函数（含 3.6 的 `*` 无条件/内联处理记录），不展开业务函数内部实现。
5. 接口函数必须能在 scope 内定位到定义；范围外记 unresolved，不输出。
6. 不臆造：无法解析的宏/目标写明尝试过的搜索方式。
7. 输出路径一律绝对路径；函数名带作用域限定；重载用文件+行号区分。
8. 每个工作项处理完立即落盘；最终 JSON 最后一次性原子写出。
9. 关键文件存在性双方法确认，禁止 `test -f`。
10. 全中文提示词，同一概念用词前后一致。

- [ ] **Step 3: Write the global error-code table**

Under a `### 全局错误码` heading, include exactly:

| 错误码 | 触发条件 | 处理 |
|--------|----------|------|
| `usage_error` | 参数缺失、过多、为空，或 service_name 含非法字符 | 打印用法，结束 |
| `invalid_project_dir` | project_dir 不存在或不是目录 | 报告，结束 |
| `output_dir_failed` | 输出目录创建失败 | 报告，结束 |
| `empty_scope` | 分析范围为空 | 报告，结束 |
| `service_not_found` | 找不到 moduleName 精确匹配的 HdfDriverEntry | 报告范围文件数、搜索模式，结束 |
| `multiple_matches` | 多个候选无法唯一确定 | 列出候选（变量、文件、行号），结束 |
| `bind_not_found` | 入口结构体没有有效 `.Bind` | 报告，结束 |
| `bind_definition_not_found` | Bind 函数体不在 scope 内 | 报告，结束 |
| `bind_ambiguous` | 多个同名 Bind 定义无法区分 | 列出候选，结束 |
| `dispatch_not_found` | 找不到 Dispatch 函数 | 报告，结束 |
| `state_conflict` | progress.json 的 project_dir/service_name 或其他状态文件的 service_name 与当前调用不一致 | 报告，结束，不删除旧状态 |
| `state_io_error` | 状态文件读写失败或 JSON 不可解析 | 报告，结束 |
| `output_write_failed` | 临时文件写入、回读校验、mv 或最终文件存在性确认失败 | 报告，保留 phase=output，便于重试 |
| `self_check_failed` | 自检清单第 1-7 项失败 | 报告，`trace_state.status` 保持 `self_check`，`phase` 保持不变，结束，不写最终 JSON |

- [ ] **Step 4: Write the terminology table**

Write a `### 术语表` heading, then include exactly this table:

| 用词 | 含义 |
|------|------|
| 服务注册点 | 通过 moduleName 精确定位到的 `HdfDriverEntry` 定义及其注册引用 |
| 服务入口变量 | `HdfDriverEntry` 类型的变量，如 `g_ArDriverEntry` |
| Bind 函数 | 入口结构体 `.Bind` 字段指向的函数 |
| Dispatch 函数 | Bind 阶段链接到 Host 的 `ioService.Dispatch`（或等价成员）的函数 |
| 分发函数 | 根据 cmdId 选择处理路径的函数 |
| cmdId 用例 | 分发结构中的一个具体 case 或条件分支 |
| 外层 cmd | 调用链上最外层的具体 cmdId 用例的宏/表达式和值 |
| 接口函数 | cmdId 用例分支中直接接收客户端 data（或 data 派生值）的业务函数 |
| 工作项 | 追踪队列中的单元：`(分发函数, 当前 cmdId 变量, 外层 cmd, data 变量, 调用路径)` |
| 中间结果 | `progress.json`、`tmp/entry_state.json`、`tmp/trace_state.json` 等状态文件 |
| scope_files | 分析范围内展开后的精确文件清单 |
| scope_roots | clean_code.txt 中原始的文件/文件夹条目清单 |

- [ ] **Step 5: Write the anti-rationalization table**

Include exactly these 12 rows:

| 想法 | 现实 |
|------|------|
| “分发层级太深，先记到这里” | 不设深度上限，追到接口函数或明确终止条件。 |
| “候选太多，只取前 N 个” | 不设扇出上限，全部枚举。 |
| “宏算不出来，跳过这个 case” | 保留记录，value 记 null。 |
| “default 也算接口” | default 只进 default_cases，不进最终 JSON。 |
| “这个函数像 helper，跳过” | 读函数体确认，不能凭名字判断。 |
| “范围外可能有定义，去全仓 grep 一下” | 禁止；scope 是硬边界。 |
| “短函数名就够了” | 必须带作用域限定名。 |
| “状态最后一起存” | 每个工作项处理完立即落盘。 |
| “有中间结果也从头跑一遍” | 先走恢复检查，按 phase 续跑。 |
| “用 sub-agent 并行快一点” | 禁止；全部由主 agent 完成。 |
| “只收 reply 也算接口” | 纯 reply、不读 data 的函数不算。 |
| “业务函数内部还有处理，继续展开” | 只记直接目标，不展开业务实现。 |

- [ ] **Step 6: Write the usage examples**

Include three examples:

1. **首次分析** — `/hal-service-api-finder /srv/ohos/drivers ar_service`; show initialization, phase 1, phase 2, phase 3, self-check, output; report `127 files / 8 cases / 8 interfaces / 0 unresolved`.
2. **断点续跑** — progress phase `trace_dispatch`, cases 1-3 analyzed, case 4 in_progress; resume and finish.
3. **service 未找到** — `/srv/ohos/drivers ar_service_typo`; report `service_not_found`, scope file count, searched pattern, suggestions.

Use the exact sample names from rr1: `g_ArDriverEntry`, `HdfArDriverBind`, `ArDriverDispatch`, `IActivityInterface`, `ArService`, `ActivityInterfaceStub::OnRemoteRequest`, `ActivityInterfaceStubGetVersion`.

- [ ] **Step 7: Verify sections 九-十二**

Run:

```bash
grep -n 'mcp__codegraph__codegraph_explore' hal-service-api-finder/SKILL.md
grep -n 'ast_grep_search' hal-service-api-finder/SKILL.md
grep -n '不调用 sub-agent' hal-service-api-finder/SKILL.md
grep -n '术语' hal-service-api-finder/SKILL.md
grep -n '抗理性化' hal-service-api-finder/SKILL.md
grep -n 'state_io_error' hal-service-api-finder/SKILL.md
grep -n 'output_write_failed' hal-service-api-finder/SKILL.md
grep -n '首次分析' hal-service-api-finder/SKILL.md
grep -n '断点续跑' hal-service-api-finder/SKILL.md
```

Expected: every command prints at least one matching line.

- [ ] **Step 8: Commit**

```bash
git add hal-service-api-finder/SKILL.md
git commit -m "docs(hal-service-api-finder): add permissions, constraints, and examples"
```

---

### Task 10: Static walkthrough, final validation, and push

**Files:**
- Modify: `hal-service-api-finder/SKILL.md` only if a walkthrough finds a defect.

- [ ] **Step 1: Run structural and forbidden-pattern checks**

Run:

```bash
ls hal-service-api-finder/SKILL.md >/dev/null && echo "SKILL.md exists"
grep -c '^## ' hal-service-api-finder/SKILL.md
grep -n -E 'TBD|TODO|待定|占位|FIXME' hal-service-api-finder/SKILL.md || echo "no placeholders"
grep -n -E 'summary\.md|truncat' hal-service-api-finder/SKILL.md || echo "no stale summary/truncation names"
grep -n -E 'grep -rn <project_dir>|grep -r .*<project_dir>' hal-service-api-finder/SKILL.md
find hal-service-api-finder -maxdepth 1 -type f -printf '%f\n'
```

Expected:
- `SKILL.md exists`
- heading count is `12`
- `no placeholders`
- `no stale summary/truncation names`
- the project-wide grep lines appear only inside explicit prohibition sentences
- the file listing contains only `SKILL.md`

Validate every JSON block again:

```bash
python3 - <<'PY'
import json, re, pathlib
text = pathlib.Path("hal-service-api-finder/SKILL.md").read_text(encoding="utf-8")
blocks = re.findall(r"```json\n(.*?)\n```", text, re.S)
for i, block in enumerate(blocks, 1):
    json.loads(block)
print(f"{len(blocks)} JSON blocks valid")
PY
```

Expected: `N JSON blocks valid` with no exception.

- [ ] **Step 2: Walk through the ar_service sample manually**

Read `hal-service-api-finder/SKILL.md` end to end and verify each item:

| # | Walkthrough item | Expected prompt behavior |
|---|------------------|--------------------------|
| 1 | Search `"ar_service"` in scope | Finds `.moduleName = "ar_service"` in `g_ArDriverEntry` |
| 2 | Registration | Finds `HDF_INIT(g_ArDriverEntry)`; records `registration.form = "HDF_INIT"` |
| 3 | Bind | Reads `HdfArDriverBind`; records `host_type = "HdfArHost"` |
| 4 | Dispatch | Finds `hdfArHost->ioService.Dispatch = ArDriverDispatch` |
| 5 | Stub/impl | Finds `GetOrNewObject(serviceImpl, IActivityInterface::GetDescriptor())`, `ArService`, `IActivityInterface` |
| 6 | SendRequest | Follows `hdfArHost->stub->SendRequest(cmdId, ...)` |
| 7 | OnRemoteRequest | Locates `ActivityInterfaceStub::OnRemoteRequest`; outer cmd is still empty before its switch |
| 8 | Outer cmd | The first switch case (`CMD_ACTIVITY_INTERFACE_GET_VERSION`) becomes the outer cmd |
| 9 | Handler | Records `ActivityInterfaceStubGetVersion` as the direct target |
| 10 | Default | `default` branch goes to `default_cases`, not final JSON |
| 11 | Output | Eight records; GET_VERSION has value 0; other values resolved from headers |
| 12 | Resume | Interrupt after case 3; rerun skips cases 1-3 and continues at case 4 |

If any row fails, fix `SKILL.md` before continuing.

- [ ] **Step 3: Confirm the final diff and commit**

Run:

```bash
git status --short
git diff --stat
git add hal-service-api-finder/SKILL.md
git commit -m "feat(hal-service-api-finder): add complete SKILL.md"
git status --short
```

Expected: the commit succeeds; the working tree is clean.

- [ ] **Step 4: Push to the public repository**

Run:

```bash
git push origin main
git log -1 --oneline
git status --short --branch
```

Expected:
- push succeeds;
- the latest commit is visible;
- branch is `main` and tracks `origin/main`;
- working tree is clean.

---

## Plan Self-Review

- **Spec coverage:** Tasks 1-10 cover frontmatter, principles, usage, scope, recovery, all three phases, output, permissions, constraints, anti-rationalization, examples, and static walkthrough. Every spec section maps to a task.
- **Placeholder scan:** The plan contains no TBD/TODO/fill-in instructions; every content step specifies exact rules, tables, schemas, commands, and expected results.
- **Type consistency:** Field names are consistent across tasks: `cmd_id_macro`, `cmd_id_value`, `interface_function`, `file`, `phase`, `status`, `queue`, `cases`, `handlers`, `default_cases`, `unresolved`, `dispatch_cache`, `visited`, `counters`.
