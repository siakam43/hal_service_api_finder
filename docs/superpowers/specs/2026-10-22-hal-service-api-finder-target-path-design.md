# hal-service-api-finder target_path 接口落地区域设计

- 日期：2026-10-22
- 状态：设计已确认，待实现
- 目标文件：`hal-service-api-finder/SKILL.md`

## 1. 背景与目标

阶段三目前把 cmdId 分支中“接收 data 且不构成分发”的直接业务函数记为接口。新需求：

- 如果直接目标不在 `<target_path>` 下，不把它记为接口；
- 沿该函数中“data 或 data 派生值作为实参”的调用继续追踪，直到到达定义在 `<target_path>` 下、且接收 data 的业务函数；
- 只有这样的函数才作为外部接口输出；
- 如果追踪不到，记 `unresolved`，不输出。

`<target_path>` 默认值为 `<project_dir>/vendor/hisi/`，定义为可人工修改的变量。

## 2. 范围

### 2.1 修改

- `hal-service-api-finder/SKILL.md`：
  - 第一章核心原则 3、第十章约束 4、第十一章抗理性化对应行；
  - 第三章「参数解析」：新增 `<target_path>`；
  - 第七章开头、3.1、3.2、3.5、3.6、3.8；
  - 新增 3.9「target_path 深入追踪」，原 3.9 顺延为 3.10；
  - 3.10 的去重、终止、unresolved.kind；
  - 第七章收尾的 `03_trace.json` 字段说明；
  - 第十章术语表、第十二章示例一。

### 2.2 不修改

- 最终 JSON 四字段格式、文件名、原子写入。
- 01_entry.json、02_bind.json、03_trace.json、error.json 的 schema。
- 服务定位、Bind 分析、cmdId 解析规则。
- clean_code.txt 硬边界、搜索方法、codegraph、macro.json。

## 3. `<target_path>` 变量

在第三章「参数解析」末尾新增：

```text
### 目标路径（非命令行参数）

`<target_path>` 默认为 `<project_dir>/vendor/hisi/`；归一化为绝对路径并补尾斜杠，相对路径按 `<project_dir>` 展开，可人工修改。函数定义文件的绝对路径以 `<target_path>` 开头（区分大小写）才算在目标路径下。`<target_path>` 不扩大 scope；目标路径不存在或不在 scope 内时不会有函数命中。
```

约定：

- `<target_path>` 不是命令行参数，不参与参数个数校验，可人工修改。
- `<project_dir>` 已是绝对路径，因此 `<target_path>` 默认也是绝对路径；人工改成相对路径时，按 `<project_dir>` 展开。
- 判断函数在目标路径下：函数定义文件的绝对路径以 `<target_path>` 开头（区分大小写，`<target_path>` 末尾有 `/`，保证目录边界）。
- `<target_path>` 只是过滤器，不扩大分析范围；目标文件仍需在 `scope_files.txt` 中才能读取。
- 目标路径不存在，或不在 scope 内时，不会有函数命中；如果人工把 `<target_path>` 改到 scope 外，目标定义无法读取，按范围外规则记 unresolved。

## 4. 3.6 分类规则

保留现有判定顺序（转发器 → 内层 dispatcher → 排除项 → data 业务函数 → 忽略），对第 4 类增加路径判断：

- 接收 data/派生参数且不构成分发的业务函数：
  - 定义文件在 `<target_path>` 下 → 记为接口函数（现有行为）；
  - 不在 `<target_path>` 下 → 不记为接口，按 3.8 入队 `kind = "data_chase"` 工作项，由 3.9 继续追踪。
- 3.6 的表格行同步改为「在 `<target_path>` 下记为接口；否则入队 `data_chase`」；分支级的多个 data 消费目标规则改为逐个目标分别判断。
- 接收 cmdId + data 且函数体内有分发 → 仍按内层 dispatcher 入队，不受 `<target_path>` 影响。
- 排除项（纯日志、断言、框架 sbuf/parcel 转换、纯校验辅助、无业务动作、只收 cmdId 不收 data、只收 reply）仍然排除，即使定义在 `<target_path>` 下也不记为接口。
- 一个分支只有内联 data 处理、没有调用目标时：
  - 分发函数定义在 `<target_path>` 下 → 记为接口；
  - 否则 → `unresolved`（`kind = "target_path"`）。
- 无 cmdId 分支、无条件直接处理 data 的 `*` 规则同样应用上述路径判断。

## 5. 工作项模型（3.1、3.2、3.8）

### 5.1 3.1 初始化

初始 Dispatch / OnRemoteRequest 工作项增加字段 `kind = "dispatch"`；其余字段不变。

### 5.2 3.2 处理循环

步骤 1 改为：

```text
1. 读取队列中的第一个工作项（不移出队列；处理完在第 7 步移除）。若 `kind = "data_chase"`，执行 3.9 后跳到第 7 步；否则继续第 2 步。
```

### 5.3 3.8 入队规则

3.8 明确支持两类工作项：

- `kind = "dispatch"`：现有转发器/内层 dispatcher，字段不变；
- `kind = "data_chase"`：由 3.6 或 3.9 在发现“不在目标路径下的 data 业务函数”时创建，字段：
  - `kind = "data_chase"`；
  - `id`：沿用递增计数器；
  - `function`：目标定义处限定名；
  - `file` / `line`：目标定义文件与签名行；
  - `data_vars`：目标函数中与 data 派生实参对应的形参名；
  - `effective_outer`：触发该追踪的最外层 cmd 表达式与值；
  - `origin_case_id`：触发追踪的原 case id；
  - `path`：父路径 + 目标限定名；
  - `evidence_chain`：从原目标到当前函数的调用链文本。

环检测与去重：

- `dispatch` 项沿用现有键。
- `data_chase` 键为 `(kind="data_chase", 定义文件+行号, effective_outer.expr, effective_outer.value, sorted(data_vars), origin_case_id)`。
- 队列命中静默去重；`visited` 命中按现有“严格前缀 → cycle，否则菱形去重”规则处理。
- 不设深度/扇出上限。

## 6. 新增 3.9「target_path 深入追踪」

处理 `kind = "data_chase"` 工作项：

1. 按 3.2 的同样方法读取函数完整函数体，确认完整。
2. 按 3.5 找出函数体内所有“实参含 data 或 data 派生值”的调用。
3. 对每个调用按 3.7 解析目标定义，然后分类：
   - 定义不在 scope、无法唯一确定或函数体不完整 → `unresolved`（`kind = "definition"`），停止该目标；
   - 接收 cmdId + data 且函数体内有分发逻辑 → 按普通 dispatcher 入队（`kind = "dispatch"`），由它的分支继续应用 target_path 规则；
   - 接收 data 的业务函数，定义在 `<target_path>` 下 → 记为接口（见第 7 节）；
   - 接收 data 的业务函数，定义不在 `<target_path>` 下 → 入队新的 `data_chase` 工作项，`data_vars` 取该目标中与 data 派生实参对应的形参；
   - 排除项 → 忽略，把调用原文与忽略原因追加到当前 case 的 `evidence`；
   - 不接收 data/派生参数 → 不追踪。
4. 如果一个函数有多个 data 实参调用，全部处理，不设扇出上限。
5. 如果函数体内没有任何 data 实参调用，且函数自身只是直接处理 data（内联处理）：
   - 该函数在 `data_chase` 中意味着它不在 `<target_path>` 下，记 `unresolved`（`kind = "target_path"`），detail 记录起点函数与追踪链。
6. 每条未到达 `<target_path>` 函数的追踪路径各记一条 `unresolved`（`kind = "target_path"`），detail 包含起点函数、`evidence_chain` 和已尝试的 data 实参调用；已到达的路径正常记接口，不因其他路径失败而丢弃。
7. 发现真环 → `unresolved`（`kind = "cycle"`）；重复/菱形路径静默去重。

## 7. 接口记录与归属

- `interface_function`、`file`、`definition_line` 取 `<target_path>` 下的最终函数；
- `cmd_id_macro` / `cmd_id_value` 取触发该追踪的 `effective_outer`；
- 直接由 `data_chase` 命中的接口，把最终函数的 `{name, file, line}` 追加到 `origin_case_id` 对应 case 的 `handlers`；`origin_case_id` 为 null（无条件 `*` 处理）时不追加 case handlers，只记录 handler 与 evidence；经 dispatcher 分支命中的接口，按该分支当前 case 的普通规则归属；
- 被追加的 case 的 `evidence` 记录追踪链，例如：
  `DemoDispatchTarget -> DemoVendorParse(data) -> DemoVendorInterface(data)`；
- 同一 cmdId 可产生多个 `<target_path>` 接口；同一函数经不同路径到达时按现有四元组/去重规则合并；
- 最终 JSON 仍只有四个字段，不增加路径字段。

## 8. unresolved 与边界

- `unresolved.kind` 新增 `target_path`。
- `target_path`：追踪链未到达目标路径、内联处理但不在目标路径、目标路径下无接收 data 的函数。
- `definition`：目标定义不在 scope、无法解析、函数体不完整。
- `cycle`：真环。
- `<target_path>` 不扩大 scope；scope 外的目标仍按 `definition` 处理。
- 单个目标失败不中止其他分支或工作项。

## 9. 3.10（原 3.9）调整

- 标题改为「3.10 终止条件、去重与收尾」。
- 终止条件增加：`data_chase` 工作项处理完，且其所有 data 实参调用已分类。
- 去重规则补充 `data_chase` 键（见 5.3）。
- `unresolved.kind` 列表增加 `target_path`。
- 3.1/3.2 中对原 3.9 的引用改为 3.10；收尾写 `03_trace.json` 的逻辑不变。

## 10. 其他章节

- 第七章开头改为：

```text
阶段三只做一件事：从阶段二的 Dispatch（或 stub 的 OnRemoteRequest）出发，沿全部分发路径追踪 cmdId 与 data 的流向。对不在 `<target_path>` 下的 data 业务函数，沿 data 实参继续追踪；到 `<target_path>` 内接收 data 且不构成分发的业务函数即停止，不展开其内部实现，也不做完整污点传播。最终把接口函数候选写入 `<out>/03_trace.json`。
```

- 第一章核心原则 3 改为：

```text
3. **只识别目标路径内的业务函数。** cmdId 用例分支中直接接收 data 且不构成分发的业务函数若位于 `<target_path>` 下，即为接口；否则沿 data 实参继续追踪到 `<target_path>` 下的业务函数。仍不展开 `<target_path>` 内业务函数的内部实现。
```

- 第七章 `03_trace.json` 的 `unresolved` 元素字段说明中，`kind` 取值增加 `target_path`。
- 第十章约束 4 改为：

```text
4. 只输出 `<target_path>` 下接收 data 且不构成分发的业务函数（含 3.6 的 `*` 无条件/内联处理记录）；目标路径外只沿 data 实参继续追踪，不展开目标路径内业务实现。
```

- 第十章术语表新增：

```text
| target_path | 外部接口落地区域，默认 `<project_dir>/vendor/hisi/`，可人工修改 |
```

- 第十章术语表「接口函数」行更新为：

```text
| 接口函数 | cmdId 用例分支中直接接收客户端 data（或 data 派生值）且定义在 `<target_path>` 下的业务函数 |
```

- 第七章 3.5 的局部追踪 bullet 改为：

```text
- 只做分类所需的局部追踪，不要求完整污点传播路径；C++ 间接调用、虚函数、函数指针、重载等解析可参考 `taint-path-walker` skill，本 skill 只追踪到 `<target_path>` 内的业务函数或明确终止条件。
```

- 第十一章抗理性化对应行改为：

```text
| “业务函数内部还有处理，继续展开” | `<target_path>` 外只沿 data 实参追踪到目标路径；`<target_path>` 内不展开业务实现。 |
```

- 第十二章示例一阶段三改为：以 `DemoDriverDispatch` 初始化内存队列，追踪 `SendRequest` 到 `DemoInterfaceStub::OnRemoteRequest`，枚举 8 个 cmdId 用例并解析宏值；对不在 `<target_path>` 下的直接目标继续沿 data 实参追踪，直到 `DemoVendorInterface(data)` 落在 `<target_path>` 下；队列为空后写 `<out>/03_trace.json`。

## 11. 验证方式

1. `grep -n 'target_path' SKILL.md` 覆盖：变量定义、3.6 分类、3.8 工作项、3.9 处理、3.10 去重、术语表。
2. 第七章存在 `### 3.9 target_path 深入追踪` 和 `### 3.10 终止条件、去重与收尾`；不再有指向旧 3.9 的引用。
3. `unresolved.kind` 两处列表均包含 `target_path`。
4. 12 个顶级章节、5 个 JSON 块保持不变且可解析；最终 JSON 四字段不变。
5. 记录 `wc -l`；预计从 917 行增加到约 980-1000 行。

## 12. 预期效果

- 只有 `<target_path>` 下接收 data 的业务函数进入最终 JSON。
- 目标路径外的直接处理函数会被继续下钻，不再作为接口输出。
- 追踪失败有 `target_path` unresolved 证据，便于人工调整 `<target_path>` 后复核。
