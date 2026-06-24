# fault-records.md - 运维故障记录表

用途：

- 查询节点维修记录。
- 查询节点故障状态。
- 查询历史处理情况。

链接：

```text
https://iqeubg8au73.feishu.cn/sheets/IC10sYPdjhvLnpt2GVGc1EZNngh
```
这是一个飞书表格，不要用 web_fetch 访问（飞书表格不是普通网页）。使用 lark-cli sheets 查询，加上 `--as user`，用 `--url` 参数传链接。

常用工作表：

```text
故障记录表（南洋维护）
```

查询关键字：

- 节点 IP。
- hostname。
- 故障时间。
- 维修状态。

返回时汇总：

- 故障时间。
- 故障现象。
- 当前状态。
- 处理结论。
- 是否仍需要进一步确认。

## 查询方法（按优先级）

### 1. 搜索指定状态或 IP

用 `+cells-search` 定位目标行，再按行号读取详细数据。

```bash
# 搜索状态关键字（如"处理中"、"正在检修"）
lark-cli sheets +cells-search \
  --url "https://iqeubg8au73.feishu.cn/sheets/IC10sYPdjhvLnpt2GVGc1EZNngh" \
  --sheet-name "故障记录表（南洋维护）" \
  --find 处理中 \
  --as user

# 搜索节点 IP
lark-cli sheets +cells-search \
  --url "https://iqeubg8au73.feishu.cn/sheets/IC10sYPdjhvLnpt2GVGc1EZNngh" \
  --sheet-name "故障记录表（南洋维护）" \
  --find 10.140.217.14 \
  --as user
```

得到匹配行号后，通过行号 range 读取完整行：

```bash
# 例如 C2 表示第 2 行的 C 列命中，读取第 2 行整行
lark-cli sheets +cells-get \
  --url "https://iqeubg8au73.feishu.cn/sheets/IC10sYPdjhvLnpt2GVGc1EZNngh" \
  --sheet-name "故障记录表（南洋维护）" \
  --range "2:2" \
  --as user --format json
```

### 2. 已知具体序号时按行读取

```bash
lark-cli sheets +cells-get \
  --url "..." --sheet-name "故障记录表（南洋维护）" \
  --range "31:31" --as user --format json
```

### 3. 全表读取（必须处理分页）

这种方式的 range 参数如 `A:Z` 只返回前 22 行（`has_more: true`），会遗漏后面的数据，不推荐。优先用搜索。

## 坑点备忘

| 错误做法 | 后果 | 正确做法 |
|---|---|---|
| 用 web_fetch 拉飞书表格链接 | 404 / 空内容 | 用 lark-cli sheets + `--as user` |
| 用 lark-cli 不加 `--as user` | bot 身份无权限（40403 / 91403） | 加 `--as user` |
| 用 `--sheet-url` 参数 | 非法参数 | 用 `--url` |
| 用 `+read` 命令 | 返回 403，已废弃 | 用 `+cells-get` |
| 读 `A:Z` 这种大 range 只取第一页 | 漏掉后面的行 | 用 `+cells-search` 先定位目标行再按行读取 |

## 表结构（列索引）

| 索引 | 列名 | 示例 |
|---|---|---|
| 0 | 序号 | 612 |
| 1 | 设备归属 | 910C |
| 2 | 状态 | 处理中 / 正在检修 / 已闭环 |
| 6 | 故障硬件设备类别 | 光模块 / NPU / 总线 |
| 8 | 故障描述 | P1-端口局部失效 |
| 9 | 涉及节点 | POD7-11 |
| 10 | 业务IP | 10.140.216.139 |
| 14 | 故障原因分析 | 端口降lane |
| 15 | 处理过程 | 2026/6/13 已报障400... |
| 16 | 最终解决步骤 | — |
