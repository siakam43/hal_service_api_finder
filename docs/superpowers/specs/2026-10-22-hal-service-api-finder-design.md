# hal-service-api-finder 设计规格

- 日期：2026-10-22
- 状态：设计已确认，待实现
- 交付物：`hal-service-api-finder/SKILL.md`（纯 markdown，无脚本，无 sub-agent）

## 1. 背景与目标

开发一个纯 markdown 的 skill，名为 `hal-service-api-finder`。该 skill 根据 OpenHarmony/HDF HAL 层 service 的注册流程，识别指定 service 对外提供的全部接口函数。

核心判断链路：

```text
HdfDriverEntry.moduleName 精确定位
  → HdfDriverEntry 初始化（.Bind）
  → Bind 函数
  → Host 结构体 + Dispatch 函数
  → cmdId 分发
  → 每个 cmdId 用例直接调用的接口函数
```

最终输出 cmdId 到接口函数的 JSON 映射，供后续安全攻击面分析使用。

## 2. 术语

| 用词 | 含义 |
|------|------|
| 服务注册点 | 通过 moduleName 精确定位到的 `HdfDriverEntry` 定义及其注册引用 |
| 服务入口变量 | `HdfDriverEntry` 类型的变量，如 `g_DemoDriverEntry` |
| Bind 函数 | 入口结构体 `.Bind` 字段指向的函数 |
| Dispatch 函数 | Bind 阶段链接到 Host 的 `ioService.Dispatch`（或等价成员）的函数 |
| 分发函数 | 根据 cmdId 选择处理路径的函数，包括 Dispatch、转发器、OnRemoteRequest、内层分发函数 |
| cmdId 用例 | 分发结构中的一个具体 case 或条件分支 |
| 外层 cmd | 调用链上最外层的具体 cmdId 用例的宏/表达式和值 |
| 接口函数 | cmdId 用例分支中直接接收客户端 data（或 data 派生值）的业务函数 |
| 工作项 | 追踪队列中的单元：`(分发函数, 当前 cmdId 变量, 外层 cmd, data 变量, 调用路径)` |
| 中间结果 | `progress.json`、`tmp/entry_state.json`、`tmp/trace_state.json` 等状态文件 |
| scope_files | 分析范围内展开后的精确文件清单 |
| scope_roots | clean_code.txt 中原始的文件/文件夹条目清单 |

## 3. 输入、输出与交付

### 3.1 调用方式

```text
/hal-service-api-finder <project_dir> <service_name>
```

- 两个参数均必填且不能为空；缺失任意一个，打印用法并结束。
- `project_dir` 必须是已存在的目录；相对路径转换为绝对路径后使用。
- `service_name` 是 `HdfDriverEntry.moduleName` 的精确字符串，区分大小写，不做模糊匹配、不做大小写归一化。
- `service_name` 仅允许字母、数字、下划线、点和连字符；不得包含 `/`、`\`、空白、换行、NUL、引号或 shell 元字符；不满足 → 报错 `usage_error` 并结束。
- 参数多于两个时，打印用法并结束。

### 3.2 输出位置

```text
<project_dir>/.ethunter_out/hal-service-api-finder/
  progress.json
  tmp/scope_files.txt
  tmp/scope_roots.txt（仅 clean_code.txt 分支存在）
  tmp/entry_state.json
  tmp/trace_state.json
  <service_name>_api.json
```

- 输出目录不存在时用 `mkdir -p` 创建；创建失败则报告并结束。
- 最终文件名固定为 `<service_name>_api.json`，例如 `demo_service_api.json`。
- 不生成 summary 报告；调试信息全部保存在中间结果中。

### 3.3 交付物

- 只创建 `hal-service-api-finder/SKILL.md`。
- skill 目录内不放置脚本、测试代码或样例工程。

## 4. 总体架构

采用“分阶段状态机 + 工作队列”：

```text
locate_entry → analyze_bind → trace_dispatch → output → done
                    │                │
                    └── entry_state  └── trace_state
```

| 阶段 | 产物 | 说明 |
|------|------|------|
| locate_entry | entry_state.entry | 按 moduleName 精确定位入口、Bind |
| analyze_bind | entry_state.bind | 提取 Dispatch、stub、实现类 |
| trace_dispatch | trace_state | 工作队列追踪 cmdId 分发 |
| output | `<service_name>_api.json` | 自检后组装最终 JSON |
| done | progress.json | 报告完成 |

- 全部由主 agent 完成，不调用 sub-agent。
- 每个工作项处理完立即保存 trace_state，断点最多丢失一个工作项。
- 不设深度上限、不设扇出上限；只有明确的终止条件（见 8.6）。

## 5. 分析与搜索边界

### 5.1 clean_code.txt 优先，且是硬边界

双方法检查 `<project_dir>/.ethunter_out/clean_code.txt` 是否存在。

存在时：

1. 逐行读取，去掉首尾空白；忽略空行和以 `#` 开头的注释行。
2. 以 `/` 开头为绝对路径，否则拼接为 `<project_dir>/<行内容>`。
3. 路径去重、规范化（分隔符统一为 `/`，去掉前导 `./`）。
4. 逐项检查：
   - 普通文件 → 加入 scope；
   - 文件夹 → 递归展开为其中全部常规文件；
   - 不存在 → 忽略，不报错。
5. 生成两个文件：
   - `tmp/scope_files.txt`：展开后的绝对文件路径，去重、排序；
   - `tmp/scope_roots.txt`：原始文件/文件夹条目，去重、排序。
6. scope 为空 → 报错 `empty_scope` 并结束。

**硬边界规则：**

- clean_code.txt 存在时，**不得分析或搜索列表以外的源文件**。
- 所有入口、Bind、Dispatch、接口函数定义、宏定义的查找都只能在 scope 内进行。
- 范围外不搜索；已知目标在范围外记入 `unresolved`，不输出、不臆造。

**搜索纪律（适用于所有范围分支）：**

- 所有代码搜索只允许在 `scope_files.txt` 列出的文件内进行；`scope_roots.txt` 非 null 时才允许按根目录搜索。
- `grep -I` 必须保留，跳过二进制文件，避免大工程中无意义的慢搜索。
- 两种搜索位置模板（文件参数必须来自 scope 清单；允许附加 `-w`、`-E`、`-F`、`-H`、`--` 等 grep 参数），其中 `<out>` = `<project_dir>/.ethunter_out/hal-service-api-finder`：

```bash
# 方式一：精确文件清单（-H 保证输出带文件名）
xargs -r -a "<out>/tmp/scope_files.txt" -d '\n' grep -I -H -n -F "<pattern>"

# 方式二：原始根目录，仅 scope_roots.txt 非 null 时使用
xargs -r -a "<out>/tmp/scope_roots.txt" -d '\n' grep -I -rn -F "<pattern>"
```

- 需要正则/词边界时，把 `-F` 换成 `-E` 并追加 `-w` 等参数，但文件参数仍必须来自上述清单。
- 禁止 `grep -rn <project_dir>`、`rg <project_dir>`、`find <project_dir> -exec grep` 等任何等价的全仓搜索。
- `xargs`/`grep` 退出码 1 或 123 表示“没有匹配”，不是工具失败；权限/读取错误应如实报告。
- 不得用 `2>/dev/null` 隐藏错误。
- codegraph 返回的每个文件都必须按 `scope_files.txt` 过滤；范围外的搜索命中直接丢弃，不读取、不记录；若分析中已明确知道某个目标定义在范围外，记入 `unresolved`。

### 5.2 clean_code.txt 不存在

1. `.ethunter_out/.etignore` 存在：按下方 find 命令收集 C/C++ 文件，再按 `.etignore` 规则排除；`.etignore` 语法与 `.gitignore` 一致。
2. `.etignore` 也不存在：直接使用下方 find 命令收集 C/C++ 文件。
3. 两种情况的 find 结果都规范化为绝对路径、去重、排序后写入 `scope_files.txt`；结果为空 → 报错 `empty_scope` 并结束。

C/C++ 扩展名集合：

```text
.c .h .cpp .cxx .cc .hpp .hxx .hh .inl .ipp
```

find 模板：

```bash
find <project_dir> -type f \( -name "*.c" -o -name "*.h" -o -name "*.cpp" -o -name "*.cxx" -o -name "*.cc" -o -name "*.hpp" -o -name "*.hxx" -o -name "*.hh" -o -name "*.inl" -o -name "*.ipp" \)
```

无 clean_code.txt 时，`scope_roots` 置为 `null`，该场景不使用根目录 grep，统一使用 `scope_files.txt`。若存在旧的 `tmp/scope_roots.txt`，删除或忽略，避免陈旧状态被误用。

### 5.3 macro.json

- 路径：`<project_dir>/.ethunter_out/macro.json`。
- 按需查阅，不预先全量读取。
- 内容仅代表编译指令中 `-D` 等动态定义的宏，不是全部宏。
- 遇到 `#ifdef` / `#if` 时：先在代码中追溯宏定义，再查 macro.json；两处都找不到，默认分析该分支（不排除）。
- `macro.json` 不存在是正常情况，直接继续分析。
- `#ifdef` 无法判断时分析受保护分支；`#if/#else` 无法判断时两个分支都分析（宁可多分析，不可漏）。

### 5.4 codegraph

- 双方法检查 `<project_dir>/.codegraph` 是否存在。
- 存在时，代码搜索、定义定位、调用关系查询优先使用 `mcp__codegraph__codegraph_explore`。
- 未安装 codegraph 或未配置 MCP → 使用 grep/find/Read/ast-grep，不中断分析。
- codegraph 结果必须按 scope 过滤（见 5.1）。

### 5.5 文件存在性检查

关键文件/目录的存在性判断一律双方法交叉确认，禁止 `test -f` / `[ -f ]`：

| 方法 | 操作 |
|------|------|
| 方法一 | `ls <path>` |
| 方法二 | `Read <path>`（目录返回目录类错误也视为存在） |

| 结果 | 处理 |
|------|------|
| 两次都确认存在 | 存在 |
| 两次都确认不存在 | 不存在，按规则跳过/新建 |
| 两次不一致 | 用第三种方法（如 `find`）确认，取多数 |
| 两次都无法得出确定结论 | 立即停止，向用户报告，不得假设缺失后继续 |

## 6. 阶段一：定位服务注册点

### 6.1 精确搜索

在 scope_files 内搜索 `"<service_name>"` 精确字符串（含引号，`grep -F`）。

对每个命中点：

1. 读取命中行前后足够上下文（建议至少 30 行，直到能看清完整初始化块）。
2. 判断是否位于 `HdfDriverEntry` 类型的初始化中：

```c
struct HdfDriverEntry g_DemoDriverEntry = {
    .moduleVersion = 1,
    .moduleName = "demo_service",
    .Bind = HdfDemoDriverBind,
    .Init = HdfDemoDriverInit,
    .Release = HdfDemoDriverRelease,
};
```

3. 只接受 `.moduleName` 字段与 service_name 完全相等（区分大小写）的初始化。
4. 提取：
   - 入口变量名（如 `g_DemoDriverEntry`）；
   - 定义文件绝对路径、初始化器声明起始行号；
   - `.moduleName`、`.Bind`、`.Init`、`.Release` 的值；
   - 结构体中其他与注册相关的字段（仅记录）。

`.Bind` 缺失、为空、`NULL` 或 `nullptr` → 报错 `bind_not_found`，说明原因并结束。

### 6.2 注册形式确认

首选形式：

```c
HDF_INIT(g_DemoDriverEntry);
```

- 在 scope 内搜索 `HDF_INIT(<入口变量>)`（允许空白差异，也检查 `HDF_INIT(&<入口变量>)`）。
- `HDF_INIT` 必须是有效可执行语句，不能位于注释、字符串或日志中。
- 找到 → `registration.form = "HDF_INIT"`，记录文件、行号、证据。

找不到时：

1. 在 scope 内搜索入口变量的所有引用（word boundary）。
2. 排除定义/初始化本身。
3. 逐个阅读引用上下文，判断是否为其他注册形式（注册函数调用、注册宏、放入驱动表等）；必须是有效可执行语句，不能是注释或日志。
4. 找到 → `registration.form = "other"`，记录文件、行号、证据。
5. 仍找不到 → `registration.form = "unverified"`，记录“moduleName 已精确命中入口结构体，但未发现注册引用”。

同一候选有多处有效注册证据时，记录全部位置；`registration.file` / `registration.line` 取 `scope_files.txt` 顺序中的第一处，`evidence` 列出全部位置。条件编译分支按 5.3 处理。

**只要 moduleName 精确命中的 HdfDriverEntry 存在，就视为找到服务；注册引用只作为证据，不作为硬性失败条件。**

完全找不到 moduleName 精确命中 → 报错 `service_not_found`，报告：
- 分析范围文件数；
- 搜索过的模式；
- 建议检查 service_name 拼写、clean_code.txt 是否包含入口文件。

### 6.3 多命中处理

1. 收集全部候选 `(入口变量, 文件, 行号)`，按文件+行号去重。
2. 唯一候选 → 直接使用。
3. 多个候选：
   - 恰好一个带 `HDF_INIT` 或其他注册引用 → 选它；
   - 多个都带注册引用 → 报错 `multiple_matches`，列出候选并结束；
   - 都无注册引用 → 报错 `multiple_matches`，列出候选并结束。

### 6.4 阶段产物

写入 `tmp/entry_state.json` 的 `entry` 和 `scope` 字段，保存后进入 `analyze_bind`。

## 7. 阶段二：分析 Bind

### 7.1 定位 Bind 定义

1. 在 scope 内搜索 Bind 函数定义；优先选择与入口结构体同一文件的定义。
2. 读到的定义函数名必须与解析后的 Bind 名完全一致（C++ 限定名按限定名匹配）；候选按 `(file, 签名行)` 去重。
3. 若 `.Bind` 的值是宏或包装名，先在 scope 内解析到实际函数名，再定位定义；包装/初始化调用链维护 visited 集合，重复出现即停止并记录环。
4. 若 `.Bind` 是 `Class::Method` 且限定名搜索无定义，定位该类定义，在类体内搜索未限定方法名。
5. 校验存在完整函数体（不是仅声明）。
6. 多个同名定义时：
   - 签名匹配 `int32_t (*)(struct HdfDeviceObject *)` 的定义优先；
   - 仍不唯一 → 报错 `bind_ambiguous`，列出候选并结束。
7. 完全找不到函数体 → 报错 `bind_definition_not_found`，说明原因并结束。

### 7.2 提取 Host 与 Dispatch

完整阅读 Bind 函数体，识别：

1. Host 结构体类型与实例：
   - `new <HostType>`；
   - `<HostType> *host = ...`；
   - 栈/静态实例；
   - `sptr` / `std::unique_ptr` / `std::make_unique` / 工厂函数返回的实例。
   - 找不到 Host 类型时记录为 `null`，继续查找 Dispatch 赋值。

2. Dispatch 赋值，接受以下形式：

```c
host->ioService.Dispatch = X;
host->ioService = { .Dispatch = X, ... };
host.ioService.Dispatch = X;
host->Dispatch = X;              // Host 直接内嵌 IDeviceIoService 时
```

3. 若 Bind 中无 Dispatch：
   - 追踪 Bind 直接调用的初始化函数（继续跟随其直接调用，环检测）；
   - 在 scope 内搜索该 Host 类型的 Dispatch 赋值；
   - 在 scope 内搜索 `IDeviceIoService` 相关的 Dispatch 赋值。
4. 多个 Dispatch 函数全部保留，按函数定义去重，后续逐一追踪。
5. Dispatch 定义不在 scope 内时，`file`/`line` 记 `null`，`via` 记赋值文件:行号与左侧表达式；阶段三跳过该条目并记入 `trace_state.unresolved`。若所有 Dispatch 条目都被跳过且 `stub.on_remote_request` 非 `null`，阶段三直接用该 `OnRemoteRequest` 初始化工作项。
6. 仍找不到 → 报错 `dispatch_not_found`，说明原因并结束。

### 7.3 提取 stub、接口与实现类（可选项）

1. 查找 Host 中名称包含 `stub`（不区分大小写）的成员赋值，或明显的 stub 类型成员赋值。
2. 追踪赋值表达式：
   - `GetOrNewObject(serviceImpl, IDemoInterface::GetDescriptor())`；
   - `new XxxStub(...)`；
   - 其他工厂函数/注册函数。
3. 记录：
   - stub 成员名；
   - 接口类型（如 `IDemoInterface`）；
   - 实现类（如 `DemoService`）及创建位置（必须有类体，不能只是前置声明）；
   - stub 类（如 `DemoInterfaceStub`）。搜索 `<interface_type>Stub` 以及去掉接口名前导 `I` 后的 `<StubName>`，用 `OnRemoteRequest` 交叉确认。
4. 定位 `OnRemoteRequest`：
   - 由接口类型推导 stub 类，搜索 `<StubClass>::OnRemoteRequest` 定义；
   - 或搜索实现类/接口相关类中的 `OnRemoteRequest`；
   - 多个候选时优先精确 stub 类，其次最近的基类；其余候选写入 evidence；找不到则留空，不中断。
5. 这些信息是可选项，缺失不构成阶段失败。

### 7.4 服务注册证据

查找 `deviceObject->service = &host->ioService` 或等价赋值，记录文件、行号、证据。仅作为诊断信息，不作为硬性失败条件；缺失时 `file`/`line`/`evidence` 记 `null`。

### 7.5 阶段产物

阶段开始时若已有 bind 对象，先将 `bind.status` 置为 `in_progress`；提取成功后写 `completed`。写入 `tmp/entry_state.json` 的 `bind` 字段，保存后进入 `trace_dispatch`。

## 8. 阶段三：cmdId 分发追踪

### 8.1 初始化工作队列

1. 为 `entry_state.bind.dispatch_functions` 中每个定义文件非 null 的 Dispatch 函数创建一个工作项；file 为 null 的条目不生成工作项，记入 `trace_state.unresolved`（kind = "definition"）。若所有条目都被跳过且 `bind.stub.on_remote_request` 非 null，则用该 OnRemoteRequest 创建初始工作项。
2. 初始值：
   - `dispatcher`：Dispatch 函数限定名；
   - `current_cmd_expr`：cmdId 参数名（通常 `cmdId`）；
   - `current_cmd_value`：`null`；
   - `outer_cmd_expr` / `outer_cmd_value`：`null`；
   - `data_vars`：目标函数中与客户端 data 对应的形参名（如 `data`）；由 `SbufToParcel` 等转换出的局部变量按 3.5 追踪，不写入 data_vars；
   - `parent_case_id`：`null`；
   - `path`：`[dispatcher]`；
   - `status`：`pending`。
3. Dispatch 若直接调用 `stub->SendRequest(cmdId, ...)`，将对应 stub 的 `OnRemoteRequest` 作为新工作项入队，外层仍为空。
4. 每个工作项按以下流程处理，处理完立即落盘。

### 8.2 识别分发结构

完整阅读分发函数体（含全部分支；`#ifdef` 结合 macro.json），识别：

- `switch (cmdId)` 的 `case` / `case A ... B` / 多标签共享分支 / `default`；
- `if / else if` 对 cmdId 或派生值的比较、范围判断、掩码判断；
- 数组/映射/函数指针表：以 cmdId 或派生键索引；
- 表项注册：初始化列表、`insert`、注册函数调用；
- 宏生成的分发代码；
- `SendRequest` / `OnRemoteRequest` / 已知 dispatcher 的转发；
- 没有 cmdId 分支、直接调用 data 消费函数的情况。

### 8.3 每个 cmdId 用例

#### 8.3.1 外层归属

```text
effective_outer = 当前分支已确定外层 ?? 工作项.outer ?? 本用例

- 内层 case 继承其外层 case 的 effective_outer，不得用内层 case 覆盖外层。
```

- 工作项外层为空 → 本用例成为外层，记录其宏/表达式和值。
- 工作项外层已存在 → 保留外层；本用例只作为内层证据。
- 因此 `Dispatch → SendRequest → OnRemoteRequest` 时，`OnRemoteRequest` 的 switch case 成为外层 cmd。
- 嵌套分发的工作项继承 `effective_outer`，最终接口函数记录到该外层 cmd 下。

#### 8.3.2 cmdId 数值解析

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

**具名 cmdId 即使值无法解析，也必须保留在最终输出中（值为 `null`）。**

#### 8.3.3 分支分析

对分支内语句做轻量 data 追踪：

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

对分支内每个调用，按以下顺序判定，命中即停（保证各行互斥）：

1. 已知转发器；
2. 接收 cmdId 且函数体内有分发逻辑；
3. 排除项：纯日志、断言、框架 sbuf/parcel 转换、纯校验辅助、无业务动作、只收 cmdId 不收 data 也不分发、只收 reply 不读 data；
4. 接收 data/派生参数且不构成分发的业务函数 → 接口函数候选；
5. 其余 → 忽略。

| 目标特征 | 处理 |
|----------|------|
| 已知转发器（`SendRequest`、`OnRemoteRequest`、已知 dispatcher） | 入队为新工作项 |
| 接收 cmdId 且函数体内有分发逻辑 | 入队为内层 dispatcher |
| 接收 cmdId + data，且函数体内有分发 | 先作为分发函数入队，不直接记为接口 |
| 接收 data/派生参数，且不构成分发 | 记为接口函数候选 |
| 纯日志、断言、框架 sbuf/parcel 转换、纯校验辅助、无业务动作 | 忽略，写入 evidence |
| 只接收 cmdId、不接收 data、也不分发 | 忽略 |
| 只接收 reply、不读 data | 不记为接口函数 |

- 一个分支有多个 data 消费目标 → 多条记录。
- 一个分支只有内联 data 处理、没有调用目标 → 把分发函数自身记为接口函数，附 evidence。
- `default` 分支只写入 `trace_state.default_cases`，不进入最终 JSON。
- 无 cmdId 分支、无条件直接调用 data 消费函数 → 记为 `cmd_id_macro = "*"`、`cmd_id_value = null`，避免漏接口；evidence 说明“无条件直接处理”。

### 8.4 调用目标解析（C/C++）

C++ 间接调用、虚函数、函数指针、重载等解析细节可参考 taint-path-walker skill 的规则；本 skill 只追踪到 cmdId 处理位置，不要求完整污点传播路径。

- 支持直接调用、成员调用、`this->` 调用、虚函数、函数指针、表项、注册回调。
- 静态可确定 → 精确解析唯一目标。
- 不可静态确定 → 在 scope 内枚举全部可识别候选；不设数量上限。
- 仍无法确定 → 写入 `unresolved`，写明已尝试的搜索方式，不臆造。
- `stub->SendRequest` 特殊处理：
  1. 用阶段二记录的接口类型/实现类/stub 类定位具体 `Stub::OnRemoteRequest`；
  2. 把 `SendRequest` 的 cmdId 参数映射为 `OnRemoteRequest` 的 `code` 参数；
  3. 把 data/parcel 参数映射为 `OnRemoteRequest` 的 data 参数；
  4. 外层 cmd 保持不变；
  5. 找不到 `OnRemoteRequest` → 记 `unresolved`，不中断其它分支。
- C++ 函数名一律使用定义处的作用域限定名（命名空间 + 类 + 方法，`::` 连接）。
- 重载/同名函数用“定义文件 + 行号”区分，不只靠短名。
- lambda 记为 `<lambda@绝对路径:行号>`。
- 接口函数定义必须在 scope_files 内；范围外或找不到定义 → `unresolved`，不进入最终 JSON。
- 虚函数/函数指针候选全部枚举；若候选集合中有多个定义，全部追踪，不因数量截断。

### 8.5 嵌套分发与证据

- 每个 case 记录：`level`（距初始 dispatcher 的层数，仅用于诊断）、`dispatch_site`、`case_expr`、`case_value`、`outer_cmd_expr`、`outer_cmd_value`、`parent_case_id`、`handlers`、`evidence`。
- 嵌套分发产生的新工作项记录 `parent_case_id`，便于回溯。
- 外层 cmd 在整条调用链上保持不变；一个外层 cmd 可达多个叶子接口 → 多条最终记录。

### 8.6 终止条件

不设深度上限、不设扇出上限。工作项只在以下情况结束：

1. 分支内找到 data 消费的直接目标业务函数 → 记为接口函数，停止深入该业务函数内部；
2. 分支内确实没有 data 消费目标 → 记录 evidence，结束该分支；
3. 目标无法静态确定，或定义不在 scope 内 → 记入 `unresolved`；
4. 同一 `(函数定义文件+行号, current_cmd_expr, current_cmd_value, outer_cmd_expr, outer_cmd_value, sorted(data_vars), data_derived)` 重复出现，且 `visited` 条目的 `path` 是候选工作项 `path` 的严格前缀 → 记 `cycle`，停止该分支；队列重复/菱形重复按去重终止，不写 `unresolved`。不同 data 上下文（形参集合或派生状态不同）产生不同键，必须分别入队。

- 环检测和 `visited` 去重只用于防止无限递归，不作为数量限制。
- 若函数既做分发又直接处理 data：按分发函数继续追踪；只有不构成分发的 data 消费函数才作为接口函数。
- 所有静态可确定的分发/转发目标全部枚举，不因分发阶段数或候选数量截断。

### 8.7 工作项落盘

每处理完一个工作项立即更新并保存 `tmp/trace_state.json`：

- `queue`：工作项及 `pending/in_progress` 状态；
- `cases`：已发现用例、层级、外层 cmd、分支证据、候选接口；
- `handlers`：去重后的最终候选池；
- `default_cases`：default 分支证据；
- `unresolved`：`cmd_value` / `target` / `definition` / `data_deferred` / `cycle`；
- `dispatch_cache`：分发点解析结果缓存；
- `visited`：环检测键；
- `counters`：队列剩余数、用例数、接口数、unresolved 数。

幂等规则：

- handlers 按 `(cmd_id_macro, cmd_id_value, interface_function, file)` 去重；
- cases 按 `(dispatcher 定义文件+行号, dispatch_site 行号, case_expr, outer_cmd_expr, outer_cmd_value)` 去重；
- 恢复时重做 `in_progress` 工作项的全部分支；已 `analyzed` 的 case 通过去重合并 handlers，不重复输出；
- 重新处理同一工作项不得产生重复记录。

## 9. 最终 JSON 输出

文件：`<project_dir>/.ethunter_out/hal-service-api-finder/<service_name>_api.json`

扁平数组，严格四个字段：

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

| 字段 | 类型 | 规则 |
|------|------|------|
| `cmd_id_macro` | string | 最外层具体 case 的宏名或表达式原文；范围/掩码/组合表达式照原样 |
| `cmd_id_value` | integer 或 null | 单一可解析值；范围/掩码/组合/无法解析时为 null |
| `interface_function` | string | 定义处的作用域限定名；lambda 用 `<lambda@绝对路径:行号>` |
| `file` | string | 函数定义所在文件的绝对路径，必须在 scope_files 内 |

- `cmd_id_value` 直接取自 `trace_state.handlers`，不重新解析；3.4 判定可计算的掩码/组合保留整数，只有范围、不可计算或无法解析时为 null。
- `cmd_id_macro` 也可以为 `"*"`（3.6 的无条件/内联处理），正常输出；仅 `default` 分支不输出。
- 四元组去重是最终归一化规则；重载若四字段完全相同则合并为一条。

输出规则：

1. 同一 cmdId 对应多个接口函数 → 多条记录。
2. 完全相同的四元组去重。
3. 排序：`cmd_id_value` 升序（null 最后）→ `cmd_id_macro` → `interface_function` → `file`。
4. `default` 分支不进入最终 JSON。
5. 服务存在但未识别到接口 → 输出 `[]`，报告 0 个接口。
6. 不添加任何额外字段。
7. UTF-8 编码，JSON 缩进 2 空格。
8. 先写 `tmp/<service_name>_api.json.tmp`，Read 回读校验通过后 `mv` 为最终文件，避免半成品。
9. 最终文件必须存在，否则本次任务失败。

## 10. 状态文件与断点恢复

状态文件路径：
- `<out>/progress.json`
- `<out>/tmp/entry_state.json`
- `<out>/tmp/trace_state.json`

以下 JSON 为字段形状示例，不要求彼此处于同一时刻。

### 10.1 progress.json

```json
{
  "phase": "locate_entry",
  "service_name": "demo_service",
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

- `phase` 取值：`locate_entry` / `analyze_bind` / `trace_dispatch` / `output` / `done`。
- 阶段状态：`pending` / `in_progress` / `completed`。
- 每次阶段切换立即保存。
- 全新分析开始时，先创建初始 progress.json：写入 service_name/project_dir/scope，phase = locate_entry，locate_entry.status = in_progress，其余阶段 status = pending，error = null；写入方式为 .tmp + mv。
- 失败时不新增 `failed` 阶段：保留当前 `phase`，写入：

```json
"error": {
  "code": "<错误码>",
  "message": "<说明>",
  "evidence": "<证据或已尝试的搜索方式>"
}
```

- 下次运行从当前阶段重试，重试前清空 `error`。
- phase 是恢复的唯一权威；各阶段 `status` 仅作诊断，必须与 `phase` 同步更新。
- `scope.file_count` 在 scope 生成后写入；恢复时与 `scope_files.txt` 的行数核对，不一致以实际文件清单为准并更新。
- 只有 `trace_state.status == completed` 后，`phase` 才能设为 `output`；`output` 完成后才能设为 `done`。

### 10.2 entry_state.json

```json
{
  "service_name": "demo_service",
  "scope": {
    "file_count": 123,
    "clean_code": true,
    "scope_files": "tmp/scope_files.txt",
    "scope_roots": "tmp/scope_roots.txt"
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
  },
  "bind": {
    "status": "completed",
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

- `registration.form`：`HDF_INIT` / `other` / `unverified`。
- `bind.status`：`pending` / `in_progress` / `completed`；阶段二开始时若已有 bind 对象，必须先将 `bind.status` 置为 `in_progress`，失败时保持该值。
- `stub` 可以为 `null`；`host_type` 可以为 `null`；`service_registration` 的 `file`/`line`/`evidence` 可以为 `null`；`stub.on_remote_request` 可以为 `null`。
- 阶段二可重复执行（幂等）：中断后重跑并覆盖，不影响后续。
- 恢复时校验 `entry_state.service_name` 与当前 `service_name` 一致；不一致视为身份不一致，报 `state_conflict` 并停止，不混用旧状态。

### 10.3 trace_state.json

```json
{
  "status": "in_progress",
  "service_name": "demo_service",
  "entry_dispatchers": [
    {"name": "DemoDriverDispatch", "file": "/abs/path/...", "line": 129}
  ],
  "queue": [
    {
      "id": 1,
      "dispatcher": "DemoDriverDispatch",
      "dispatcher_file": "/abs/path/...",
      "dispatcher_line": 129,
      "current_cmd_expr": "cmdId",
      "current_cmd_value": null,
      "outer_cmd_expr": null,
      "outer_cmd_value": null,
      "data_vars": ["data"],
      "data_derived": true,
      "parent_case_id": null,
      "path": ["DemoDriverDispatch"],
      "status": "pending"
    }
  ],
  "cases": [
    {
      "id": 1,
      "dispatcher": "OHOS::HDI::Demo::V1_0::DemoInterfaceStub::OnRemoteRequest",
      "dispatcher_file": "/abs/path/...",
      "dispatcher_line": 160,
      "dispatch_site": {"file": "/abs/path/...", "line": 163},
      "level": 2,
      "case_expr": "CMD_DEMO_INTERFACE_GET_VERSION",
      "case_value": 0,
      "outer_cmd_expr": "CMD_DEMO_INTERFACE_GET_VERSION",
      "outer_cmd_value": 0,
      "parent_case_id": null,
      "status": "analyzed",
      "handlers": [
        {
          "name": "DemoInterfaceStubGetVersion",
          "file": "/abs/path/...",
          "line": 1200
        }
      ],
      "evidence": "case CMD_DEMO_INTERFACE_GET_VERSION: return DemoInterfaceStubGetVersion(data, reply, option);"
    }
  ],
  "handlers": [
    {
      "cmd_id_macro": "CMD_DEMO_INTERFACE_GET_VERSION",
      "cmd_id_value": 0,
      "interface_function": "DemoInterfaceStubGetVersion",
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

### 10.4 恢复流程

**状态文件问题分类**

- 缺失：文件不存在 → 按恢复流程重建对应阶段。
- 不可解析：文件存在但 JSON 解析失败 → 报 `state_io_error`，停止，不自动清空。
- 身份不一致：JSON 可解析，但 service_name/project_dir 与当前调用不一致 → 报 `state_conflict`，停止，不混用旧状态。
- 内容不完整：JSON 可解析、身份一致，但缺少该阶段必需字段 → 视为该阶段未完成，按恢复流程重跑对应阶段。

“有效 entry_state”：JSON 可解析、service_name 匹配、entry 存在且 module_name 匹配。
“有效 trace_state”：JSON 可解析、service_name 匹配、status 为 in_progress/self_check/completed、queue/cases/handlers/default_cases/unresolved/dispatch_cache/visited 均为数组。

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

### 10.5 全新分析与旧输出

- progress.json 不存在 → 全新分析；开始时若同 service 的旧输出存在，先重命名为 `<service_name>_api.json.bak`，完成时再原子覆盖最终文件。
- progress.json 存在但 phase 未完成 → 严格按恢复流程继续，不从头重跑。
- 强制全新分析：删除 progress.json 与 tmp/。
- 完成态 state 保留，不删除。
- 同一 project_dir 同时只保留一个 service 的状态；检测到不同 service 的 progress.json → 报 `state_conflict` 并停止，提示用户归档或删除旧 `progress.json` 与 `tmp/` 后重跑；skill 不自动删除、不继续分析。

## 11. 错误处理

### 11.1 全局错误

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

### 11.2 分支级问题（不终止整体分析）

| 情况 | 处理 |
|------|------|
| cmdId 宏无法解析 | 值记 null，保留记录，写入 unresolved |
| 调用目标无法静态确定 | 写入 unresolved，写明已尝试的搜索方式 |
| 接口函数定义不在 scope 内 | 写入 unresolved，不进入最终 JSON |
| data 被写入全局/成员并在别处读取 | 写入 unresolved（data_deferred），不臆造 |
| 重载/同名定义歧义 | 记录全部候选，写 unresolved，不进入最终 JSON |
| 环 | 记 cycle，停止该分支，其它分支继续 |
| `default` 分支 | 只写 default_cases，不进入最终 JSON |

## 12. 自检与完成标准

自检开始前将 `trace_state.status` 置为 `self_check`；全部通过后置为 `completed`，并将 `progress.phase` 置为 `output`；最终 JSON 写成功后将 `progress.phase` 置为 `done`。

- 自检项 2 要求所有 `cases` 为 `analyzed`；自检项 4 要求没有任何 `default` handler，且不存在仅由 `default_cases` 支撑的 handler。
- 回读最终 JSON 必须读到 EOF；截断视为回读失败。
- 自检失败写 `code = "self_check_failed"`；cmdId 用例数取 `trace_state.cases.length`（不含 `default_cases`）。

输出前逐项自检，全部通过才写最终 JSON：

1. `queue` 为空，没有 `pending` / `in_progress` 工作项。
2. 所有 `cases` 已 `analyzed`，或明确记录无目标/unresolved。
3. 每个 `handlers` 条目都能在 scope_files 内定位到函数定义，且函数名为限定名。
4. `default_cases` 没有混入 `handlers`。
5. handlers 已按四元组去重，排序规则已执行。
6. `unresolved` 已记录，数量与报告一致。
7. 最终 JSON 恰好包含四个字段，无额外字段。
8. 写临时文件后 Read 回读，核对数组长度、字段名、类型。
9. `mv` 后双方法确认最终文件存在。
10. `progress.json` 置 `phase = done`，所有阶段状态为 `completed`。

完成报告包含：

- 分析范围文件数；
- service_name、入口变量、Bind 函数、Dispatch 函数数量；
- cmdId 用例数；
- 识别到的接口函数数量；
- unresolved 数量；
- 最终输出文件绝对路径。

## 13. 权限、约束与风格

### 13.1 权限申请

| 工具 | 用途 |
|------|------|
| `Bash` | find/grep/xargs/ls/mkdir/mv/rm，scope 内搜索与状态落盘 |
| `Read` | 读取源码、macro.json、状态文件 |
| `Write` | 写入 progress.json、entry_state.json、trace_state.json、最终 JSON |
| `mcp__codegraph__codegraph_explore` | （可选）.codegraph 存在时优先的代码探索工具 |
| `mcp__plugin_oh-my-claudecode_t__ast_grep_search` | （可选）AST 模式搜索 |

codegraph 未安装/未配置 → 用常规工具代替，不中断分析。

### 13.2 约束规则

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

### 13.3 章节结构

`SKILL.md` 按 api-finder 风格组织：

1. 核心原则
2. 使用方法与输出
3. 初始化与分析范围
4. 任务恢复机制
5. 阶段一：定位服务注册点
6. 阶段二：分析 Bind
7. 阶段三：cmdId 分发追踪
8. 输出与自检
9. 权限申请
10. 约束规则
11. 抗理性化检查
12. 使用示例

### 13.4 frontmatter

```yaml
---
name: hal-service-api-finder
description: 当用户调用 /hal-service-api-finder <project_dir> <service_name>，或要求在 OpenHarmony/HDF HAL 层 C/C++ 代码中识别指定 service 对外提供的全部接口函数时使用。按 HdfDriverEntry.moduleName 精确定位服务注册点，分析 Bind/Dispatch/cmdId 分发，输出 cmdId 到接口函数的 JSON 映射。
---
```

### 13.5 抗理性化检查

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

## 14. 静态走查验证

按用户选择，不运行真实代码，交付前做静态走查：

### 14.1 demo_service 样例走查

使用 rr1.txt 中的样例：

1. 搜索 `"demo_service"` → 命中 `g_DemoDriverEntry` 的 `.moduleName`。
2. 读取初始化 → `.Bind = HdfDemoDriverBind`，`.Init = HdfDemoDriverInit`，`.Release = HdfDemoDriverRelease`。
3. 搜索 `HDF_INIT(g_DemoDriverEntry)` → 命中。
4. 阅读 Bind：
   - Host 类型 `HdfDemoHost`；
   - `hdfDemoHost->ioService.Dispatch = DemoDriverDispatch`；
   - `hdfDemoHost->stub = GetOrNewObject(serviceImpl, IDemoInterface::GetDescriptor())`；
   - 实现类 `DemoService`；
   - `deviceObject->service = &hdfDemoHost->ioService`。
5. 追踪 `DemoDriverDispatch`：
   - 无 switch，直接 `hdfDemoHost->stub->SendRequest(cmdId, *dataParcel, *replyParcel, option)`；
   - 解析到 `DemoInterfaceStub::OnRemoteRequest`；
   - 外层仍为空，OnRemoteRequest 的 switch case 成为外层 cmd。
6. 遍历 switch：
   - 8 个具名 case；
   - 每个 case 调用的 `DemoInterfaceStub*` 函数记为接口函数；
   - default 分支进 default_cases。
7. 最终输出 8 条记录。示例只展示 GET_VERSION 的值 0；其余值按实际头文件解析。

### 14.2 失败路径走查

| 场景 | 预期 |
|------|------|
| service_name 拼写错误 | `service_not_found`，报告范围文件数和搜索模式，退出 |
| 同 moduleName 多个带注册引用 | `multiple_matches`，列出候选，退出 |
| 入口结构体无 `.Bind` | `bind_not_found`，退出 |
| Bind 函数体在 scope 外 | `bind_definition_not_found`，退出 |
| Bind 中无 Dispatch 且全 scope 找不到 | `dispatch_not_found`，退出 |
| scope 为空 | `empty_scope`，退出 |
| handler 定义在 scope 外 | 写 unresolved，不输出该接口，其它接口继续 |
| cmdId 宏无法解析 | 值 null，保留记录 |
| default 分支 | 只写 default_cases |
| 分析中断后重跑 | 按 phase 恢复，不重复已 analyzed 用例 |

### 14.3 恢复路径走查

| 中断位置 | 预期恢复行为 |
|----------|--------------|
| 阶段一中断 | 重跑阶段一，entry 已存在则跳过 |
| 阶段二中断 | 重跑阶段二（幂等） |
| 阶段三中断 | 加载 trace_state，重做 in_progress 工作项，跳过 analyzed 用例 |
| 输出阶段中断 | 直接重新组装输出，不重跑分析 |
| phase=done 且输出丢失 | 从 trace_state 补做输出 |

### 14.4 一致性检查

1. 最终 JSON 严格四字段，文件名为 `<service_name>_api.json`。
2. 所有 `file` 为绝对路径且在 scope_files 内。
3. 所有 `interface_function` 为限定名。
4. 全文没有全仓 grep 指令。
5. 全文没有 sub-agent 指令。
6. 状态字段名与恢复步骤一一对应。
7. 术语前后一致。
8. skill 目录只有 SKILL.md。

## 15. 使用示例

### 15.1 首次分析

```text
用户：/hal-service-api-finder /srv/ohos/drivers demo_service

Agent：
  [初始化] 分析范围：scope_files.txt 共 127 个文件；macro.json 存在，按需查阅。
  [阶段一] 定位 moduleName="demo_service" → g_DemoDriverEntry；
           HDF_INIT(g_DemoDriverEntry) 确认；
           提取 Bind=HdfDemoDriverBind。
  [阶段二] 分析 HdfDemoDriverBind：
           Host=HdfDemoHost，Dispatch=DemoDriverDispatch；
           实现类=DemoService，接口=IDemoInterface。
  [阶段三] 追踪 DemoDriverDispatch → SendRequest → DemoInterfaceStub::OnRemoteRequest；
           发现 8 个 cmdId 用例，8 个接口函数；
           default 分支仅记录。
  [自检] queue 空、8 条记录、无 unresolved。
  [输出] demo_service_api.json 已生成。

报告：
  范围 127 个文件 | cmdId 用例 8 个 | 接口函数 8 个 | unresolved 0 个
  输出：/srv/ohos/drivers/.ethunter_out/hal-service-api-finder/demo_service_api.json
```

### 15.2 断点续跑

```text
用户：/hal-service-api-finder /srv/ohos/drivers demo_service

Agent：
  [恢复检查] progress.json 存在，phase=trace_dispatch；
             trace_state 中用例 1-3 已 analyzed，用例 4 为 in_progress。
  [继续] 重做用例 4，继续用例 5-8。
  [自检] 通过。
  [输出] demo_service_api.json 已生成。
```

### 15.3 service 未找到

```text
用户：/hal-service-api-finder /srv/ohos/drivers demo_service_typo

Agent：
  [阶段一] 在 scope_files 的 127 个文件中精确搜索 "demo_service_typo"，无命中。
  [退出] service_not_found：
         未找到 moduleName="demo_service_typo" 的 HdfDriverEntry。
         请检查 service_name 拼写，或确认 clean_code.txt 是否包含入口文件。
```

## 16. 仓库与交付

- 仓库根目录：`/home/admin/cc/wksp/siakam_security_skills/hal_service_api_finder`。
- 远端：`https://github.com/siakam43/hal_service_api_finder.git`，公开仓库。
- 提交身份与 api-finder 仓库一致（`siakam43` 及对应邮箱）。
- 使用已登录的 `gh` CLI 创建远端仓库并推送。
- 初始提交内容：
  - `rr1.txt`；
  - `docs/superpowers/specs/2026-10-22-hal-service-api-finder-design.md`；
  - `.gitignore`（忽略 `.omc/`）。
- 实现完成后新增 `hal-service-api-finder/SKILL.md` 并提交。
- 不提交任何凭据、token、密码或本地环境文件。
