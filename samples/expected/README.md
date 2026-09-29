# 期望结果与核对口径

这里的文件对应**一次运行**：

```
python -m logfold fold --out var samples/logs
python -m logfold restore --var var
python -m logfold report --var var
```

`samples/logs` 按文件名升序展开，顺序固定为：huge-line.log、orders.log、similar-ids.log、stacktrace.log、unique.log。模板号与文件顺序有关，换顺序结果会不同。

## 一、逐字节核对

| 期望结果 | 应与之逐字节相同的产物 |
| --- | --- |
| `templates.txt` | `var/templates.tsv` |
| `fold/orders.log.fold.txt` 等 5 个 | `var/fold/<name>.fold`（把 `.fold.txt` 去掉 `.txt`） |

`fold/huge-line.log.fold.txt` 的末行不以 `\n` 结尾，与 `samples/logs/huge-line.log` 的末行一致；这一点也要对上。

`stats.txt` 的每个数字都能在 `var/stats.json` 里找到对应字段：`file/records/bytes_in/bytes_fold` 来自 `files[]`，`templates_bytes` = `bytes_templates`，`templates` = `templates`，`bytes_in`、`bytes_out`、`ratio`、`ratio_limit`、`ratio_ok` 同名对应；其中 `file_ratio` = 该文件 `bytes_fold ÷ bytes_in`（4 位小数），`ratio` = `bytes_out ÷ bytes_in`（4 位小数）。`stats.txt` 末尾的柱状图数据 `模板号 次数 柱高` 与 `var/report.html` 里 `<rect>` 的 `x/y/height`、柱上文字一致（高 = ⌊次数×120÷最大次数+0.5⌋，x = 40+32i，y = 160−高）。

## 二、还原校验

对每个样例，`var/restore/<name>` 必须与 `samples/logs/<name>` 逐字节相同。用 `cmp` 或 `sha256sum` 核对，两侧应为下表的同一值：

| 文件 | sha256 |
| --- | --- |
| huge-line.log | `40bdf87d665079c3004d597eb15953e1d2edf50fcbde6fef519d3b28629e24ee` |
| orders.log | `b4aeafdfb4563356c8105a85b282769090d0d33d44721695b9b28883321ae4ca` |
| similar-ids.log | `a7c7a283b5d54358f8d1aa77b6f355e4a48201b58caeeb7d0a87e70789bafd68` |
| stacktrace.log | `c539255acbe84f771e46b339a6f4c26777f30541aaed3696c254fc5b3fb67890` |
| unique.log | `9f368211269f939a012ec70da4503fff1bcc706391fd55d51195d962a6997c64` |

## 三、每个样例的实测值与看点

| 文件 | 记录数 | 输入字节 | 折叠字节 | 单文件折叠比 | 看点 |
| --- | --- | --- | --- | --- | --- |
| huge-line.log | 6 | 23454 | 14496 | 0.6181 | 5 条 4 KB 级 payload 共用一个模板，第 1025 个 token 是尾部槽；末条 1.5 KB 单字段整体成槽，单独一个模板；末行无换行 |
| orders.log | 12 | 1049 | 653 | 0.6225 | 12 条归入同一模板，8 个槽；等级列宽不同，那个空白位在合并后也变成槽 |
| similar-ids.log | 15 | 897 | 612 | 0.6823 | 路径编号、长十六进制各自成槽；`req cache=miss hit=0` 与前一模板相似度 0.667 < 0.7，新建模板；`req\|sid=…` 与 `dequeue …` 两组同长度记录相似度只有 0.444 和 0.333，各自新建模板，组内第二条再以 0.889 并入 |
| stacktrace.log | 6 | 702 | 120 | 0.1709 | 一条记录 = `Traceback` 行加后续缩进行；模板里带 `\n` 字面量；异常行另成模板（两条同模板） |
| unique.log | 17 | 1006 | 544 | 0.5408 | 17 条记录（含一条空行）折成 11 个模板，模板数接近行数；空行模板的模板文本为空 |

全量：输入 27108 字节，输出 20462 字节（折叠文件 16425 + 模板清单 4037），`ratio` = 0.7548 ≤ 0.85，模板 24 个。样例不含触发模板上限（512）强制并入的记录。
