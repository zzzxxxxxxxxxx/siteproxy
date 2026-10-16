# JP 来源请求明细导出独立校验报告（task-5）

- 校验对象：`jp-requests.ndjson.gz`（72,749,703 字节，2,541,447 行，多成员 gzip），分片 `.cf-export-tmp/jp-parts/*.ndjson.gz`（31 个），统计 `.cf-export-tmp/jp-stats.json`
- 数据集：ZoneHttpRequestsAdaptiveGroups，zone `f7e915e518036f92b5fdadf785cc99b1`，filter `clientCountryName="JP"`，`datetimeMinute_geq/_leq` 分钟桶语义
- 窗口：2026-09-16T16:03:00Z ~ 2026-10-16T16:03:00Z（window.json，pinnedAt 2026-10-16T16:03:49Z，lookbackDays=30）；实际有数据的分钟 2026-09-16T16:03:00Z ~ 2026-10-16T15:44:00Z
- 校验方：verifier，独立编写脚本，未复用 jp-export 的代码路径（该脚本仅作只读阅读以确认过滤语义）
- 复现脚本与证据：`.cf-export-tmp/verify/jp-check.mjs`、`jp-parts-check.mjs`、`jp-api-check.mjs`、`jp-count-dist.mjs`，输出 `jp-check.json`、`jp-parts-check.json`、`jp-api-check.json`、`jp-count-dist.json`

## 结论

**通过。** 行数、字段、总量、逐日/逐小时对账、分片覆盖与首尾边界全部与独立查询精确一致（0 差异），merge 为无损拼接。有两处任务描述中的前提与实际数据不符（见“发现”），但不影响导出正确性。

## 一、发现（按严重度）

**无阻断性问题。** 以下为需要知悉的两点：

1. **该数据集确实在采样，任务描述中的前提不成立。** `sampleInterval` 并非恒为 1：分布为 1~330，均值 1.39；`count` 也非“多为 1”而是 71.25% 的行为 1，均值 1.77，271 个不同取值，最大 850。这不影响正确性——`count` 已是按 sampleInterval 放大后的计数，且我的独立复算与 NDJSON 完全相等。但据此推断“对账应能精确或接近精确”的理由应当修正为：数据是预聚合采样组，同窗口重查返回同一组，故可精确对账；跨查询窗口宽度仍可能出现估计漂移（见未验证项）。
2. **行字段名与任务给出的清单不一致，且清单漏了 userAgent。** 实际 22 个字段为：
   `minute, clientIP, userAgent, host, clientRequestPath, method, protocol, edgeStatus, originStatus, contentType, colo, clientDeviceType, clientSSLProtocol, sampleInterval, securityAction, originIP, requestSource, count, visits, edgeRequestBytes, edgeResponseBytes, originResponseDurationMs`。
   任务清单里的 `path`/`device`/`tls` 对应实际的 `clientRequestPath`/`clientDeviceType`/`clientSSLProtocol`，并额外存在 `userAgent`。这不影响数据本身，但下游按清单取字段会取空。我已按实际字段名校验：2,541,447 行全部恰好含这 22 个字段，缺失 0、多余 0。

## 二、校验项与结果

| 校验项 | 方法 | 结果 |
| --- | --- | --- |
| gzip 完整性 | `gzip -t` 合并文件 + 31 个分片 | 全部通过（多成员 gzip 正常） |
| 行数 | 独立流式解压计数 | 2,541,447，与 jp-stats.totalLines 一致 |
| 逐行 JSON 解析 | 逐行 JSON.parse | 0 解析失败 |
| 字段齐全 | 22 字段逐一检查 | 0 缺失、0 多余、0 类型异常 |
| 排序与重复 | 分钟单调性 + 每分钟 16 维键去重 | 0 逆序、0 重复维度元组、单分钟最多 1247 行 |
| 窗口外行 | minute 是否在声明窗口内 | 0 行越界 |
| 总量 | 独立复算 sum(count) | 4,498,666 = jp-stats.totalCount |
| 其他统计 | edgeResponseBytes / uniqueClientIPs / uniqueHosts / byRequestSource / byDay | 与 jp-stats 全部逐项一致 |
| 分片覆盖 | 31 个分片逐日边界与合计 | 09-16~10-16 无缺日；首片 16:03 起、末片 15:44 止，均在界内；每片 rows/count 与 jp-stats.byDay 完全一致 |
| merge 无损 | sha256(解压合并文件) vs sha256(按日期序拼接解压分片) | 两边同为 `3d0b988e…42af3`，**完全一致** |
| 随机小时抽样 | 独立查询 5 个小时：09-16T16:03-16:59、09-30T01、10-09T22、10-11T02、10-16T15:00-15:59 | 5/5 小时总量完全相等；逐分钟比对 0 处不同、0 个“仅 API 有”的分钟 |
| 独立总量 A | 按天 31 次查询，dims=`clientIP`，分钟桶过滤 | 合计 4,498,666，且逐日与 NDJSON 逐日 0 处不一致 |
| 独立总量 B | 3×10 天块查询，dims=`clientIP` | 合计 4,498,666，与 A 及 NDJSON 一致 |
| 窗口尾部 | 独立查询 2026-10-16T15:45~16:03 | 0 行、0 count，确认 15:44 之后无数据（非摄入延迟导致漏导） |

抽样小时明细（API vs NDJSON）：2189/2189、4055/4055、2308/2308、7534/7534、9/9，逐分钟差异均为 0。

## 三、分片与首尾边界

- 分片数 31，对应 2026-09-16 ~ 2026-10-16 每日一片，区间内无缺日。
- 首片 2026-09-16：min 16:03:00Z（= 窗口起点），max 23:57:00Z，rows=4,102，count=7,799。
- 末片 2026-10-16：min 00:09:00Z，max 15:44:00Z，rows=106,410，count=173,307。
- `window.json` 的 end 为 16:03:00Z，而 `jp-stats.window.to` 为 15:44:00Z：前者是固定窗口上界，后者是实际最后有数据的分钟；我已复核 15:45~16:03 在导出后重查仍为 0 行，说明该 19 分钟区间确实无数据。
- 窗口起点之前的更早数据按 30 天 lookback 设计未导出，属范围取舍，非缺口。

## 四、未验证项与残余风险

1. jp-export 自审的“721 个小时逐分钟 missing=0/extra=0”结论我没有逐小时完整复现。我独立做的是：5 个抽样小时逐分钟精确比对 + 全窗口按天（31 次）与 3×10 天分块两次独立总量复算，均 0 差异。这强烈支持其结论，但不等于逐小时全量复现。
2. jp-export 提到的“宽窗口偶发采样放大/缩小”未在我抽到的样本中出现（5/5 精确一致），我未刻意复现到该噪声场景；因此“跨不同窗口宽度重查是否可能漂移”仍是未验证项。
3. `visits` 指标没有独立来源可对（jp-stats 未收录其合计），仅验证了字段存在与数值类型。
4. 我只校验了 `clientCountryName="JP"` 这一过滤条件下的导出；该条件本身是否正确无法从数据外部证明（依赖导出脚本的过滤器，已阅读确认与声明一致）。
5. 工作区根目录的 `jp-client-ips.csv`/`jp-client-ip-hosts.csv` 与 siteproxy 相关导出不在本任务范围，未校验。

## 五、证据文件

- `.cf-export-tmp/verify/jp-check.json`：逐行结构、总量、byDay/byHour/bySource、sampleInterval 分布
- `.cf-export-tmp/verify/jp-parts-check.json`：31 分片边界与合计比对
- `.cf-export-tmp/verify/jp-api-check.json`：5 小时抽样、独立总量 A/B、尾部检查
- `.cf-export-tmp/verify/jp-count-dist.json`：count 与 sampleInterval 分布
