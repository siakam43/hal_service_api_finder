---
name: hal-service-api-finder
description: 当用户调用 /hal-service-api-finder <project_dir> <service_name>，或要求在 OpenHarmony/HDF HAL 层 C/C++ 代码中识别指定 service 对外提供的全部接口函数时使用。按 HdfDriverEntry.moduleName 精确定位服务注册点，分析 Bind/Dispatch/cmdId 分发，输出 cmdId 到接口函数的 JSON 映射。
---

# hal-service-api-finder

根据 OpenHarmony/HDF HAL 层 service 的注册流程，识别指定 service 对外提供的全部接口函数，输出 cmdId 到接口函数的 JSON 映射，用于安全攻击面分析。

## 一、核心原则

1. **分析质量优先于分析效率。** 宁可慢，不可草率。每个分支都要追到接口函数或明确的终止条件，不得因任务长而跳读、合并或提前退出。不设深度上限、不设扇出上限，所有静态可确定的目标全部追踪。
2. **分析范围是硬边界。** 存在 `clean_code.txt` 时，只分析列表（含文件夹展开）内的文件；所有搜索都限制在 scope 内，禁止 `grep -rn <project_dir>`。
3. **只识别直接目标。** cmdId 用例分支中直接接收 data 的业务函数就是接口函数；不展开业务函数的内部实现。
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

中间结果：
- progress.json
- tmp/scope_files.txt
- tmp/scope_roots.txt（仅 clean_code.txt 分支存在）
- tmp/entry_state.json
- tmp/trace_state.json

不生成 summary 报告。

### 完成报告

完成时报告：分析范围文件数、service_name、入口变量、Bind 函数、Dispatch 函数数量、cmdId 用例数、接口函数数量、unresolved 数量、最终输出文件绝对路径。

## 三、初始化与分析范围

### 参数解析

1. 校验两个参数均非空：`<project_dir>` 与 `<service_name>` 缺失任意一个、为空或仅含空白字符、或参数多于两个时，打印用法并结束。
2. 将 `<project_dir>` 解析为绝对路径：相对路径按当前工作目录展开并规范化，后续一律使用绝对路径。
3. `<project_dir>` 不存在或不是目录 → 报错 `invalid_project_dir` 并结束，不创建任何输出。
4. `<service_name>` 仅去除首尾空白，保持大小写与其余字符不变，用于后续精确匹配。
- `service_name` 不得包含 `/`、`\`、换行、NUL 或引号；包含这些字符 → 报错 `usage_error` 并结束。
- 搜索时按字面量处理并正确加引号（优先 `grep -F`）。

### 输出目录初始化

- 输出根目录：`<out>` = `<project_dir>/.ethunter_out/hal-service-api-finder`。
- 执行 `mkdir -p <project_dir>/.ethunter_out/hal-service-api-finder/tmp`。
- 创建失败（权限不足、只读文件系统、同名文件占用等）→ 报错 `output_dir_failed` 并结束。
- 后续所有中间文件与最终 `<service_name>_api.json` 均写入 `<out>`。

### 分析范围确定

按以下优先级确定分析范围，命中即停止；范围一经生成即为本次分析的硬边界：

1. 先用双方法检查 `<project_dir>/.ethunter_out/clean_code.txt`：存在则按“clean_code.txt 优先，且是硬边界”建立 scope。
2. `clean_code.txt` 不存在时，检查 `<project_dir>/.ethunter_out/.etignore`：存在则全量 find C/C++ 文件并按 `.etignore` 规则排除。
3. `.etignore` 也不存在时，直接全量 find C/C++ 文件。
4. 无论走哪条分支，都必须生成 `tmp/scope_files.txt`；存在 `clean_code.txt` 时还必须生成 `tmp/scope_roots.txt`，否则 `scope_roots` 置为 null。
- 若存在与当前 `project_dir`/`service_name` 匹配的 `progress.json`，先执行第四章的恢复检查；只有全新分析才重新生成 scope 和 scope 文件。

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
- codegraph 返回的每个文件都必须按 `scope_files.txt` 过滤；范围外的结果直接丢弃。

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

## 四、任务恢复机制

## 五、阶段一：定位服务注册点

## 六、阶段二：分析 Bind

## 七、阶段三：cmdId 分发追踪

## 八、输出与自检

## 九、权限申请

## 十、约束规则

## 十一、抗理性化检查

## 十二、使用示例
