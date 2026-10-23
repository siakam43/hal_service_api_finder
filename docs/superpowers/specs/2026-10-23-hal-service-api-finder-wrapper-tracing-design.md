# hal-service-api-finder 封装函数下钻设计

- 日期：2026-10-23
- 状态：设计已确认，待实现
- 目标文件：`hal-service-api-finder/SKILL.md`

## 1. 背景与目标

当前策略：追踪到 `<target_path>` 内第一个接收 data 且不构成分发的业务函数即停止并记录为接口。

新需求：

- 如果这个函数只是明显的封装函数（只做校验/日志/别名，把 data 单线传给名称相近的下游函数），继续向下追踪；
- 停在第一个“真正处理 data”的函数，作为外部接口；
- 最终接口仍必须在 `<target_path>` 内；
- 如果始终找不到真正处理的函数，保留最深的、在 `<target_path>` 内的封装函数作为接口。

判定不追求严格：只对非常明显的单线透传继续，复杂或不确定的场景立即停止。

## 2. 范围

### 2.1 修改

- `hal-service-api-finder/SKILL.md`：
  - 第一章核心原则 3、第七章开头、第十章约束 4；
  - 3.6 分类规则、3.8 `data_chase` 字段、3.9 处理流程；
  - 第十章术语表、第十二章示例一。

### 2.2 不修改

- 最终 JSON 四字段格式、文件名、原子写入。
- 01_entry.json、02_bind.json、03_trace.json、error.json 的 schema。
- `<target_path>` 默认值、路径前缀匹配、scope 硬边界。
- cmdId 解析、dispatcher 追踪、`unresolved.kind` 取值集合（仍为 `target_path` 等）。
- 服务定位、Bind 分析、搜索方法。

## 3. 封装函数判定

只对同时满足以下前置条件的函数做封装判定：

- 定义在 `<target_path>` 下；
- 接收 data/data 派生参数且不构成分发；
- 不是纯日志、断言、框架转换、纯校验辅助等排除项。

判定为封装必须同时满足：

1. **自身不处理 data**：不读取/解析 data 字段，不基于 data 做业务分支或计算，不写入成员/全局，不执行业务动作。只允许参数校验、空指针/长度/魔数/范围检查、日志、局部别名和类型转换。
2. **函数体简单**：没有复杂控制流或难以判断的逻辑；一旦有疑问，按“真正处理”处理，立即停止。
3. **只有一个业务 data 下游调用**：实参含 data/派生值的调用中，排除纯校验/纯日志/框架转换调用后，恰好只剩一个。
   - 0 个：目标路径内函数直接记为接口（保守停止）。
   - ≥ 2 个：当前函数就是接口，停止。
4. **下游函数名相近**：去掉命名空间/类限定后，一个是另一个的前缀或后缀扩展，多出的部分不超过 8 个字符（如 `Foo` / `FooImpl`、`Parse` / `DoParse`）。函数名完全相同不算。
5. **下游是普通业务函数**：不是 dispatcher，定义可解析且在 scope 内。

任一条件不满足：

- 当前函数记为接口，停止；
- 下游是 dispatcher、名字不相近、定义不可解析或函数体复杂，都属于保守停止。

## 4. 下钻与 fallback

### 4.1 `fallback_handler`

`data_chase` 工作项新增：

```text
fallback_handler：{function, file, line} 或 null
```

含义：目前已知最深的、在 `<target_path>` 内的封装函数。

- 每判定一层封装，把 `fallback_handler` 更新为当前封装函数；
- 非封装路径保持原值；
- 下游在 `<target_path>` 外时，fallback 随工作项继续传递。

### 4.2 流程

1. 3.6 遇到 `<target_path>` 下的 data 业务函数：先做封装判定。
   - 封装 → 不记录，按 3.8 入队 `data_chase`，目标为唯一业务下游，`fallback_handler` = 当前函数；
   - 真正处理 → 记录当前函数。
2. 3.9 处理 `data_chase` 工作项：
   - 当前函数在 `<target_path>` 下：
     - 封装 → 入队下一个 `data_chase`，`fallback_handler` = 当前函数；
     - 真正处理 → 记录当前函数；
     - 0 个业务下游 → 记录当前函数（保守停止）。
   - 当前函数在 `<target_path>` 外：
     - 按现有规则沿全部业务 data 调用追踪，fallback 不变；
     - 没有任何业务 data 调用 → 若 `fallback_handler` 非 null，记录 fallback；否则 `unresolved`（`kind = "target_path"`）。
3. 找到真正处理函数（必须在 `<target_path>` 内）→ 记录该函数，不再使用 fallback。
4. 链条离开 `<target_path>` 后结束且未找到处理函数 → 记录最深 `fallback_handler`。
5. 没有任何 target_path 函数被访问、也没有 fallback → `unresolved`（`kind = "target_path"`）。
6. 封装函数只允许单线追踪；唯一业务下游是 dispatcher 时，按保守规则记录当前封装函数，不再进入 dispatcher 的多路径。
   - 注意：target_path 函数自身是 dispatcher 时，仍按现有规则继续 cmdId 分发。

### 4.3 多个业务下游

- 当前函数有 ≥2 个业务 data 下游调用 → 当前函数本身作为接口，记录并停止。
- 这适用于 3.6 的直接目标和 3.9 中 target_path 内的当前函数。

## 5. 记录与证据

- 记录真正处理函数时：`interface_function` / `file` / `definition_line` 取该函数；`cmd_id_macro` / `cmd_id_value` 取 `effective_outer`；写入顶层 `handlers` 并追加到对应 case。
- 记录 fallback 时：使用封装函数的 `function` / `file` / `line`；evidence 标注“最深封装函数（未找到真正处理函数）”。
- 两种情况的 evidence 都记录完整链：
  `起点 -> 封装1(data) -> 封装2(data) -> 真正处理(data)` 或 `... -> 封装N(data) [fallback]`。
- 去重、排序、最终 JSON 四字段不变。

## 6. 3.6、3.8、3.9 的文档改动

- 3.6：
  - 在“接收 data 且不构成分发”判定中，先做封装判定；
  - 封装 → 入队 `data_chase`（fallback = 当前函数）；否则记录当前函数；
  - 补充封装判定清单和名字相近规则。
- 3.8：
  - `data_chase` 字段新增 `fallback_handler`；
  - 环检测键包含 `fallback_handler`（按 function/file/line 或 null 区分），避免同一函数经不同封装链到达时 fallback 被错误合并。
- 3.9：
  - 对 target_path 下游先做封装判定；
  - 当前函数为 target_path 封装时继续单线追踪；
  - 当前函数为 target_path 真正处理时记录；
  - 无业务下游时按 4.2 处理 fallback / unresolved；
  - 步骤 5-6 的“内联处理 / 未到达 target_path”措辞按新 fallback 规则调整。
- 3.10 的终止条件、去重键、unresolved.kind 不需要新增取值。

## 7. 全局文档

- 第一章核心原则 3 补充：跳过明显封装，直到真正处理 data 的函数；找不到时保留最深 target_path 封装函数。
- 第七章开头补充同样的语义。
- 第十章约束 4 补充封装下钻与 fallback。
- 第十章术语表新增：

```text
| 封装函数 | target_path 内只做校验/日志/别名，并将 data 单线传给名称相近下游的函数 |
| fallback_handler | 已发现的最深 target_path 封装函数；未找到真正处理函数时作为外部接口 |
```

- 第十二章示例一阶段三补充：跳过封装函数，记录 `DemoVendorInterface(data)`；若只到封装层，则记录最深封装函数。

## 8. 验证方式

1. `grep -n -E '封装函数|fallback_handler' SKILL.md` 覆盖 3.6、3.8、3.9、术语表、约束和示例。
2. 3.8 `data_chase` 字段包含 `fallback_handler`；3.9 的流程包含封装下钻、真正处理记录、fallback 记录和 unresolved。
3. 12 个顶级章节、5 个 JSON 块保持不变且可解析；最终 JSON 四字段不变。
4. `<target_path>` 的默认值、前缀匹配、scope 硬边界未改变。
5. 记录 `wc -l`；预计从 970 行增加到约 1010-1040 行。

## 9. 预期效果

- 明显的单线封装不会进入最终 JSON；
- 接口落在 target_path 内真正处理 data 的第一层业务函数；
- 只有封装层时保留最深封装函数，不丢失接口；
- 复杂或不确定的函数保守地直接作为接口，避免错误下钻。
