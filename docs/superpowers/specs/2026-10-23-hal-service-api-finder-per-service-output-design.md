# hal-service-api-finder 按服务隔离输出目录设计

- 日期：2026-10-23
- 状态：设计已确认，待实现
- 目标文件：`hal-service-api-finder/SKILL.md`

## 1. 背景与目标

当前输出根目录 `<out>` = `<project_dir>/.ethunter_out/hal-service-api-finder`，同一项目分析多个服务时会互相覆盖快照、scope 文件和最终 JSON。

目标：每个服务使用独立输出目录和独立备份目录，互不影响。

## 2. 新路径

```text
<out>     = <project_dir>/.ethunter_out/hal-service-api-finder/<service_name>
<out>_bak = <project_dir>/.ethunter_out/hal-service-api-finder/<service_name>_bak
最终输出  = <out>/<service_name>_api.json
```

- `mkdir -p <out>` 同时创建 `hal-service-api-finder/` 父目录。
- `scope_files.txt`、`scope_roots.txt`、`01_entry.json`、`02_bind.json`、`03_trace.json`、`error.json` 全部位于 `<out>`。
- 备份只覆盖本服务的 `<service_name>_bak`，其他服务目录不受影响。
- `service_name` 已限制为字母、数字、下划线、点和连字符，可直接作为目录名。

## 3. 修改范围

### 3.1 修改

- `hal-service-api-finder/SKILL.md`：
  - 第二章最终输出路径；
  - 第三章「输出目录初始化」的 `<out>` 定义；
  - 第三章「搜索方法与纪律」中的 `<out>` 定义；
  - 第十二章示例一的绝对路径与最终文件路径。

### 3.2 不修改

- `clean_code.txt`、`.etignore` 仍在 `<project_dir>/.ethunter_out/`。
- `macro.json` 仍在 `<project_dir>/.ethunter_out/macro.json`。
- codegraph 仍在 `<project_dir>/.codegraph`。
- 快照 schema、最终 JSON 四字段、备份/失败规则、分析行为。
- 其他使用 `<out>` 变量的章节（第四章、第七章、第八章）自动跟随新定义，无需逐条改写。

## 4. 具体文本

### 4.1 第二章

```text
最终输出：<project_dir>/.ethunter_out/hal-service-api-finder/<service_name>/<service_name>_api.json
```

### 4.2 第三章输出目录初始化

```text
- 输出根目录：`<out>` = `<project_dir>/.ethunter_out/hal-service-api-finder/<service_name>`。
```

### 4.3 第三章搜索方法与纪律

```text
所有阶段统一使用以下三种方法；后续只写方法名与参数，不再重复命令。其中 `<out>` = `<project_dir>/.ethunter_out/hal-service-api-finder/<service_name>`。
```

### 4.4 第十二章示例一

```text
1. 初始化：校验参数；若 `/srv/ohos/drivers/.ethunter_out/hal-service-api-finder/demo_service` 已存在，先删除旧 `<out>_bak`（若存在），再把 `<out>` 备份为 `<out>_bak`，新建空目录；确定分析范围并生成 `<out>/scope_files.txt`（存在 `clean_code.txt` 时另生成 `<out>/scope_roots.txt`）。
```

完成报告中的最终文件路径：

```text
/srv/ohos/drivers/.ethunter_out/hal-service-api-finder/demo_service/demo_service_api.json
```

## 5. 验证方式

1. 全文最终输出路径不再出现 `hal-service-api-finder/<service_name>_api.json`（缺少服务子目录）。
2. 第三章两处 `<out>` 定义都包含 `/<service_name>`。
3. 示例一的备份路径为 `demo_service` / `demo_service_bak`，最终文件在 `demo_service/` 下。
4. 12 个顶级章节、5 个 JSON 块保持不变且可解析；最终 JSON 四字段不变。
5. 记录 `wc -l`。

## 6. 预期效果

- 同一项目分析多个服务时，各自输出、快照、备份完全隔离。
- 服务级备份只保留该服务最近一次结果。
- 其他路径变量与分析行为不变。
