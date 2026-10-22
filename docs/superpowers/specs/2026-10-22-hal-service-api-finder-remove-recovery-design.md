# hal-service-api-finder 去除任务恢复机制设计

- 日期：2026-10-22
- 状态：设计已确认，待实现
- 目标文件：`hal-service-api-finder/SKILL.md`

## 1. 背景与目标

当前 skill 的任务恢复机制占据第四章约 232 行，并渗透到第三、五、六、七、八、十、十二章：

- `progress.json`、`entry_state.json`、`trace_state.json` 三套状态 schema；
- phase/status 转换、恢复流程、状态文件问题分类；
- counters、每工作项落盘、断点续跑、状态冲突、状态文件错误码。

这些机制让 skill 过长、流程过重。目标：

- 去掉任务恢复机制，不做断点续跑；
- 每次执行先备份旧输出目录，再新建输出目录，从零重新分析；
- 每个阶段结束只写一个简单快照，供调试查看；
- 只保留每个阶段的分析结果和必要细节。

## 2. 范围

### 2.1 修改

- `hal-service-api-finder/SKILL.md`：重写第四章，简化第三、五、六、七、八、十、十一、十二章中的状态与恢复引用。
- 旧实现计划与旧设计规格顶部各加一句历史说明，不改正文。

### 2.2 不修改

- 分析行为：服务定位、Bind 分析、cmdId 追踪的判定规则不变。
- clean_code.txt 硬边界、macro.json、codegraph、搜索方法。
- 环检测、内存中的 visited / dispatch_cache、case/handler 去重。
- 最终 JSON 四字段格式和文件名。
- `rr1.txt` 或其他原始需求文件。

## 3. 输出目录与启动清理

`<out>` = `<project_dir>/.ethunter_out/hal-service-api-finder`
`<out>_bak` = `<project_dir>/.ethunter_out/hal-service-api-finder_bak`

启动时依次执行：

```bash
if [ -d "<out>" ]; then
  if [ -d "<out>_bak" ]; then rm -rf "<out>_bak" || { echo 'output_dir_failed' >&2; exit 1; }; fi
  mv "<out>" "<out>_bak" || { echo 'output_dir_failed' >&2; exit 1; }
fi
mkdir -p "<out>" || { echo 'output_dir_failed' >&2; exit 1; }
```

规则：

- 只有 `<out>` 目录存在时才处理备份；`<out>` 不存在时保留现有 `<out>_bak`。整段在同一个 shell 中一次执行，失败时向 stderr 输出 `output_dir_failed`。
- `<out>` / `<out>_bak` 按目录处理；若同名路径是普通文件，`mv`/`mkdir` 会失败并报 `output_dir_failed`。
- 已有 `<out>` 时，若 `<out>_bak` 存在则先删除，再把 `<out>` 整体 `mv` 为 `<out>_bak`，只保留最近一次备份。旧最终 JSON、阶段快照、scope 文件、error.json 一并保留在备份中。
- 新目录创建后重新生成 `<out>/scope_files.txt`；存在 `clean_code.txt` 时同时生成 `<out>/scope_roots.txt`。
- 不读取旧结果，不判断 project/service 匹配，不续跑。
- 所有路径变量使用双引号；`<project_dir>` 来自用户参数，替换时按 shell 规则转义。
- `rm`/`mv`/`mkdir` 任一步失败 → 报 `output_dir_failed`，停止；若新 `<out>` 未成功创建，只在对话中报告，不写 error.json。

这里的 `[ -d ]` 仅用于 shell 控制流；最终文件/目录存在性仍按第三章双方法确认。

新输出目录结构：

```text
<out>/
  scope_files.txt
  scope_roots.txt（仅 clean_code.txt 分支存在）
  01_entry.json
  02_bind.json
  03_trace.json
  <service_name>_api.json
  error.json          # 仅失败时
```

不再有 `tmp/` 子目录。

## 4. 阶段快照

中间快照直接写入 `<out>`，不要求原子写入；写入失败 → 报 `snapshot_write_failed`，停止。最终 `<service_name>_api.json` 仍采用临时文件 + `mv`。

### 4.1 01_entry.json

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

### 4.2 02_bind.json

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

### 4.3 03_trace.json

```json
{
  "service_name": "demo_service",
  "entry_dispatchers": [],
  "cases": [],
  "handlers": [],
  "default_cases": [],
  "unresolved": []
}
```

- 队列、`visited`、`dispatch_cache` 只存在于内存，阶段三结束后不写入文件。
- 不写 `status`、`phase`、`counters`、`queue`、`visited`、`dispatch_cache`。
- `cases`、`handlers`、`default_cases`、`unresolved` 的元素结构沿用现有定义，仅去掉状态/恢复字段。

### 4.4 error.json

仅在失败时写入：

```json
{
  "phase": "init|locate_entry|analyze_bind|trace_dispatch|output",
  "code": "<错误码>",
  "message": "<说明>",
  "evidence": "<证据或已尝试的搜索方式>"
}
```

`phase` 取值固定为 `init` / `locate_entry` / `analyze_bind` / `trace_dispatch` / `output` 之一，依次表示初始化、阶段一、阶段二、阶段三或最终输出阶段失败；只作失败位置标签，不参与恢复。写入 error.json 后停止；不写最终 JSON。

## 5. 各阶段行为

### 5.1 初始化

1. 校验参数。
2. 按第 3 节备份旧 `<out>`、新建 `<out>`。
3. 确定分析范围，生成 `scope_files.txt` / `scope_roots.txt`。
4. 进入阶段一。

不创建 progress.json，不检查旧状态。

### 5.2 阶段一：定位服务注册点

- 逻辑不变（精确查找 moduleName、注册形式、多命中、Bind 缺失检查）。
- 成功 → 写 `01_entry.json`，进入阶段二。
- 失败 → 写 `error.json`，停止。

### 5.3 阶段二：分析 Bind

- 逻辑不变（Bind 定义、Host、Dispatch、stub/接口/实现类、服务注册证据）。
- 成功 → 写 `02_bind.json`，进入阶段三。
- 失败 → 写 `error.json`，停止。

### 5.4 阶段三：cmdId 分发追踪

- 工作队列、`visited`、`dispatch_cache`、`cases`、`handlers`、`default_cases`、`unresolved` 只在内存中维护。
- 保留环检测、分发点缓存、case/handler 去重；这些是单次运行的正确性机制，不是恢复机制。
- 队列为空后 → 写 `03_trace.json`，进入输出。
- 全局失败指工具/文件 I/O 或无法继续读取源码等不可恢复错误；单个分支的解析失败只记 `unresolved`，不中止。
- 全局失败 → 若已收集到部分追踪结果，先写 `03_trace.json` 的部分快照，再写 `error.json`（`code = "internal_error"`）；否则只写 `error.json`（`code = "internal_error"`）。

### 5.5 输出

- 自检以 `03_trace.json` 为准；不再有 trace_state.status 或 phase 转换。
- 组装最终四字段 JSON，写临时文件，Read 回读，`mv` 为 `<service_name>_api.json`。
- 自检失败 → `error.json`（phase=output，code=self_check_failed），停止，不写最终 JSON。
- 最终写入失败 → `error.json`（phase=output，code=output_write_failed），停止。
- 完成后直接报告，不设置 phase=done。

## 6. 删除的机制

- 删除 `progress.json`、`entry_state.json`、`trace_state.json` 及全部 schema。
- 删除恢复流程、状态文件问题分类、phase/status 转换、counters、per-work-item 落盘、断点续跑、续跑去重。
- 删除 `state_conflict`、`state_io_error` 错误码。
- 删除最终 JSON 的 `.bak` 重命名逻辑（由整体输出目录备份替代）。
- 删除 `scope_roots.txt` 的陈旧状态清理逻辑（每次新建目录，不存在陈旧文件）。
- 删除“匹配 progress.json 时先恢复、不重建 scope”的规则；scope 每次重新生成。

## 7. 错误处理

- 失败统一写 `error.json`（若 `<out>` 已存在）并在对话中报告，然后停止。
- 保留的错误码：
  - `usage_error`、`invalid_project_dir`、`output_dir_failed`、`empty_scope`；
  - `service_not_found`、`multiple_matches`、`bind_not_found`、`bind_definition_not_found`、`bind_ambiguous`、`dispatch_not_found`；
  - `snapshot_write_failed`、`internal_error`、`self_check_failed`、`output_write_failed`。
- 不再有 `progress.error`。

## 8. 保留的不变量

- clean_code.txt 是硬边界；搜索方法与纪律不变。
- macro.json、codegraph、搜索方法、候选判定、完整函数体检查不变。
- 环检测、单次运行内的 visited/dispatch_cache、case/handler 去重保留。
- 最终 JSON 仍是四字段扁平数组，文件名仍为 `<service_name>_api.json`。
- 最终 JSON 仍采用原子写入。
- 12 个顶级章节保持不变。

## 9. 验证方式

1. 静态检查 `SKILL.md` 中不再出现指向旧机制的引用（`progress.json`、`entry_state.json`、`trace_state.json`、`恢复流程`、`断点续跑`、`state_conflict`、`state_io_error` 等）；允许“不做任务恢复”“不写 counters”这类明确的否定表述。
2. 用临时目录实测备份逻辑：旧 `<out>` → `<out>_bak`，新 `<out>` 为空；已有 `_bak` 时先删除再备份。
3. `01_entry.json`、`02_bind.json`、`03_trace.json`、`error.json` 示例均可解析为 JSON。
4. 12 个顶级章节保持不变；最终 JSON 四字段不变。
5. 记录重构前后 `wc -l`；恢复机制已移除，但新增快照 schema 与字段说明使实际行数为 1020 行，未达到早期约 850-900 行的估计。
6. 旧计划/旧规格顶部有历史说明；除此之外没有其他文件改动。

## 10. 预期效果

- 第四章从 232 行降到 157 行（含 JSON 示例）。
- 第五、六、七、八章删除大量状态更新和续跑描述。
- 每次执行行为固定为：备份旧输出 → 新建目录 → 从零分析 → 阶段快照 → 最终 JSON。
- 调试时只需查看 `01_entry.json`、`02_bind.json`、`03_trace.json`、`error.json`。
