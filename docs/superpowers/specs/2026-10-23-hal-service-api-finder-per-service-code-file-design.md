# hal-service-api-finder 按服务命名分析范围文件设计

- 日期：2026-10-23
- 状态：设计已确认，待实现
- 目标文件：`hal-service-api-finder/SKILL.md`

## 1. 背景与目标

同一项目分析多个服务时，`clean_code.txt` 是共享输入，容易让不同服务误用同一份分析范围。

目标：将分析范围输入文件改为按服务命名：

```text
<project_dir>/.ethunter_out/<service_name>_code.txt
```

不回退读取旧的 `clean_code.txt`。

## 2. 文件位置与判定顺序

1. 先双方法检查 `<project_dir>/.ethunter_out/<service_name>_code.txt`；存在则按它建立 scope，作为硬边界。
2. 不存在时检查 `<project_dir>/.ethunter_out/.etignore`；存在则全量 find C/C++ 文件并按 `.etignore` 规则排除。
3. `.etignore` 也不存在时全量 find。
4. 不回退读取 `clean_code.txt`。

- 文件位于 `.ethunter_out/` 下，是输入文件，不进入 `<out>`，不随输出目录备份/重建。
- `scope_roots.txt` 仅在该文件存在时生成。
- `.etignore`、`macro.json`、codegraph、输出目录规则不变。

## 3. 全文替换

以下位置的 `clean_code.txt` 全部改为 `<service_name>_code.txt`：

- 第一章核心原则 2；
- 第三章「分析范围确定」步骤与说明；
- 第三章标题「clean_code.txt 优先，且是硬边界」「clean_code.txt 不存在」；
- 第三章 clean_code 分支的检查、硬边界规则、`scope_roots` 说明；
- 第四章中间文件清单中的 `scope_roots.txt` 条件；
- 第五章 `service_not_found` 的提示与 `01_entry.json` 的 `scope` 字段说明；
- 第九章权限表 `Read` 行；
- 第十章约束 3；
- 第十章术语表 `scope_roots`；
- 第十二章示例一的 scope 生成说明、示例三的覆盖范围提示。

标题使用反引号：

```text
### `<service_name>_code.txt` 优先，且是硬边界
### `<service_name>_code.txt` 不存在
```

`01_entry.json` 的 `clean_code` 布尔字段保留，含义改为“本次是否使用 `<service_name>_code.txt`”。

## 4. 示例调整

- 示例一：`存在 demo_service_code.txt 时另生成 <out>/scope_roots.txt`。
- 示例三：`确认 demo_service_code.txt 覆盖了该驱动的定义文件`。

## 5. 不修改

- 最终输出路径、快照 schema、错误码、搜索方法。
- `.etignore` 的共享逻辑。
- `clean_code` JSON 字段名（只改描述）。
- 分析行为、target_path、封装判定。

## 6. 验证方式

1. `grep -n 'clean_code.txt' SKILL.md` 无输出。
2. `grep -n '<service_name>_code.txt' SKILL.md` 覆盖原则、范围判定、标题、权限、约束、术语、示例。
3. `clean_code` 布尔字段仍存在于 `01_entry.json` 示例与说明中。
4. 12 个顶级章节、5 个 JSON 块保持不变且可解析。
5. 记录 `wc -l`。

## 7. 预期效果

- 每个服务使用自己的分析范围文件，多服务分析不再共享同一个 `clean_code.txt`。
- 缺少按服务文件时仍按 `.etignore` 或全量 find 继续，不中断。
- 输出、快照、分析规则不变。
