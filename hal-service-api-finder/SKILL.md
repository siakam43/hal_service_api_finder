---
name: hal-service-api-finder
description: 当用户调用 /hal-service-api-finder <project_dir> <service_name>，或要求在 OpenHarmony/HDF HAL 层 C/C++ 代码中识别指定 service 对外提供的全部接口函数时使用。按 HdfDriverEntry.moduleName 精确定位服务注册点，分析 Bind/Dispatch/cmdId 分发，输出 cmdId 到接口函数的 JSON 映射。
---

# hal-service-api-finder

根据 OpenHarmony/HDF HAL 层 service 的注册流程，识别指定 service 对外提供的全部接口函数，输出 cmdId 到接口函数的 JSON 映射，用于安全攻击面分析。

## 一、核心原则

1. **分析质量优先于分析效率。** 宁可慢，不可草率。每个分支都要追到接口函数或明确的终止条件，不得因任务长而跳读、合并或提前退出。不设深度上限、不设扇出上限，所有静态可确定的目标全部追踪。
2. **分析范围是硬边界。** 存在 `clean_code.txt` 时，只分析列表（含文件夹展开）内的文件；所有搜索都限制在 scope 内，禁止 `grep -rn <project_dir>`。
3. **只识别目标路径内真正处理 data 的业务函数。** cmdId 用例分支中直接接收 data 且不构成分发的业务函数若位于 `<target_path>` 下，先判断是否为明显封装：是则沿唯一 data 下游继续，直到真正处理 data 的函数；否则当前函数即接口。找不到真正处理函数时，保留最深的 `<target_path>` 封装函数。仍不展开最终接口的内部实现。
4. **证据优先，不臆造。** 每个接口、每个 cmdId 值、每个调用目标都要有代码证据；无法确定的写入 `unresolved`，不得猜测。
5. **全部由主 agent 完成。** 不调用 sub-agent，不把分析拆给子代理。

## 二、使用方法与输出

### 使用方法

/hal-service-api-finder <project_dir> <service_name>

- 两个参数均必填，不能为空；缺失任意一个或参数多于两个，打印用法并结束。
- project_dir 必须是已存在的目录；相对路径转换为绝对路径后使用。
- service_name 是 HdfDriverEntry.moduleName 的精确字符串，区分大小写，不做模糊匹配。

### 输出

最终输出：<project_dir>/.ethunter_out/hal-service-api-finder/<service_name>_api.json

最终 JSON 为扁平数组，每条记录四个字段：
- cmd_id_macro：最外层具体 case 的宏名或表达式原文。
- cmd_id_value：整数；无法解析或非单一取值时为 null。
- interface_function：定义处的作用域限定名。
- file：函数定义所在文件的绝对路径。

不生成 summary 报告。

### 完成报告

完成时报告：分析范围文件数、service_name、入口变量、Bind 函数、Dispatch 函数数量、cmdId 用例数、接口函数数量、unresolved 数量、最终输出文件绝对路径。

## 三、初始化与分析范围

### 参数解析

1. 校验两个参数均非空：`<project_dir>` 与 `<service_name>` 缺失任意一个、为空或仅含空白字符、或参数多于两个时，打印用法并结束。
2. 将 `<project_dir>` 解析为绝对路径：相对路径按当前工作目录展开并规范化，后续一律使用绝对路径。
3. `<project_dir>` 不存在或不是目录 → 报错 `invalid_project_dir` 并结束，不创建任何输出。
4. `<service_name>` 仅去除首尾空白，保持大小写与其余字符不变，用于后续精确匹配。
- `service_name` 仅允许字母、数字、下划线、点和连字符；不得包含 `/`、`\`、空白、换行、NUL、引号或 shell 元字符；不满足 → 报错 `usage_error` 并结束。
- 搜索时按字面量处理并正确加引号（优先 `grep -F`）。
### 目标路径（非命令行参数）

`<target_path>` 默认为 `<project_dir>/vendor/hisi/`；归一化为绝对路径并补尾斜杠，相对路径按 `<project_dir>` 展开，可人工修改。函数定义文件的绝对路径以 `<target_path>` 开头（区分大小写）才算在目标路径下。`<target_path>` 不扩大 scope；目标路径不存在或不在 scope 内时不会有函数命中。

### 输出目录初始化

- 输出根目录：`<out>` = `<project_dir>/.ethunter_out/hal-service-api-finder`。
- 若 `<out>` 已存在，先整体备份为 `<out>_bak`（旧 `<out>_bak` 先删除，只保留最近一次备份），再新建空 `<out>`；不得直接删除旧结果。`<out>` 不存在时保留现有 `<out>_bak`。

### 分析范围确定

按以下优先级确定分析范围，命中即停止；范围一经生成即为本次分析的硬边界：

1. 先用双方法检查 `<project_dir>/.ethunter_out/clean_code.txt`：存在则按“clean_code.txt 优先，且是硬边界”建立 scope。
2. `clean_code.txt` 不存在时，检查 `<project_dir>/.ethunter_out/.etignore`：存在则全量 find C/C++ 文件并按 `.etignore` 规则排除。
3. `.etignore` 也不存在时，直接全量 find C/C++ 文件。
4. 无论走哪条分支，都必须生成 `<out>/scope_files.txt`；存在 `clean_code.txt` 时还必须生成 `<out>/scope_roots.txt`，否则 `scope_roots` 置为 null。
- 每次执行都重新生成 `<out>/scope_files.txt`；`<out>/scope_roots.txt` 仅在存在 `clean_code.txt` 时生成；不读取旧 scope。

### clean_code.txt 优先，且是硬边界

双方法检查 <project_dir>/.ethunter_out/clean_code.txt 是否存在。

存在时：
1. 逐行读取，去掉首尾空白；忽略空行和以 # 开头的注释行。
2. 以 / 开头为绝对路径，否则拼接为 <project_dir>/<行内容>。
3. 路径去重、规范化（分隔符统一为 /，去掉前导 ./）。
4. 普通文件加入 scope；文件夹递归展开为其中全部常规文件；不存在的路径忽略。
5. 生成 <out>/scope_files.txt（展开后的绝对文件路径，去重、排序）和 <out>/scope_roots.txt（原始文件/文件夹条目，去重、排序）；写入失败 → 写 `error.json`（`phase = "init"`，`code = "internal_error"`），停止。
6. scope 为空 → 写 `error.json`（`phase = "init"`，`code = "empty_scope"`），停止。

硬边界规则：
- clean_code.txt 存在时，不得分析或搜索列表以外的源文件。
- 入口、Bind、Dispatch、接口函数定义、宏定义的查找都只能在 scope 内进行。
- 范围外不搜索；若已知目标在范围外，记入 `unresolved`，不输出、不臆造。
- 搜索方式见下方 `### 搜索方法与纪律`。

### clean_code.txt 不存在

1. `.ethunter_out/.etignore` 存在：按下方 find 命令收集 C/C++ 文件，再按 `.etignore` 规则排除；`.etignore` 语法与 `.gitignore` 一致。
2. `.etignore` 也不存在：直接使用下方 find 命令收集 C/C++ 文件。
3. 两种情况的 find 结果都规范化为绝对路径、去重、排序后写入 `<out>/scope_files.txt`；写入失败 → 写 `error.json`（`phase = "init"`，`code = "internal_error"`），停止；结果为空 → 写 `error.json`（`phase = "init"`，`code = "empty_scope"`），停止。

C/C++ 扩展名：
.c .h .cpp .cxx .cc .hpp .hxx .hh .inl .ipp

```bash
find <project_dir> -type f \( -name "*.c" -o -name "*.h" -o -name "*.cpp" -o -name "*.cxx" -o -name "*.cc" -o -name "*.hpp" -o -name "*.hxx" -o -name "*.hh" -o -name "*.inl" -o -name "*.ipp" \)
```

无 clean_code.txt 时，`scope_roots = null`，不使用 `scope_roots.txt`。

### 搜索方法与纪律

所有阶段统一使用以下三种方法；后续只写方法名与参数，不再重复命令。其中 `<out>` = `<project_dir>/.ethunter_out/hal-service-api-finder`。

1. 精确字符串搜索：在 scope 内查找字面量。
2. 正则/词边界搜索：需要正则时把 `-F` 换成 `-E`；需要词边界时加 `-w`。
3. 指定文件搜索：已知文件在 scope 内时使用。

```bash
# 1. 精确字符串搜索
xargs -r -a "<out>/scope_files.txt" -d '\n' grep -I -H -n -F "<pattern>"

# 2. 正则/词边界搜索
xargs -r -a "<out>/scope_files.txt" -d '\n' grep -I -H -n -E "<pattern>"

# 3. 指定文件搜索
grep -I -H -n -E "<pattern>" "<file>"
```

全局纪律：

- 只搜索 `scope_files.txt` 中的文件；`scope_roots.txt` 非 null 时才允许按根目录搜索（把 xargs 的 `-a` 参数改为 `<out>/scope_roots.txt`，并给 `grep` 加 `-r`）。
- 禁止 `grep -rn <project_dir>`、`rg <project_dir>`、`find <project_dir> -exec grep` 等全仓搜索。
- `grep -I` 保留，跳过二进制文件；不得用 `2>/dev/null` 隐藏错误；退出码 1/123 表示无匹配，不是失败。
- codegraph 结果按 `scope_files.txt` 过滤；范围外搜索命中直接丢弃，已知范围外目标记 `unresolved`。

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

## 四、中间结果记录

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

## 五、阶段一：定位服务注册点

阶段一只做一件事：在 scope 内找到 `.moduleName` 与 `<service_name>` 精确相等的 `HdfDriverEntry` 初始化器，确认注册形式，写出 `01_entry.json`。本阶段不分析 Bind 内部逻辑，不进入 Dispatch。

### 精确搜索

按「精确字符串搜索」查找带双引号的完整字面量 `"<service_name>"`。

- 每个命中都必须用 Read 读取至少 30 行上下文（`offset=max(1, 命中行-15)`，`limit` 不少于 31）；向上读到初始化器声明行，向下读到闭合 `};`。不得只凭 grep 行下结论。
- 一个命中只有同时满足以下条件才成为候选：
  1. 命中行是 `.moduleName` 成员初始化（允许 `.moduleName=...` 等空白差异），引号内字符串与 `<service_name>` 完全相等（区分大小写，`grep -F` 已按字面量匹配）；
  2. 向上可确认该成员位于 `HdfDriverEntry` 类型的初始化器内（声明形如 `struct HdfDriverEntry <var> = {` 或 `HdfDriverEntry <var> = {`，可带 `static`、`const` 等限定符），且 `.moduleName` 处于该初始化器的成员层级；
  3. 命中不在注释或字符串常量中。
- 不满足的命中（注释、日志、其他结构体字段、同名子串、非 `HdfDriverEntry` 初始化器等）丢弃，并记录文件、行号与原因，供 `service_not_found` 的 evidence 使用。
- 每个合法初始化器形成一个候选，提取并记录：
  - `variable`：初始化器变量名；
  - `file`：`scope_files.txt` 中登记的绝对路径；
  - `line`：初始化器声明起始行号（`struct HdfDriverEntry <var> = {` 或 `HdfDriverEntry <var> = {` 所在行）；
  - `module_name`：`<service_name>`；
  - `bind` / `init` / `release`：对应成员的赋值表达式原文；成员缺失或值为 `NULL` / `nullptr` / 空时记 `null`。

合法命中示例：

```c
struct HdfDriverEntry g_DemoDriverEntry = {
    .moduleVersion = 1,
    .moduleName = "demo_service",
    .Bind = HdfDemoDriverBind,
    .Init = HdfDemoDriverInit,
    .Release = HdfDemoDriverRelease,
};

HDF_INIT(g_DemoDriverEntry);
```

上例应提取 `variable = g_DemoDriverEntry`、`bind = HdfDemoDriverBind`、`init = HdfDemoDriverInit`、`release = HdfDemoDriverRelease`。

### 注册形式确认

对每个候选的 `variable`，按「正则/词边界搜索」查找其变量名（加 `-w` 避免子串误命中）。

对每个引用读取足够上下文（至少覆盖其所在语句或初始化器），按以下优先级判定，命中即停：

1. `HDF_INIT`：找到 `HDF_INIT(<entry_var>)` 或 `HDF_INIT(&<entry_var>)`。必须是有效可执行语句，不能位于注释、字符串或日志中；确认括号匹配、变量一致，记录该行文件、行号，`evidence` 为语句原文。
2. `other`：没有 `HDF_INIT`，但找到其他语义明确的注册证据，例如注册函数调用（`RegisterXxx(<entry_var>)`）、驱动表/数组条目（`{ .moduleName = ..., .driver = <entry_var> }`）、注册宏。必须是有效可执行语句，不能是注释或日志。
3. `unverified`：以上都没有。`file`、`line`、`evidence` 均记 `null`。

- 变量声明、`extern`、普通读取/赋值、注释或日志提及都不是注册证据，不得记为 `other`。
- 同一候选有多处有效注册证据时，记录全部位置；`registration.file` / `registration.line` 取 `scope_files.txt` 顺序中的第一处，`evidence` 列出全部位置。
- 条件编译分支按第三章 macro.json 规则处理。
- 记录 `entry.registration = {form, file, line, evidence}`；`form` 只能是 `HDF_INIT` / `other` / `unverified`。

### 服务存在判定

- 存在一个 `moduleName` 精确匹配的 `HdfDriverEntry` 初始化器，即判定服务存在。
- 注册引用是辅助证据，不是硬门槛：`registration.form == unverified` 时仍继续写阶段产物并进入阶段二，不得因此报 `service_not_found`。
- `.moduleName` 不匹配、或类型不是 `HdfDriverEntry` 的初始化器不算命中。

### 多命中处理

候选按 `(file, 初始化器声明起始行)` 去重，同一初始化器的多处文本命中只算一个候选；同名但声明行不同的初始化器不得合并。

- 候选数为 0：按“失败处理”报 `service_not_found`。
- 候选数为 1：直接使用该候选，无论其注册形式。
- 候选数 ≥ 2：先对每个候选完成“注册形式确认”，再按以下规则决定：
  - 恰好一个候选有明确注册证据（`registration.form` 为 `HDF_INIT` 或 `other`）→ 使用该候选；
  - 有两个及以上已注册候选，或没有任何已注册候选 → 报 `multiple_matches`，列出全部候选的 `variable`、`file`、`line`、`registration.form`、`registration.evidence`，写 `error.json` 并停止；不得任选、不得按文件顺序猜测。
- `unverified` 候选不计入“已注册”。

### 失败处理

失败一律写 `error.json = {phase: "locate_entry", code, message, evidence}`，不写 `01_entry.json`，不进入阶段二，停止。

- `service_not_found`：没有任何合法候选。`message` 必须包含 scope 文件数（`scope_files.txt` 行数）和已搜索模式（至少 `"<service_name>"`），并建议检查 service_name 拼写与 clean_code.txt 覆盖范围；`evidence` 列出被拒命中及其原因；零命中时明确写“0 命中”。
- `bind_not_found`：已选定唯一候选，但 `.Bind` 缺失、为空或为 `NULL` / `nullptr`。`message` 与 `evidence` 记录 `entry` 变量、文件、行号及 `.Bind` 原文/缺失情况。该检查在多命中消歧之后执行；失败时不写 `01_entry.json`，停止。

### 阶段产物

定位成功后：

1. 组装 `01_entry.json`：
   - `service_name`：当前 `<service_name>`；
   - `scope`：`file_count` 为 `<out>/scope_files.txt` 的行数；`clean_code` 为本次是否使用 clean_code.txt（布尔值）；`scope_files` 固定为 `"scope_files.txt"`；`scope_roots` 在使用 clean_code.txt 时为 `"scope_roots.txt"`，否则为 `null`；
   - `entry`：本阶段提取的 `variable`、`file`、`line`、`module_name`、`bind`、`init`、`release`、`registration`。

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

2. 直接写 `<out>/01_entry.json`；写入失败 → 删除可能残留的 `<out>/01_entry.json`，再写 `error.json`（`phase = "locate_entry"`，`code = "snapshot_write_failed"`），停止。
3. 进入 `## 六、阶段二：分析 Bind`。

## 六、阶段二：分析 Bind

阶段二只做一件事：在 scope 内定位 `entry.bind` 指向的 Bind 函数定义，提取 Host 类型、全部 Dispatch 函数、可选的 stub/接口/实现类信息与服务注册证据，写出 `02_bind.json` 后进入阶段三。本阶段不做 cmdId 追踪，不展开业务函数。

### 定位 Bind 定义

输入：`01_entry.json` 的 `entry.bind`、`entry.file`、`entry.init`（阶段一已确认 `.Bind` 非空）。

1. 归一化 `.Bind` 取值：
   - 去掉首尾空白、取地址符 `&` 和强制类型转换（如 `(int32_t (*)(struct HdfDeviceObject *))HdfDemoDriverBind`），得到待解析标识符。
   - 标识符是宏时：按「精确字符串搜索」查找 `#define <宏名>`，必要时查 `macro.json`，逐层展开到函数名；无法展开或展开后找不到定义 → 报 `bind_definition_not_found`。
   - 标识符是包装函数时（如函数体只有 `return HdfDemoDriverBind(dev);`）：沿调用继续解析到真正创建 Host、设置 Dispatch 的函数，`bind.function` 记真实 Bind 函数；若包装函数自身完成 Host/Dispatch 初始化，则以包装函数为准。
   - C++ 作用域函数保留限定名（如 `DemoDriver::Bind`）。若限定名在 scope 内找不到定义，定位该类的类定义并在类体内搜索未限定方法名；`bind.function` 仍记限定名。
   - 解析包装函数/初始化调用链时维护 visited 集合（键为 `(file, 签名行)`）；重复出现 → 停止并记录环，不得无限展开。
2. 查找定义：
   - 优先按「指定文件搜索」在 `entry.file` 内查找 `<bind_name>`（该文件必在 scope 内）。
   - 未找到再按「正则/词边界搜索」在全 scope 查找 `<bind_name>`。

   - 只接受函数定义；函数调用、声明、注释、字符串中的同名文本一律不算。
   - 读到的定义函数名必须与待解析标识符完全一致（C++ 限定名按限定名匹配；`HdfDemoDriverBindEx` 等不算）。
   - 候选按 `(file, 签名行)` 去重，同一函数的多处命中只算一个候选。
3. 验证完整函数体：对每个候选用 Read 从签名行读到配对的闭合 `}`，确认函数体完整（以 `);` 结尾的声明不算）。不得只看 grep 命中行。`bind.file` 记 scope 内绝对路径，`bind.line` 记函数签名所在行。
4. 多定义处理（按顺序）：
   - 候选 = 带完整函数体的定义。`entry.file` 内有候选时，候选范围取 `entry.file`；否则取全 scope。
   - 签名兼容优先：兼容 `int32_t (*)(struct HdfDeviceObject *)` 指返回类型为 `int32_t` 或 `int`，只有一个参数且参数类型为 `struct HdfDeviceObject *`（允许写作 `HdfDeviceObject *`，忽略参数名与顶层限定符）。
   - 候选范围内兼容候选恰好 1 个 → 选定；≥ 2 个 → 报 `bind_ambiguous`，evidence 列出全部候选的 `name`、`file`、`line`、签名。
   - 兼容候选为 0：完整函数体候选恰好 1 个 → 选定；≥ 2 个 → 报 `bind_ambiguous`；一个都没有 → 报 `bind_definition_not_found`，evidence 列出搜索名、scope 文件数以及被拒命中（声明、调用、注释、无函数体）与原因。

### 提取 Host 与 Dispatch

在 Bind 函数体（含展开后的真实函数体）内定位 `deviceObject` 参数与 Host 实例；参数名以实际代码为准（常见 `deviceObject`、`device`、`obj`）。

Host 实例的创建形式都要识别：

- `new <HostType>`：`HdfDemoHost *hdfDemoHost = new HdfDemoHost();`、`new (std::nothrow) HdfDemoHost()`、`auto *host = new HdfDemoHost;`
- 指针实例：`HdfDemoHost *hdfDemoHost = &g_arHost;` 或由函数返回的指针；
- 栈实例：`HdfDemoHost hdfDemoHost;`、`HdfDemoHost hdfDemoHost = {};`
- 静态实例：`static HdfDemoHost hdfDemoHost;`、`static HdfDemoHost *hdfDemoHost = ...;`
- 智能指针/工厂：`sptr<HostType> host = ...`、`std::unique_ptr<HostType>`、`std::make_unique<HostType>()`、工厂函数返回的实例。

`bind.host_type` 记 Host 实例的类型名（去掉 `struct`/`class`、指针与顶层限定符，如 `HdfDemoHost`）。多个实例时，取最终承载 Dispatch 赋值、且与 `deviceObject->service = &<host>->ioService` 一致的实例；无法唯一确定时仍保留全部实例上的 Dispatch 函数，并在 `via` 中区分实例，`host_type` 记承载 Dispatch 的实例类型或 null。

接受以下 Dispatch 赋值形式（`host` 代表识别出的实例，`.` 与 `->` 互换视为等价）：

- `host->ioService.Dispatch = X;`
- `host->ioService = { .Dispatch = X };`（含 `= (struct IDeviceIoService){ .Dispatch = X }` 等等价初始化）
- `host.ioService.Dispatch = X;`
- `host->Dispatch = X;`

对每个有效赋值：

- 用 Read 读取完整语句或初始化器以及所在函数上下文，确认赋给的是本 Bind 识别出的 host 实例，且不在注释/字符串中。
- 记录全部 Dispatch 函数：`name` 为函数名（宏/包装同样展开到真实函数），`file`/`line` 为函数定义所在文件与签名行（用与 Bind 相同的“搜索 + 完整函数体”方法在 scope 内确认），`via` 记赋值左侧表达式原文（如 `hdfDemoHost->ioService.Dispatch`）；经初始化函数间接发现时写 `<初始化函数名>: <左侧表达式>`。
- 按 `(name, file, line)` 去重；同一函数有多处赋值时，`via` 用 `;` 连接全部赋值左侧表达式。
- Dispatch 函数定义不在 scope 内时，`file`/`line` 记 null，`via` 记 `<赋值文件>:<赋值行> <左侧表达式>`，不丢弃该函数。阶段三初始化队列时跳过 file 为 null 的 Dispatch 条目并记入 `03_trace.json` 的 `unresolved`（`kind = "definition"`）；若所有 Dispatch 条目都被跳过且 `bind.stub.on_remote_request` 非 null，则直接用该 OnRemoteRequest 初始化工作项。

Bind 函数体内没有 Dispatch 赋值时，按以下顺序继续，不得因 Bind 内没有赋值就判定失败：

1. 跟随 Bind 函数体内调用的初始化/创建函数（接收 host、deviceObject 或返回 host 的函数），在 scope 内读取其完整函数体并查找上述赋值形式；维护 visited 集合（键为 `(file, 签名行)`），重复即停止并记录环。
2. 检查阶段一记录的 `entry.init` 及其调用的 host 初始化函数，只在其中查找 Dispatch 赋值，不展开其他逻辑。
3. 按「正则/词边界搜索」在 scope 内查找 Dispatch 赋值 `(\.|->)Dispatch[[:space:]]*=` 与 host 标识 `<host_var>`；其中 `<host_var>` 替换为实际变量名，可再用 host 类型名补充搜索。
4. 对每个命中读取上下文，只保留作用于同一 host 实例（直接或经指针传递）的赋值。

以上步骤完成后仍未找到任何 Dispatch 赋值 → 报 `dispatch_not_found`，停止。

### 提取 stub、接口与实现类（可选）

在 Bind 函数体、Host 创建/初始化函数以及 `entry.init`（非 null 时）中查找 stub 成员赋值。stub 信息缺失不是失败条件。

1. 识别 stub 成员赋值：赋值左侧为 host/deviceObject 上名称包含 `stub`（不区分大小写）的成员，如 `hdfDemoHost->stub = ...`。
2. 按右值形式追踪：
   - `new XxxStub` / `new (std::nothrow) XxxStub(...)`：`stub_class` 记 `XxxStub`；`interface_type` 从 stub 基类、模板参数或 `GetDescriptor()` 返回值确定。
   - `GetOrNewObject(serviceImpl, IDemoInterface::GetDescriptor())`：`interface_type` 取描述符所属接口或 `serviceImpl` 声明中的接口类型；`impl_class` 沿 `serviceImpl = impl`、`impl = new DemoService()` 等赋值回溯到实际传入的实现类；`stub_class` 取 scope 内该接口对应的生成 Stub 类型，搜索 `<interface_type>Stub` 以及去掉接口名前导 `I` 后的 `<StubName>`（如 `IDemoInterface` → `DemoInterfaceStub`），并用 `OnRemoteRequest` 交叉确认；无法确定时记 null，不猜测。
   - 其他等价形式（工厂函数返回 stub 等）同样回溯到 stub 类型与接口。
3. 定位实现类定义：在 scope 内搜索 `class <impl_class>` / `struct <impl_class>`（允许命名空间限定），必须看到类体（不是仅前置声明），`impl_file` 记类定义文件；不在 scope 内或仅有前置声明记 null，不视为失败。
4. 定位 `OnRemoteRequest`：按「正则/词边界搜索」优先搜索 `<stub_class>::OnRemoteRequest`；未命中则在 stub 类定义文件及其基类定义中搜索 `OnRemoteRequest` 定义。多个候选时优先精确 stub 类，其次最近的基类；记录完整作用域限定名、文件、定义行，其余候选写入 `stub.evidence`。找不到时 `on_remote_request` 记 null，不视为失败。
5. 组装 `bind.stub = {member, interface_type, impl_class, impl_file, stub_class, on_remote_request, evidence}`；`evidence` 记 stub 赋值与 `new`/`GetOrNewObject` 表达式原文。没有 stub 赋值或无法确认任何 stub 字段 → `bind.stub = null`。

### 服务注册证据

在 Bind 函数体及跟随的初始化函数中查找 deviceObject 与 host ioService 的关联赋值，例如：

- `deviceObject->service = &host->ioService;`
- `deviceObject->service = &host.ioService;`
- 经指针别名或强制类型转换的等价写法。

记录 `bind.service_registration = {file, line, evidence}`：`file`/`line` 取 `scope_files.txt` 顺序中的第一处有效赋值，`evidence` 记语句原文（多处时列出全部）。该字段仅作证据，缺失不失败：记 `{file: null, line: null, evidence: null}` 后继续。

合法示例：

```c
hdfDemoHost->ioService.Dispatch = DemoDriverDispatch;
hdfDemoHost->ioService.Open = nullptr;
hdfDemoHost->ioService.Release = nullptr;

DemoService *impl = new (std::nothrow) DemoService();
...
sptr<IDemoInterface> serviceImpl = impl;
hdfDemoHost->stub = OHOS::HDI::ObjectCollector::GetInstance().GetOrNewObject(serviceImpl,
    IDemoInterface::GetDescriptor());
...
deviceObject->service = &hdfDemoHost->ioService;
```

若 host 声明为 `HdfDemoHost *hdfDemoHost`，上例应提取：`host_type = HdfDemoHost`；`dispatch_functions = [{name: "DemoDriverDispatch", file: <定义文件>, line: <定义行>, via: "hdfDemoHost->ioService.Dispatch"}]`；`stub.member = "stub"`、`stub.interface_type = "IDemoInterface"`、`stub.impl_class = "DemoService"`、`stub.stub_class = "DemoInterfaceStub"`、`stub.on_remote_request` 指向该 Stub 的 `OnRemoteRequest` 定义；`service_registration.evidence = "deviceObject->service = &hdfDemoHost->ioService"`。`Open` / `Release` 不是 Dispatch，忽略。

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

- `service_name` 原样取自 `01_entry.json`；`entry` 只写 `variable`、`file`、`module_name` 三个字段。
- `bind` 对象字段形状见上方示例（`function` / `file` / `line` / `host_type` / `dispatch_functions` / `stub` / `service_registration`），不写 `status`。
- 直接写 `<out>/02_bind.json`；写入失败 → 删除可能残留的 `<out>/02_bind.json`，再写 `error.json`（`phase = "analyze_bind"`，`code = "snapshot_write_failed"`），停止。
- 进入 `## 七、阶段三：cmdId 分发追踪`。

失败处理：

- 任一失败码一律写 `error.json`（`phase = "analyze_bind"`），不写 `02_bind.json`，不进入阶段三，停止。
- `bind_definition_not_found`：宏/包装无法展开，或展开后 scope 内没有带完整函数体的定义。message 记录搜索名与 scope 文件数；evidence 列出展开链、被拒命中及原因。
- `bind_ambiguous`：存在多个可选定义且无法按同文件/签名规则唯一确定。evidence 列出全部候选的 `name`、`file`、`line`、签名。
- `dispatch_not_found`：Bind 体内、跟随的初始化函数中以及 scope 搜索均未发现 Dispatch 赋值。evidence 记录 host 变量/类型、已跟随的函数、搜索模式与命中处理结果。

## 七、阶段三：cmdId 分发追踪

阶段三只做一件事：从阶段二的 Dispatch（或 stub 的 OnRemoteRequest）出发，沿全部分发路径追踪 cmdId 与 data 的流向。对不在 `<target_path>` 下的 data 业务函数，沿 data 实参继续追踪；到 `<target_path>` 内接收 data 且不构成分发的业务函数后，只对明显的单线封装继续下钻，停在真正处理 data 的函数；找不到时保留最深的 `<target_path>` 封装函数。不展开最终接口的内部实现，也不做完整污点传播。最终把接口函数候选写入 `<out>/03_trace.json`。

- 不设深度上限、不设扇出上限；所有静态可确定的目标全部追踪，环检测是唯一的循环保护。
- 接口函数定义必须在 scope 内；定义在 scope 外时记入 `unresolved`，不输出、不臆造。

### 3.1 输入与队列初始化

输入取自 `02_bind.json` 的 `bind`：`dispatch_functions`（`name` / `file` / `line` / `via`）、`stub.on_remote_request`、`stub.stub_class`、`stub.interface_type`、`stub.impl_class`。`bind.host_type` 与 `bind.service_registration` 只作证据，不参与追踪。

在内存中初始化：

1. `service_name` 取 `02_bind.json` 的顶层 `service_name`；`entry_dispatchers` 原样记录阶段二的全部 Dispatch 条目（包括 `file == null` 的条目）。
2. `queue`、`cases`、`handlers`、`default_cases`、`unresolved`、`dispatch_cache`、`visited` 初始化为空数组。
3. 对每个 `dispatch_functions` 条目：
   - `file` 非 null：创建一个 `kind = "dispatch"` 的工作项，字段为 `id`、`dispatcher`、`dispatcher_file`、`dispatcher_line`、`current_cmd_expr`、`current_cmd_value`、`outer_cmd_expr`、`outer_cmd_value`、`data_vars`、`data_derived`、`parent_case_id`、`path`：
     - `id` 从 1 递增；`dispatcher` / `dispatcher_file` / `dispatcher_line` 取条目值；
     - `current_cmd_expr` 记该 Dispatch 实际承载 cmdId 的表达式：优先 cmdId 形参名（如 `cmdId`）；若 cmdId 由 `HdfSbufReadUint32(data, &cmdId)` 等从 data 读入局部变量，则读函数体后修正为该局部变量；`current_cmd_value = null`；
     - `outer_cmd_expr = null`，`outer_cmd_value = null`；
     - `data_vars` 记签名中的 data 形参（通常为 `["data"]`），`data_derived = true`，后续按 3.5 更新；
     - `parent_case_id = null`，`path = [dispatcher]`。
   - `file == null`：不创建工作项，写一条 `unresolved`：`{kind: "definition", detail: "Dispatch <name> 定义不在 scope 内", tried: [<via>, <已执行的 scope 搜索模式>]}`。
4. 若所有 Dispatch 条目都被跳过（初始队列为空）且 `stub.on_remote_request` 非 null：用该 OnRemoteRequest 创建初始工作项：
   - `kind = "dispatch"`；
   - `dispatcher` / `dispatcher_file` / `dispatcher_line` 取 `on_remote_request` 的值；
   - `current_cmd_expr` 取 OnRemoteRequest 的 cmdId 形参（通常为 `code`），`current_cmd_value = null`；
   - `outer_cmd_expr = null`，`outer_cmd_value = null`，`parent_case_id = null`；
   - `data_vars` 取 data/parcel 形参（不含 `reply`、`option`）；`data_derived = true`；`path = [on_remote_request.name]`。
5. 若全部条目被跳过且 `on_remote_request` 也为 null：写一条 `unresolved`（`kind = "definition"`，detail 说明没有可用入口），不创建队列项，直接执行 3.10 的收尾。

初始 Dispatch 不预先入队 OnRemoteRequest；只有读取函数体确认其直接调用 `stub->SendRequest(cmdId, ...)` 时才入队（见 3.3、3.8）。这种直接转发的工作项 `outer_cmd_expr` / `outer_cmd_value` 仍为 null。

### 3.2 单个工作项的处理循环

队列非空时反复执行：

1. 读取队列中的第一个工作项（不移出队列；处理完在第 7 步移除）。若 `kind = "data_chase"`，执行 3.9 后跳到第 7 步；否则继续第 2 步。
2. 定位并完整读取 dispatcher 定义：已知 `dispatcher_file` / `dispatcher_line` 时直接 Read 该文件，从签名行读到配对的闭合 `}`；否则按「正则/词边界搜索」在 scope 内搜索精确名称，只接受带完整函数体的定义（调用、声明、注释、字符串中的同名文本不算），并读完整函数体。C++ 限定名找不到定义时，在 scope 内的类体中搜索未限定方法名，但记录限定名。定义不在 scope 内或无法唯一确定 → 记 `unresolved`（`kind = "definition"`），移除该工作项并继续。
3. 按 3.4 确定 `effective_outer`。
4. 识别函数体内的全部分发结构（见 3.3）。
5. 对每个分支依次执行：
   - 解析 cmdId 表达式与取值（3.4）；
   - 追踪 data 派生（3.5）；
   - 解析并分类分支内的每个调用目标（3.6、3.7）；
   - 仅 `<target_path>` 下的接口候选写入 `handlers`；转发器、内层 dispatcher 与 `data_chase` 入队（3.8）；无法确定的目标写入 `unresolved`。
6. `default` 分支只写入 `default_cases`；函数体内没有任何 cmdId 分发结构时，按“无条件直接处理”规则处理（3.6）。
7. 该工作项的全部分支处理完后，从 `queue` 移除该项，把完整 `visited` 条目（含 `path` 与环检测键）追加到 `visited`（见 3.10）。
8. 队列仍有项 → 继续下一项；队列为空 → 执行 3.10 收尾。

必须读取完整控制流，不得只凭 grep 命中行或单个 case 片段下结论；同一函数内的嵌套分发、fallthrough、宏展开都要覆盖。

### 3.3 分发结构识别

读取 dispatcher 完整函数体，识别并枚举以下全部分发结构：

- **switch / case**：每个 `case` 常量表达式是一个分支；连续多个 `case` 标签共享同一段代码时，每个标签分别形成分支；fallthrough 的代码归属每个落入它的标签。`default` 单独处理，不进入最终 JSON。内层 switch 的每个 case 仍在本工作项内分析，若有外层 case，`parent_case_id` 记该外层 case 的 id；转发工作项内发现的 case 继承该工作项的 `parent_case_id`；否则为 null。
- **range / mask / if-else**：`if (cmdId == X)`、`if (cmdId >= A && cmdId <= B)`、`else if` 链、`if (cmdId & MASK)`、`if ((cmdId >> N) == X)`、三元表达式等，每个能导向不同处理逻辑的条件分支都是一个分支。
- **数组 / map / 函数指针表**：以 cmdId 为下标的静态数组、`std::map` / `unordered_map`、`{cmd, handler}` 条目数组、函数指针表。每个已初始化条目是一个分支；`case_expr` 记下标/键表达式，`case_value` 按 3.4 解析。下标范围未知时枚举全部已初始化条目。
- **表注册**：`RegisterXxx(cmd, fn)`、`Insert(cmd, handler)`、注册表初始化器等，每个注册项是一个分支，分发位置记注册语句。
- **宏生成分发**：X-macro、生成 case 的 `#define`、宏展开后形成的 switch。在 scope 内展开并枚举生成的 case；无法静态展开时记 `unresolved`（`kind = "target"`），列出已尝试的展开。
- **SendRequest / OnRemoteRequest 转发**：`stub->SendRequest(cmdId, data, reply)`、直接调用 `OnRemoteRequest`、调用已知 dispatcher。这些目标本身不是接口函数，按 3.8 入队；若转发位于某个 case 内，该 case 照常记录。
- **无条件直接处理**：没有 cmdId 分支，但函数直接消费 data，或无条件调用 data 消费函数。按 3.6 的 `"*"` 规则记录。

每个识别出的分支先写入 `cases`（字段：`id`、`dispatcher`、`dispatcher_file`、`dispatcher_line`、`dispatch_site`、`level`、`case_expr`、`case_value`、`outer_cmd_expr`、`outer_cmd_value`、`parent_case_id`、`handlers`、`evidence`）：

- `level` 取工作项 `path` 的长度（入口工作项为 1，每次转发入队 +1）；
- `case_expr` 保留源码原文（宏名不替换为值）；
- `outer_cmd_expr` / `outer_cmd_value` 按 3.4 填写；
- `evidence` 记分支条件、注册语句或表项原文。
- `default_cases` 记录 `dispatcher`、`dispatcher_file`、`dispatcher_line`、`dispatch_site`、`evidence`；不写 handlers。

### 3.4 外层归属与 cmdId 取值

```text
外层归属：
effective_outer = 当前分支已确定外层 ?? 工作项.outer ?? 本用例

- 内层 case 继承其外层 case 的 effective_outer，不得用内层 case 覆盖外层。
- 工作项外层为空且当前分支没有已确定外层 → 本用例成为外层，记录宏/表达式和值。
- 外层已存在 → 保留外层，本用例只作内层证据。
- Dispatch → SendRequest → OnRemoteRequest 时，OnRemoteRequest 的 switch case 成为外层。
- 嵌套分发继承 effective_outer。
```

- 每个 case 的 `outer_cmd_expr` / `outer_cmd_value` 记 `effective_outer` 的表达式与值；`handlers` 的 `cmd_id_macro` / `cmd_id_value` 也取 `effective_outer`，内层 case 只作为证据写入对应 case 的 `evidence`。
- Dispatch 直接透传 cmdId 给 `SendRequest`（没有外层 case）时，入队的 OnRemoteRequest 工作项外层仍为 null，因此其 switch case 成为外层；若 SendRequest 位于已确定外层的分支内，则保留该外层，不因进入 OnRemoteRequest 而改写。
- 具名 cmdId 即使值无法解析，也必须保留在最终输出中（值为 `null`）。

cmdId 取值按以下规则解析，结果写入 `case_value`；失败时 `case_value = null`，`case_expr` 仍保留原文：

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

解析细节：

- `#define` 只在 scope 内搜索与展开；维护宏展开 visited 集合，重复宏名即停止，防止宏环。展开时保留括号与 C 整数运算语义；除法/取模除数为 0 视为无法解析。
- 枚举按显式赋值与隐式递增（前一个值 +1）计算；匿名枚举、命名空间内枚举、`enum class` 都要处理。
- 宏/枚举/常量优先查代码，未命中或存在条件编译歧义时再查 `macro.json`；`macro.json` 只是辅助，不能覆盖代码中的确定定义。
- 同名宏存在多个定义（不同文件或不同条件编译分支）时，不得任选：`case_value = null`，写 `unresolved`（`kind = "cmd_value"`），detail/tried 列出全部候选定义的文件、行号与表达式。
- 每个无法解析或存在冲突的 cmdId 至少产生一条 `unresolved`；同一 case 的重复解析只记一条。

### 3.5 data 派生追踪

在每个分支内追踪哪些实参是 data 派生值。工作项的 `data_vars` 只记当前函数内与客户端 data 对应的形参名（不是调用点实参，也不是局部 parcel）；分支级的派生/覆盖只在当前工作项内使用，不回写 `data_vars`。

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

- 按分支作用域追踪：一个分支内的派生/覆盖不自动带到兄弟分支；控制流合流处无法确定时，保留派生标记并在该 case 的 `evidence` 中注明“合流处按派生保守处理”。
- 只做分类所需的局部追踪，不要求完整污点传播路径；C++ 间接调用、虚函数、函数指针、重载等解析可参考 `taint-path-walker` skill，本 skill 只追踪到真正处理 data 的业务函数（明显单线封装继续按 3.6 下钻）或明确终止条件。
- 全局变量或成员变量写入后在另一个函数读取时，写 `unresolved`（`kind = "data_deferred"`），记录写入点、读取点与已尝试的 scope 搜索；不得假设跨函数传播。
- `reply` 及其派生值永远不参与 data 派生判断。

### 3.6 被调用目标分类

对分支内的每个调用目标，按以下顺序判定，命中即停（保证表格各行互斥）：

1. 已知转发器；
2. 接收 cmdId 且函数体内有分发逻辑；
3. 排除项：纯日志、断言、框架 sbuf/parcel 转换、纯校验辅助、无业务动作、只收 cmdId 不收 data 也不分发、只收 reply 不读 data；
4. 接收 data/派生参数且不构成分发的业务函数 → 定义在 `<target_path>` 下记为接口候选，否则入队 `kind = "data_chase"` 工作项；
5. 其余 → 忽略。

纯校验辅助即使接收 data，也按排除项处理，不写 handlers。

| 目标特征 | 处理 |
|----------|------|
| 已知转发器（`SendRequest`、`OnRemoteRequest`、已知 dispatcher） | 入队为新工作项 |
| 接收 cmdId 且函数体内有分发逻辑 | 入队为内层 dispatcher |
| 接收 cmdId + data，且函数体内有分发 | 先作为分发函数入队，不直接记为接口 |
| 接收 data/派生参数，且不构成分发 | 在 `<target_path>` 下记为接口候选；否则入队 `data_chase` |
| 纯日志、断言、框架 sbuf/parcel 转换、纯校验辅助、无业务动作 | 忽略，写入 evidence |
| 只接收 cmdId、不接收 data、也不分发 | 忽略 |
| 只接收 reply、不读 data | 不记为接口函数 |

目标是否在 `<target_path>` 下，按函数定义文件的绝对路径前缀判断（区分大小写）。

分支级规则：

- 一个分支有多个 data 消费目标 → 每个目标分别按 `<target_path>` 判断：在目标路径下的各写一条 `handlers`，共用同一 `effective_outer`，并把 `{name, file, line}` 追加到当前 case 的 `handlers`；不在目标路径下的逐个入队 `data_chase`。
- 若函数既有 cmdId 分支，又在 `switch`/`if` 之前无条件消费 data（如 `ParseCommon(data)`），该无条件调用按 `"*"` 规则记录，不归属任何具体 case。
- 一个分支只有内联 data 处理、没有调用目标 → 分发函数定义在 `<target_path>` 下时，把自身记为接口函数，`cmd_id_macro` / `cmd_id_value` 取 `effective_outer`；没有外层时 `cmd_id_macro = "*"`、`cmd_id_value = null`，并在该 case 的 `evidence` 中记录内联处理代码原文。不在 `<target_path>` 下时，记 `unresolved`（`kind = "target_path"`），detail 记录分发函数与内联处理位置。
- `default` 分支只写入 `default_cases`（记录 dispatcher、分发位置与 evidence），不进入最终 JSON。
- 无 cmdId 分支、函数自身直接消费 data（不经过调用）→ 在 `<target_path>` 下记为 `cmd_id_macro = "*"`、`cmd_id_value = null`；否则记 `unresolved`（`kind = "target_path"`）。
- 无 cmdId 分支、无条件调用 data 消费函数 → 目标在 `<target_path>` 下时记为 `cmd_id_macro = "*"`、`cmd_id_value = null`；否则入队 `data_chase`。两种情况都在 evidence 中说明“无条件直接处理”；无当前 case 时写入追踪链。
- 只接收 cmdId、不接收 data、也不分发，或只接收 reply、不读 data 的目标，不得写入 `handlers`。
- 纯日志/断言/框架转换/校验辅助等被忽略的调用，把调用原文与忽略原因追加到当前 case 的 `evidence`，不得静默丢弃。

### 3.7 C/C++ 目标解析

- 支持直接调用 `Foo(...)`、成员调用 `obj.Foo()` / `obj->Foo()`、`this->Foo()`、虚函数、函数指针、表项、注册回调、宏包装后的调用名。
- 虚函数/重载：能在 scope 内静态确定唯一目标时取精确目标；不能唯一确定时，枚举 scope 内全部可识别候选（不设数量上限），逐个按 3.6 分类并记录，候选来源写入该 case 的 `evidence`。仍无可识别候选 → `unresolved`（`kind = "target"`），`tried` 列出搜索的符号、文件与拒绝原因。
- 函数指针与表项：沿初始化赋值、表初始化器、注册调用找到实际赋入的函数；若同一指针/表项存在多个可能赋值，枚举全部候选。
- 命名使用定义处的作用域限定名（C++ 如 `Namespace::Class::Method`；C 函数用函数名）。重载以 `(file, definition_line)` 区分，限定名相同不得合并。
- lambda 使用 `<lambda@绝对路径:行号>` 命名；直接调用时分析其函数体，作为回调/表项传入时解析到该 lambda。
- 接口函数定义必须在 scope 内且必须是带完整函数体的定义；只有声明、定义在 scope 外或无法确定定义位置 → `unresolved`（`kind = "definition"`），不得写入 `handlers`，不得输出。
- `stub->SendRequest` 特殊处理：优先使用阶段二记录的 `stub.on_remote_request`；缺失时再用 `stub.stub_class`、`stub.interface_type`、`stub.impl_class` 信息在 scope 内定位对应的 `Stub::OnRemoteRequest`；把 SendRequest 的 cmdId 实参映射为 OnRemoteRequest 的 `code`，data/parcel 实参映射为 data 参数。保留当前外层 cmd（无外层时新工作项外层为 null，OnRemoteRequest 的 case 因此成为外层）。找不到 OnRemoteRequest 定义 → `unresolved`（`kind = "definition"`），记录 stub 类名、接口名与已搜索位置。
- 直接调用 `OnRemoteRequest`、调用已知 dispatcher、经宏/包装函数转发到上述目标的，按同样的映射与入队规则处理。
- 每次解析调用目标后写/更新内存中的 `dispatch_cache`（`site`、`target`、`candidates`、`resolved`）；同一 site+target 复用缓存结果，不重复搜索。
- `stub` 为 null 时相关字段按 null 处理；SendRequest 接收者可静态识别且与阶段二记录的 stub 成员/类不匹配时，写 `unresolved`（`kind = "target"`），不静默映射到 `on_remote_request`。

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
- `effective_outer`：触发追踪的最外层 cmd 表达式与值，无外层时为 null；
- `origin_case_id`：触发追踪的原 case id，无条件处理时为 null；
- `path = 父工作项.path + [目标限定名]`；
- `evidence_chain`：从触发追踪的原目标到当前函数的调用链文本。

入队前做环检测与去重：

- `dispatch` 环检测键：`(函数定义文件+行号, current_cmd_expr, current_cmd_value, outer_cmd_expr, outer_cmd_value, sorted(data_vars), data_derived)`。不同 data 上下文（形参集合或派生状态不同）产生不同键，必须分别入队。
- `data_chase` 环检测键：`(kind="data_chase", 定义文件+行号, effective_outer.expr, effective_outer.value, sorted(data_vars), origin_case_id)`。
- 键命中 `queue`（按对应 `kind` 的键）：静默去重，不入队，不写 `unresolved`。
- 键命中 `visited`：以准备入队的候选工作项的 `path` 为准；若该 visited 条目的 `path` 是候选 `path` 的严格前缀 → 真环，写 `unresolved`（`kind = "cycle"`），detail 记录该键与路径，终止该分支；否则是菱形/重复路径，静默去重，不写 `unresolved`。
- 同一目标以不同的 cmd 上下文到达时键不同，必须分别入队；不得因函数名相同而合并。
- 不设深度上限、不设扇出上限：同一分支的全部转发器/内层 dispatcher 都入队，按队列逐个处理。

### 3.9 target_path 深入追踪

处理 `kind = "data_chase"` 工作项：

1. 按 3.2 的同样方法读取函数完整函数体；定义不在 scope 内或无法唯一确定 → 记 `unresolved`（`kind = "definition"`），结束该工作项。
2. 按 3.5 找出函数体内所有“实参含 data 或 data 派生值”的调用。
3. 对每个调用按 3.7 解析目标定义，然后分类：
   - 定义不在 scope、无法唯一确定或函数体不完整 → 记 `unresolved`（`kind = "definition"`），继续下一个调用；
   - 已知转发器（`SendRequest`、`OnRemoteRequest`、已知 dispatcher）→ 按 3.8 入队 `kind = "dispatch"` 工作项，`current_cmd_expr` 取调用点传入的 cmdId 实参，`parent_case_id = origin_case_id`，其余字段按 3.8 构造；
   - 接收 cmdId（包括从 data 派生得到的 cmdId）且函数体内有分发逻辑 → 按 3.8 入队 `kind = "dispatch"` 工作项，`current_cmd_expr` 取调用点传入的 cmdId 实参，`parent_case_id = origin_case_id`，其余字段按 3.8 构造；
   - 排除项 → 忽略，把调用原文与忽略原因追加到 `origin_case_id` 对应 case 的 `evidence`；`origin_case_id` 为 null 时写入当前追踪链；
   - 接收 data 的业务函数，定义在 `<target_path>` 下 → 记为接口（见第 4 条）；
   - 接收 data 的业务函数，定义不在 `<target_path>` 下 → 按 3.8 入队新的 `kind = "data_chase"` 工作项；
   - 不接收 data/派生参数 → 不追踪。
4. 命中 `<target_path>` 业务函数时：
   - `interface_function`、`file`、`definition_line` 取该函数；
   - 把 `{cmd_id_macro, cmd_id_value, interface_function, file, definition_line}` 写入顶层 `handlers`，按现有五元组去重；
   - `cmd_id_macro` / `cmd_id_value` 取 `effective_outer`；无外层时 `cmd_id_macro = "*"`、`cmd_id_value = null`；
   - 把 `{name, file, line}` 追加到 `origin_case_id` 对应 case 的 `handlers`；`origin_case_id` 为 null 时不追加；
   - 在对应 case 的 `evidence` 追加追踪链（起点 → ... → 目标(data)）；
   - 按现有 handler 五元组去重。
5. 如果函数自身直接处理 data（不经过调用），记一条 `unresolved`（`kind = "target_path"`），detail 记录起点函数与内联处理位置；该记录不影响其他 data 实参调用的继续处理。
6. 对每条未到达 `<target_path>` 且未入队 dispatcher/`data_chase` 的追踪路径，记一条 `unresolved`（`kind = "target_path"`）；如果该函数没有任何 data 实参调用且没有内联处理，也记一条。detail 包含起点函数、`evidence_chain` 和已尝试的 data 实参调用。由 `definition` 或 `cycle` 终止的路径不再重复记 `target_path`；已入队的路径由后续工作项处理。
7. 真环 → `unresolved`（`kind = "cycle"`）；队列重复/菱形重复静默去重。
8. 不设深度/扇出上限；同一函数有多个 data 实参调用时全部处理。

### 3.10 终止条件、去重与收尾

单个分支在以下任一条件满足时终止：

1. 找到 `<target_path>` 下接收 data 且不构成分发的业务函数（接口函数已写入 `handlers`）；
2. 没有未处理的 data 消费目标（纯 cmdId 处理、纯转发已入队、data 消费目标已入队 `data_chase`、或目标被忽略）；
3. 目标无法确定，已写入 `unresolved`；
4. 真环命中（键在 `visited` 且其 `path` 是当前 `path` 的严格前缀）；队列重复/菱形重复按去重终止，不写 `unresolved`。

`data_chase` 工作项在 3.9 的全部 data 实参目标分类完成后终止；继续入队的 dispatcher 或 `data_chase` 作为新的工作项处理，不阻塞当前项移除。

- 分支终止后不得继续展开该分支；其他分支、其他工作项继续。
- 同时分发并处理 data 的函数按 dispatcher 处理：继续分析其分支，不因它消费 data 就提前整体记为接口；只有满足 3.6“内联 data 处理且无调用目标”的分支才把 dispatcher 自身记为接口函数。
- 不设深度上限、不设扇出上限；终止条件之外不得提前退出。

追踪期间不写状态文件。队列为空后，一次性写 `03_trace.json`。

单次运行内的去重规则：

- case 以 `(dispatcher_file, dispatcher_line, dispatch_site.file, dispatch_site.line, case_expr, outer_cmd_expr, outer_cmd_value)` 去重；重复命中时复用已有 case id，合并 evidence 与 handlers。
- handler 以 `(cmd_id_macro, cmd_id_value, interface_function, file, definition_line)` 去重；同一业务函数在不同 cmdId 下分别保留多条记录。
- `default_cases` 以 `(dispatcher_file, dispatcher_line, dispatch_site.file, dispatch_site.line)` 去重。
- `unresolved` 以 `(kind, detail)` 去重，`tried` 合并后去重；同类候选只在首次出现时写 detail，后续补入 tried。
- `dispatch_cache` 以 `(site.file, site.line, target)` 去重，条目包含 `site`、`target`、`candidates`、`resolved`。
- `visited` 条目记录函数定义与上下文：dispatch 项记录 3.8 的 dispatch 字段，data_chase 项记录 3.8 的 data_chase 字段；以对应 kind 的环检测键去重。
- `data_chase` 工作项以 `(kind, 定义文件+行号, effective_outer.expr, effective_outer.value, sorted(data_vars), origin_case_id)` 去重。
- `unresolved.kind` 取值：`cmd_value` / `target` / `definition` / `data_deferred` / `cycle` / `target_path`。

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

- `cases` 元素字段：`id`、`dispatcher`、`dispatcher_file`、`dispatcher_line`、`dispatch_site`、`level`、`case_expr`、`case_value`、`outer_cmd_expr`、`outer_cmd_value`、`parent_case_id`、`handlers`、`evidence`；其中 `handlers` 是 `{name, file, line}` 引用列表。
- 顶层 `handlers` 元素字段：`cmd_id_macro`、`cmd_id_value`、`interface_function`、`file`、`definition_line`；`definition_line` 仅用于定位与去重，最终 JSON 不包含。
- `default_cases` 元素字段：`dispatcher`、`dispatcher_file`、`dispatcher_line`、`dispatch_site`、`evidence`。
- `unresolved` 元素字段：`kind`、`detail`、`tried`；`kind` 取 `cmd_value` / `target` / `definition` / `data_deferred` / `cycle` / `target_path`。

- 写入失败 → 删除可能残留的 `<out>/03_trace.json`，再写 `error.json`（`phase = "trace_dispatch"`，`code = "snapshot_write_failed"`，message/evidence 记录写入错误），停止。
- 写入成功后进入 `## 八、输出与自检`。
- 其他全局失败（工具/文件 I/O 或无法继续读取源码等不可恢复错误）→ 若已有部分追踪结果，先写 `<out>/03_trace.json` 的部分快照，再写 `error.json`（`phase = "trace_dispatch"`，`code = "internal_error"`，message/evidence 记录失败详情），停止；单个分支的解析失败只写 `unresolved`，不中止整体追踪。

## 八、输出与自检

本阶段只做两件事：完成自检，然后把 `03_trace.json` 的 `handlers` 组装为最终 JSON 并写入磁盘。不新增分析、不展开接口函数、不修改追踪结论。

### 8.1 输出 JSON

最终输出：`<out>/<service_name>_api.json`。只从 `03_trace.json` 的 `handlers` 组装：丢弃 `definition_line` 等仅用于定位与去重的字段，每条记录仅保留以下四个字段。

```json
[
  {
    "cmd_id_macro": "CMD_DEMO_INTERFACE_GET_VERSION",
    "cmd_id_value": 0,
    "interface_function": "DemoInterfaceStubGetVersion",
    "file": "/abs/path/demo_interface_stub.cpp"
  }
]
```

字段定义：

| 字段 | 类型 | 规则 |
|------|------|------|
| `cmd_id_macro` | string | 最外层具体 case 的宏名或表达式原文；范围/掩码/组合表达式照原样 |
| `cmd_id_value` | integer 或 null | 单一可解析值；范围/掩码/组合/无法解析时为 null |
| `interface_function` | string | 定义处的作用域限定名；lambda 用 `<lambda@绝对路径:行号>` |
| `file` | string | 函数定义所在文件的绝对路径，必须在 scope_files 内 |

- `cmd_id_value` 直接取自 `03_trace.json` 的 `handlers`，不重新解析；3.4 判定可计算的掩码/组合保留整数，只有范围、不可计算或无法解析时为 null。
- `cmd_id_macro` 也可以为 `"*"`（3.6 的无条件/内联处理）；此类记录按原样保留，`cmd_id_value` 为 null，正常参与去重与排序。仅 `default` 分支不输出。
- 示例中的 `DemoInterfaceStubGetVersion` 是全局函数；若是命名空间/类成员，必须写定义处完整限定名。

输出规则：

1. 同一 cmdId 对应多个接口函数 → 多条记录。
2. 完全相同的四元组去重。四元组去重是最终输出的归一化规则；重载若四字段完全相同，会在最终 JSON 中合并为一条。
3. 排序：`cmd_id_value` 升序（null 最后）→ `cmd_id_macro` → `interface_function` → `file`。
4. `default` 分支不进入最终 JSON。
5. 服务存在但未识别到接口 → 输出 `[]`，报告 0 个接口。
6. 不添加任何额外字段。
7. UTF-8 编码，JSON 缩进 2 空格。
8. 先写 `<out>/<service_name>_api.json.tmp`，Read 回读校验通过后 `mv` 为最终文件。
9. 最终文件必须存在，否则本次任务失败。

组装步骤：

1. 遍历 `03_trace.json` 的 `handlers`，每条只取 `cmd_id_macro`、`cmd_id_value`、`interface_function`、`file`，其余字段一律丢弃。
2. 按四元组 `(cmd_id_macro, cmd_id_value, interface_function, file)` 去重，完全相同的记录只保留一条；`cmd_id_value` 为 null 也参与比较。
3. 按输出规则 3 排序；`cmd_id_value` 为 null 的记录排在全部整数之后，其余键按字符串升序比较。
4. 没有记录时输出空数组 `[]`，不输出 null 或空文件。
5. 临时文件为 `<out>/<service_name>_api.json.tmp`，最终文件为 `<out>/<service_name>_api.json`。

### 8.2 自检

自检以 `03_trace.json` 为准；队列与去重结果已在阶段三的内存追踪中处理，不再有 status 字段或 phase 转换。执行以下清单：

1. 队列已处理完（所有工作项已从内存队列移除）。
2. 每个 `handlers` 条目都能在 scope_files 内定位到函数定义，且函数名为限定名。
3. `default_cases` 没有混入 `handlers`。
4. `handlers` 已按阶段三的单次运行去重规则处理（不要求四元组去重或排序）。
5. `unresolved` 已记录；报告数量以此长度为准。
6. 最终 JSON 恰好包含四个字段，无额外字段；已按四元组去重并执行排序；UTF-8 编码，缩进 2 空格。
7. 写临时文件后 Read 回读，核对数组长度、字段名、类型。
8. `mv` 后双方法确认最终文件存在。

执行说明：

- 第 1 项检查内存队列；第 2-5 项检查 `03_trace.json`；第 6 项针对 8.1 组装后的最终数组核对。
- 第 2 项中 lambda 的 `interface_function` 以 `<lambda@绝对路径:行号>` 形式为合规；其余函数必须是定义处作用域限定名。每个条目都要按 `file` 与定义位置 Read 确认完整函数体，且 `file` 必须在 `scope_files.txt` 内。
- 第 3 项检查没有任何 handler 的 `cmd_id_macro == "default"`，且不存在仅由 `default_cases` 站点支撑的 handler。
- 第 5 项 unresolved 数量以 `03_trace.json` 的 `unresolved` 长度为准，报告直接使用该长度。
- 第 7 项回读后核对：数组长度等于去重后的记录数；每条键集合恰为四字段；`cmd_id_macro` / `interface_function` / `file` 为字符串，`cmd_id_value` 为整数或 null。必须读到文件末尾；截断视为回读失败。
- 第 8 项双方法确认最终文件存在（`ls` + `Read`），禁止 `test -f` / `[ -f ]`。

执行顺序：

1. 执行清单第 1-5 项。
2. 按 8.1 组装最终数组，执行清单第 6 项。
3. 写 `<out>/<service_name>_api.json.tmp`，执行清单第 7 项。
4. `mv` 为最终文件，执行清单第 8 项。
5. 按 8.3 输出完成报告。

失败处理：

- 清单第 1-6 项任一失败 → 写 `error.json`（`phase = "output"`，`code = "self_check_failed"`，message/evidence 记录失败项编号与证据），停止；不得写最终 JSON。
- 清单第 7-8 项（写临时文件、回读校验、`mv`、最终文件存在性确认）失败 → 删除可能残留的 `<out>/<service_name>_api.json.tmp`，写 `error.json`（`phase = "output"`，`code = "output_write_failed"`，message/evidence 记录临时/最终路径与失败动作），停止。
- 最终文件必须存在；无法确认存在即按 `output_write_failed` 处理，本次任务失败。

### 8.3 完成报告

写盘成功并确认最终文件存在后报告：

- 分析范围文件数；
- service_name、入口变量、Bind 函数、Dispatch 函数数量；
- cmdId 用例数（取 `03_trace.json` 的 `cases` 长度，不含 `default_cases`）；
- 接口函数数量；
- unresolved 数量；
- 最终输出文件绝对路径。

接口函数数量以最终 JSON 数组长度为准；unresolved 数量以 `03_trace.json` 的 `unresolved` 长度为准；服务存在但未识别到接口时报告 0 个接口，并说明最终 JSON 为 `[]`。

## 九、权限申请

| 工具 | 用途 |
|------|------|
| `Bash` | find/grep/xargs/ast-grep/ls/mkdir/mv/rm，scope 内搜索、备份/清理输出目录 |
| `Read` | 读取源码、macro.json、clean_code.txt/.etignore、阶段快照、待回读的临时 JSON |
| `Write` | 写入 scope_files.txt、scope_roots.txt、01_entry.json、02_bind.json、03_trace.json、error.json、最终 JSON（含临时文件） |
| `mcp__codegraph__codegraph_explore` | （可选）.codegraph 存在时优先的代码探索工具 |
| `mcp__plugin_oh-my-claudecode_t__ast_grep_search` | （可选）AST 模式搜索 |

codegraph 未安装/未配置 → 用常规工具代替，不中断分析。

## 十、约束规则

1. 全部由主 agent 完成，不调用 sub-agent。
2. 质量优先；不设深度/扇出上限；所有分支追到接口函数或明确终止条件。
3. clean_code.txt 是硬边界；grep 只用 scope_files/scope_roots；codegraph 结果按 scope 过滤；禁止全仓 grep。
4. 只输出 `<target_path>` 下真正处理 data 的业务函数（含 3.6 的 `*` 无条件/内联处理记录）；明显单线封装继续下钻，找不到真正处理函数时输出最深的 `<target_path>` 封装函数。目标路径外只沿 data 实参继续追踪，不展开最终接口内部实现。
5. 接口函数必须能在 scope 内定位到定义；范围外记 unresolved，不输出。
6. 不臆造：无法解析的宏/目标写明尝试过的搜索方式。
7. 输出路径一律绝对路径；函数名带作用域限定；重载用文件+行号区分。
8. 每个阶段结束写一次快照；最终 JSON 最后一次性原子写出。
9. 关键文件存在性双方法确认，禁止 `test -f`。
10. 全中文提示词，同一概念用词前后一致。
11. 所有 shell 命令中的路径变量使用双引号；`<project_dir>` 按 shell 规则转义。

### 全局错误码

| 错误码 | 触发条件 | 处理 |
|--------|----------|------|
| `usage_error` | 参数缺失、过多、为空，或 service_name 含非法字符 | 打印用法，结束 |
| `invalid_project_dir` | project_dir 不存在或不是目录 | 报告，结束 |
| `output_dir_failed` | 备份旧输出目录或创建新输出目录失败 | 报告，结束 |
| `empty_scope` | 分析范围为空 | 写 error.json（phase = init），报告，结束 |
| `service_not_found` | 找不到 moduleName 精确匹配的 HdfDriverEntry | 报告范围文件数、搜索模式，结束 |
| `multiple_matches` | 多个候选无法唯一确定 | 列出候选（变量、文件、行号），结束 |
| `bind_not_found` | 入口结构体没有有效 `.Bind` | 报告，结束 |
| `bind_definition_not_found` | 宏/包装无法展开，或展开后没有带完整函数体的 Bind 定义 | 报告，结束 |
| `bind_ambiguous` | 多个同名 Bind 定义无法区分 | 列出候选，结束 |
| `dispatch_not_found` | 找不到 Dispatch 函数 | 报告，结束 |
| `snapshot_write_failed` | 01_entry.json / 02_bind.json / 03_trace.json 写入失败 | 报告，结束 |
| `output_write_failed` | 最终 JSON 写入、回读、mv 或最终文件存在性确认失败 | 报告，结束 |
| `self_check_failed` | 自检清单第 1-6 项失败 | 报告，结束，不写最终 JSON |
| `internal_error` | 工具/文件 I/O 或无法继续读取源码等不可恢复错误 | 报告，结束 |

除 `usage_error` / `invalid_project_dir` / `output_dir_failed` 外，其余错误在 `<out>` 已创建时均先写 `error.json` 再结束。

### 术语表

| 用词 | 含义 |
|------|------|
| 服务注册点 | 通过 moduleName 精确定位到的 `HdfDriverEntry` 定义及其注册引用 |
| 服务入口变量 | `HdfDriverEntry` 类型的变量，如 `g_DemoDriverEntry` |
| Bind 函数 | 入口结构体 `.Bind` 字段指向的函数 |
| Dispatch 函数 | Bind 阶段链接到 Host 的 `ioService.Dispatch`（或等价成员）的函数 |
| 分发函数 | 根据 cmdId 选择处理路径的函数，包括 Dispatch、转发器、OnRemoteRequest、内层分发函数 |
| cmdId 用例 | 分发结构中的一个具体 case 或条件分支 |
| 外层 cmd | 调用链上最外层的具体 cmdId 用例的宏/表达式和值 |
| 接口函数 | cmdId 用例分支中直接接收客户端 data（或 data 派生值）且定义在 `<target_path>` 下的业务函数 |
| 封装函数 | target_path 内只做校验/日志/别名，并将 data 单线传给名称相近下游的函数 |
| fallback_handler | 已发现的最深 target_path 封装函数；未找到真正处理函数时作为外部接口 |
| 工作项 | 追踪队列中的单元，`kind` 为 `dispatch`（分发函数、当前 cmdId、外层 cmd、data 变量、`data_derived`、调用路径）或 `data_chase`（目标函数、data 变量、`effective_outer`、`origin_case_id`、调用路径） |
| 中间结果 | `scope_files.txt`、`scope_roots.txt`、`01_entry.json`、`02_bind.json`、`03_trace.json`、`error.json` 等调试文件 |
| scope_files | 分析范围内展开后的精确文件清单 |
| scope_roots | clean_code.txt 中原始的文件/文件夹条目清单 |
| target_path | 外部接口落地区域，默认 `<project_dir>/vendor/hisi/`，可人工修改 |

## 十一、抗理性化检查

| 想法 | 现实 |
|------|------|
| “分发层级太深，先记到这里” | 不设深度上限，追到接口函数或明确终止条件。 |
| “候选太多，只取前 N 个” | 不设扇出上限，全部枚举。 |
| “宏算不出来，跳过这个 case” | 保留记录，value 记 null。 |
| “default 也算接口” | default 只进 default_cases，不进最终 JSON。 |
| “这个函数像 helper，跳过” | 读函数体确认，不能凭名字判断。 |
| “范围外可能有定义，去全仓 grep 一下” | 禁止；scope 是硬边界。 |
| “短函数名就够了” | 必须带作用域限定名。 |
| “阶段结果最后一起写” | 每个阶段结束写一次快照；最终 JSON 最后写出。 |
| “想复用上次结果” | 每次执行先备份旧输出目录，再从零分析，不复用历史结果。 |
| “用 sub-agent 并行快一点” | 禁止；全部由主 agent 完成。 |
| “只收 reply 也算接口” | 纯 reply、不读 data 的函数不算。 |
| “业务函数内部还有处理，继续展开” | `<target_path>` 外只沿 data 实参追踪到目标路径；`<target_path>` 内只对明显单线封装继续下钻，不展开最终接口实现。 |

## 十二、使用示例

### 示例一：首次分析

```text
/hal-service-api-finder /srv/ohos/drivers demo_service
```

1. 初始化：校验参数；若 `/srv/ohos/drivers/.ethunter_out/hal-service-api-finder` 已存在，先删除旧 `<out>_bak`（若存在），再把 `<out>` 备份为 `<out>_bak`，新建空目录；确定分析范围并生成 `<out>/scope_files.txt`（存在 `clean_code.txt` 时另生成 `<out>/scope_roots.txt`）。
2. 阶段一：在 scope 内搜索 `"demo_service"`，定位 `struct HdfDriverEntry g_DemoDriverEntry = {...}` 与 `HDF_INIT(g_DemoDriverEntry)`；写 `<out>/01_entry.json`。
3. 阶段二：定位 `HdfDemoDriverBind` 定义，识别 Host 类型与 `ioService.Dispatch = DemoDriverDispatch`，提取 `IDemoInterface`、`DemoService`、`DemoInterfaceStub` 及其 `OnRemoteRequest`；写 `<out>/02_bind.json`。
4. 阶段三：以 `DemoDriverDispatch` 初始化内存队列，追踪 `SendRequest` 到 `DemoInterfaceStub::OnRemoteRequest`，枚举 8 个 cmdId 用例并解析宏值；对不在 `<target_path>` 下的直接目标继续沿 data 实参追踪，跳过明显封装函数，直到 `DemoVendorInterface(data)` 真正处理 data；若只到达封装层，则记录最深的 `DemoVendorInterfaceWrapper`；队列为空后写 `<out>/03_trace.json`。
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

1. 初始化与 scope 生成正常，`<out>/scope_files.txt` 共 127 个文件。
2. 阶段一按「精确字符串搜索」查找 `"demo_service_typo"`，0 命中。
3. 写 `<out>/error.json = {phase: "locate_entry", code: "service_not_found", message: ..., evidence: ...}`：`message` 记录 scope 文件数 127 与搜索模式 `"demo_service_typo"`，`evidence` 明确写“0 命中”；停止，不写 `01_entry.json`，不进入阶段二。
4. 报告：`service_not_found`，分析范围 127 个文件，已搜索模式 `"demo_service_typo"`；建议检查 service_name 拼写（如 `demo_service`），并确认 `clean_code.txt` 覆盖了该驱动的定义文件。
