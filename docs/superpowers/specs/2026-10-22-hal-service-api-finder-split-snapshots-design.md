# hal-service-api-finder 中间结果章节拆分设计

- 日期：2026-10-22
- 状态：设计已确认，待实现
- 目标文件：`hal-service-api-finder/SKILL.md`

## 1. 背景与目标

第四章「中间结果记录」目前有 118 行，包含 `01_entry.json`、`02_bind.json`、`03_trace.json`、`error.json` 四套完整定义。其中 `02_bind.json` 在第六章已有一份示例，`03_trace.json` 在第七章已有一份示例，存在重复。

目标：

- 第四章只列出中间文件清单（不展开格式），只详细说明所有阶段共用的 `error.json`；
- `01_entry.json`、`02_bind.json`、`03_trace.json` 的格式说明放到各自阶段章节；
- 删除第二章重复的中间结果清单；
- 删除第四章开头“不做任务恢复”的说明。

## 2. 范围

### 2.1 修改

- `hal-service-api-finder/SKILL.md`：第二章、第四章、第五章「阶段产物」、第六章「阶段产物」、第七章「收尾」。

### 2.2 不修改

- 分析行为、搜索方法、cmdId 追踪规则。
- `error.json` 的 schema 和写入规则（只移除“不参与恢复”一句）。
- 最终 JSON 四字段格式与原子写入方式。
- 错误码表、权限表、示例章节。
- 历史计划/规格与 `rr1.txt`。

## 3. 第二章

删除中间结果清单：

```text
中间结果（均位于输出根目录 `<out>`，定义见第三章）：
- scope_files.txt
- scope_roots.txt（仅 clean_code.txt 分支存在）
- 01_entry.json
- 02_bind.json
- 03_trace.json
- error.json（仅失败时）
```

保留最终输出路径、四字段定义和“不生成 summary 报告”。

## 4. 第四章

整章改为：

```markdown
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
```

删除原 4.1「阶段快照」标题、01/02/03 示例和元素字段说明，以及 `error.json` 中的“不参与恢复”措辞。

## 5. 第五章「阶段产物」

在现有 `service_name` / `scope` / `entry` 字段说明之后、写盘步骤之前，加入从第四章移来的 `01_entry.json` 示例，内容不变：

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

## 6. 第六章「阶段产物」

第六章已有 `02_bind.json` 示例，保留该示例，删除第四章的展开版示例。

把字段 bullet 中的：

```text
`bind` 对象字段形状见第四章 4.1 的 `02_bind.json`
```

改为：

```text
`bind` 对象字段形状见上方示例
```

## 7. 第七章「收尾」

保留第七章现有的 `03_trace.json` 示例。在其后加入从第四章移来的四条元素字段说明：

```text
- `cases` 元素字段：`id`、`dispatcher`、`dispatcher_file`、`dispatcher_line`、`dispatch_site`、`level`、`case_expr`、`case_value`、`outer_cmd_expr`、`outer_cmd_value`、`parent_case_id`、`handlers`、`evidence`；其中 `handlers` 是 `{name, file, line}` 引用列表。
- 顶层 `handlers` 元素字段：`cmd_id_macro`、`cmd_id_value`、`interface_function`、`file`、`definition_line`；`definition_line` 仅用于定位与去重，最终 JSON 不包含。
- `default_cases` 元素字段：`dispatcher`、`dispatcher_file`、`dispatcher_line`、`dispatch_site`、`evidence`。
- `unresolved` 元素字段：`kind`、`detail`、`tried`；`kind` 取 `cmd_value` / `target` / `definition` / `data_deferred` / `cycle`。
```

删除第四章原有的“队列、`visited`、`dispatch_cache` 只存在于内存”、“不写 `status`、`phase`、`counters`、`queue`、`visited`、`dispatch_cache`”和“以上元素都不写 status 等状态字段”三条说明（阶段三正文已覆盖）。

## 8. 验证方式

1. 第二章不再有中间结果清单；清单只在第四章出现一次。
2. 第四章不再包含 `01_entry.json` / `02_bind.json` / `03_trace.json` 的 JSON 示例或元素字段列表。
3. 第五章包含 `01_entry.json` 示例；第六章保留 `02_bind.json` 示例且不再引用第四章；第七章保留 `03_trace.json` 示例并包含四条元素字段说明。
4. 12 个顶级章节保持不变；JSON 代码块从 7 个降为 5 个（error/01/02/03/最终 JSON），全部可解析。
5. 记录 `wc -l`；预计从 981 行降到约 910 行。

## 9. 预期效果

- 第四章从 118 行降到约 23 行。
- 每类中间文件的格式只在一个地方定义。
- 公共失败产物 `error.json` 仍集中说明。
