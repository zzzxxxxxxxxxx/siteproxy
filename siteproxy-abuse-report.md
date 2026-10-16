# siteproxy 未授权使用情况报告

- 报告对象：`siteproxy`（Cloudflare Worker；账号邮箱与账号 ID 不在本报告中列出，如需核对可向报告方索取）
- 服务域名：`siteproxy.<自定义域名>`（zone ID 同样不在本报告中列出）
- 数据窗口：2026-08-22 至 2026-10-16（部分指标为近 28 天或当日）
- 数据来源：Cloudflare GraphQL 分析接口（`workersInvocationsAdaptive`、`workersSubrequestsAdaptiveGroups`、`httpRequestsAdaptiveGroups`）+ Cloudflare 仪表盘用量页
- 补充分析（第八节）：2026-10-17 用 zone 级 `httpRequestsAdaptiveGroups` 按 `clientCountryName = "JP"` 导出逐分钟请求明细，窗口 2026-09-16T16:03Z 至 2026-10-16T15:44Z
- 报告生成时间：2026-10-16 23:40 CST；第八节补充于 2026-10-17 02:40 CST

## 一、摘要

- 自 **2026-08-27** 起，`siteproxy` 的请求量从每天 2000-3500 次（运营者自用量级）逐步爬升，9 月 10 日之后稳定在 **每天 7 万至 18 万次**。
- 近 28 天累计 **3,292,820 次子请求**，来自 **1,933 个不同目标主机**；同一窗口 Worker 调用约 357 万次。
- 该量级由**非运营者的第三方**产生：其请求来源集中在日本大阪的三个机房 IP 与一个 Cloudflare WARP IPv6 地址，User-Agent 为 macOS Safari，访问内容以日文漫画站为主。
- 该流量使**账号级**的免费额度（10 万次/天）在当日被耗尽，触发 Cloudflare `error code: 1027`，**账号下所有 Worker（含 `dowss`）在额度恢复前全部返回错误**。
- 内容构成上除日文漫画外还有大量视频转发：剔除 YouTube 后视频类请求 **66,722 次 / 74.7 GB**，其中成人相关内容合计 **128,099 次（占窗口总请求 2.85%）**，详见第八节。

## 二、时间线

### 2.1 每日请求量（siteproxy，单位：次）

| 日期 | 请求数 | 日期 | 请求数 | 日期 | 请求数 |
|---|---|---|---|---|---|
| 08-22 | 3,530 | 09-10 | 114,850 | 09-29 | 142,065 |
| 08-23 | 2,850 | 09-11 | 70,380 | 09-30 | 74,365 |
| 08-24 | 2,230 | 09-12 | 42,660 | 10-01 | 70,669 |
| 08-25 | 2,000 | 09-13 | 103,150 | 10-02 | 101,507 |
| 08-26 | 3,180 | 09-14 | 50,210 | 10-03 | 102,532 |
| **08-27** | **6,630** | 09-15 | 86,710 | 10-04 | 153,448 |
| **08-28** | **14,800** | 09-16 | 105,930 | 10-05 | 130,755 |
| 08-29 | 14,410 | 09-17 | 87,160 | 10-06 | 116,449 |
| 08-30 | 16,690 | 09-18 | 80,930 | 10-07 | 70,321 |
| 08-31 | 20,750 | 09-19 | 64,264 | 10-08 | 140,802 |
| 09-01 | 9,370 | 09-20 | 96,506 | 10-09 | 137,121 |
| **09-02** | **44,560** | 09-21 | 119,013 | 10-10 | 135,896 |
| 09-03 | 8,000 | 09-22 | 112,210 | 10-11 | 139,126 |
| 09-04 | 25,380 | 09-23 | 172,311 | 10-12 | 181,512 |
| 09-05 | 25,870 | 09-24 | 114,437 | 10-13 | 133,699 |
| 09-06 | 42,950 | 09-25 | 102,607 | 10-14 | 136,140 |
| 09-07 | 41,450 | 09-26 | 101,706 | 10-15 | 132,651 |
| 09-08 | 64,270 | 09-27 | 179,081 | 10-16 | 137,688 |
| 09-09 | 58,470 | 09-28 | 83,825 | | |

起量点明确落在 **08-27 至 08-28**：此前日均 2000-3500，两天内升到 6,630 / 14,800，随后三周持续爬升，9 月中旬进入稳定高位。

### 2.2 逐小时分布（2026-10-16，UTC）

| 小时(UTC) | 请求数 | 小时(UTC) | 请求数 |
|---|---|---|---|
| 00:00 | 11,800 | 08:00 | **239** |
| 01:00 | 8,045 | 09:00 | 4,744 |
| 02:00 | 16,999 | 10:00 | 1,884 |
| 03:00 | 11,801 | 11:00 | 6,785 |
| 04:00 | 14,122 | 12:00 | **19,885** |
| 05:00 | 11,687 | 13:00 | 5,373 |
| 06:00 | 11,245 | 14:00 | 3,670 |
| 07:00 | 9,409 | | |

换算为北京时间（UTC+8）后呈现明显的人类作息：白天高位、16:00 前后骤降到 239、晚间 20:00 冲至峰值。这不是等间隔轮询的爬虫形态。

## 三、使用方识别

### 3.1 来源地址（2026-10-16）

| 请求数 | IP | 归属 | 反向解析 |
|---|---|---|---|
| 59,607 | 163.43.139.100 | 日本大阪，SAKURA Internet（AS9370） | 无 PTR |
| 41,929 | 2a06:98c0:3600::103 | Cloudflare 自有网段（WARP 出口） | — |
| 37,209 | 163.43.136.36 | 日本大阪，SAKURA Internet | 无 PTR |
| 34,553 | 163.43.134.149 | 日本大阪，SAKURA Internet | 无 PTR |
| 4,263 | 40.77.178.28 | Microsoft Azure（Quincy） | `msnbot-40-77-178-28.search.msn.com` |
| 1,231 | 199.30.24.249 | Microsoft Azure（Quincy） | `msnbot-199-30-24-249.search.msn.com` |
| 313 / 265 / 241 | 34.32.50.125 / 34.65.4.128 / 65.55.210.208 | 德国 / 瑞士 / 美国 | — |
| **102** | 104.28.208.136 | **中国（运营者本人）** | — |

Microsoft 两台是必应爬虫（使用 HeadlessChrome 渲染 UA）。运营者本人当日仅 102 次。

### 3.2 客户端特征

四个主要来源地址（三个大阪 IP + 一个 WARP IPv6）发来的 User-Agent **完全一致**，说明是同一客户端经由一条日本出口链访问，而非多个独立用户：

| 请求数 | User-Agent |
|---|---|
| 124,469 | `Macintosh; Intel Mac OS X 10_15_7 … Version/26.5.2 Safari/605.1.15` |
| 31,349 | 同机型，`Version/26 Safari/605.1.15` |
| 9,858 | `Windows NT 10.0; Win64; x64 … HeadlessChrome/146.0.7680.165`（必应渲染） |
| 8,564 | 同机型，`Version/18.7.5 Safari` |
| 5,561 | `iPad; CPU OS 18_7 … Mobile/15E148` |
| 3,080 | 同机型，`Version/27.0.1 Safari` |
| 1,761 | `bingbot/2.0` |

存在多个 Safari 版本，指向少数几台设备（Mac 与 iPad）共用同一条日本出口。

## 四、访问内容分布

### 4.1 分类（近 28 天，共 3,292,820 次子请求 / 1,933 个主机）

| 类别 | 子请求数 | 占比 | 说明 |
|---|---|---|---|
| 漫画 | 约 1,800,000 | 约 55% | `soraraw.com`、`i.mangaraw.lat`、`lh1-4.rawcontent.top`、`iphotomg.com`、`gomuraw.*`、`mangarawad.*` |
| 视频 | 约 140,000 | 约 4% | `r1/r3/rr5---sn-*.googlevideo.com`（YouTube 分片） |
| 音频 | 52,970 | 1.6% | `i1.sndcdn.com`、`api-v2.soundcloud.com` |
| 成人内容 | 68,450 | 2% | 52 个主机：`*.phncdn.com`、`jp.pornhub.com`、`d-hentai.net`、`*.dmm.co.jp`、`supjav.com`、`missavtv.com`、`xvideo-jp.com` 等 |
| 广告 / 统计 | 115,890 | 3.5% | 242 个主机：`google-analytics.com`、`pixel.gliacloud.com`、`googletagmanager.com`、`*.doubleclick.net`、`taboola`、`criteo`、`inmobi` 等 |
| CSP 违规报告 | 59,370 | 1.8% | `csp-reporting.cloudflare.com` |
| 其他 | 其余 | — | 搜索（DuckDuckGo、Bing）、日本票务（`l-tike.com`、`image.pia.jp`）、游戏攻略（`pokemongo-raku.com`、`img.game8.jp`）、购物（`dospara.co.jp`）、开发（`github.com`） |

成人内容一项，第八节用更细的关键词表与 30 天窗口重算为 128,099 次（2.85%），与本表 28 天粗分类的 68,450 次口径不同，两者不矛盾。

### 4.2 目标主机前 30（近 28 天，子请求数）

```
 509310  soraraw.com                    27630  pokedex.pokemongo-raku.com
 488020  i.mangaraw.lat                 25720  pokemongo-raku.com
 288200  lh3.googleusercontent.com       21000  rr5---sn-3pm7dnel.googlevideo.com
 157360  iphotomg.com                    19690  d-hentai.net
 136560  lh3.rawcontent.top              19280  gomuraw.mobi
 129670  lh1.rawcontent.top              18250  api.mangarawgo.site
 127940  lh2.rawcontent.top              17410  smaccams.com
 123670  lh4.rawcontent.top              16930  mangarawad.casa
  59370  csp-reporting.cloudflare.com    16460  img.game8.jp
  58840  cdn.ddd-smart.net               16140  duckduckgo.com
  56670  r1---sn-oguesnd6.googlevideo.com 15560 gomuraw.im
  52970  i1.sndcdn.com                   14930  pixel.gliacloud.com
  52200  media（非法主机名，见 6.1）       12440  github.com
  51520  r3---sn-oguesn6r.googlevideo.com 12200 ev-h.phncdn.com
  47130  r1---sn-ogueln6y.googlevideo.com 11500 fundingchoicesmessages.google.com
  30100  www.google-analytics.com        11500  improving.duckduckgo.com
  28180  fonts.gstatic.com               11410  mangarawad.wiki
```

（完整 1,933 个主机名的聚合视图在 Cloudflare 仪表盘的 siteproxy「指标 → 子请求」页可直接查阅。）

## 五、对账号的影响

| 项 | 数值 |
|---|---|
| siteproxy 当日调用（10-16） | 137,688 次 |
| 账号级当日额度 | 100,000 次/天（免费版，全账号共享） |
| 账号当日实际用量 | 137,819 次 → **超额** |
| 结果 | 账号下所有 Worker 返回 `HTTP 429 / error code: 1027`，直到 00:00 UTC 重置 |
| 近 30 天调用 / 子请求 | 3.57M / 3.48M |
| 同窗口错误 | 1.72k |
| Durable Object 用量（对照） | 持续时间 1.12k GB-s（限额 13,000/天）、请求 2k（限额 100,000/天），**未接近上限** |

即：账号额度是被 siteproxy 的单一来源吃光的，与 Durable Object 无关。

## 六、密码外泄机制（本地复现结论）

运营者的访问口令以 URL 路径形式存在（形如 `/<口令>/`，具体值不在本报告中列出），以下为在本地 `wrangler dev` 运行同一份代码（`build/worker.js`）并配合可记录请求头的上游站点复现的结果：

1. **口令对页面内所有脚本可见**。代理返回的 HTML 头部被注入如下代码，两个变量为顶层 `var`，即 `window` 属性：
   ```js
   var proxy_url_prefix = 'http://<代理域名>/<口令>/';
   var config_token_prefix = '/<口令>/';
   ```
   同时页面中所有链接、图片、脚本地址均被重写为携带口令的 URL。任何第三方脚本都可读取 `location.href`、`window.config_token_prefix` 或 DOM 中的 URL 并外发。
2. **HTML 属性中的绝对地址会被重写，JS 字符串中的不会**。实测 `fetch('…')`、`s.src='…'`、`new Image().src='…'` 三种写法在返回的 HTML 中均保持原样。
3. **service worker 无法注册**：`/siteproxy_service_worker.js` 返回 404，客户端拦截层未生效，因此页面脚本发出的跨域请求不受任何约束。
4. **公开资源不含口令**：`/siteproxy-response-injected.js`（无口令，200，134,269 字节）内既无口令明文，也无硬编码外发端点；`/`、`/sync`、`/cs`、错误口令路径均为 404。即不存在"无口令即可直接读出当前口令"的接口。
5. **第三方接收面很大**：上述第 4.1 节的 242 个广告/统计主机，以及 **59,370 次 CSP 违规报告**——按规范该报告必须包含 `document-uri`（触发报告的文档地址，即携带口令的代理 URL）。

综合：口令会在**代理被使用的那一刻**流向页面中的第三方脚本与上报端点。轮换口令只能使旧值失效，不能阻止新值在下次使用时再次外泄。

## 七、已采取措施与待验证

| 时间 | 措施 |
|---|---|
| 2026-10-16 23:35 CST | `token_prefix` 已轮换为新的 24 位随机值（已通过 Cloudflare API 只读核对，旧值已从配置中移除；新旧值均不在本报告中列出） |
| 2026-10-17 08:00 CST | 免费额度重置，服务恢复 |
| 2026-10-17 10:30 CST | 计划中的首次判定 |

判定口径：

| 观察结果 | 结论 | 处置 |
|---|---|---|
| 流量降至接近零（仅剩运营者自用） | 对方仅握有旧口令 | 维持现状，或加 Access 加固 |
| 旧口令路径仍被那三个大阪 IP 持续请求 | 对方在重试旧地址，确认未取得新口令 | 同上 |
| 新口令已被那三个大阪 IP 使用 | 存在自动再外泄通道，轮换无效 | 改用 Cloudflare Access 身份验证或 WAF 来源白名单 |

## 八、成人内容与视频流量专项（2026-10-17 补充）

本节数据源与前面各节不同：zone 级 `httpRequestsAdaptiveGroups`，按 `clientCountryName = "JP"` 过滤，窗口 **2026-09-16T16:03Z 至 2026-10-16T15:44Z**（zone 数据集保留期上限 4w3d），共 2,541,447 行分组、**4,498,666 次请求**、16 个来源 IP、2,547 个目标主机。口径比第四节 4.1 的 28 天粗分类更细、窗口更长，两边数字不可直接对齐。

### 8.1 视频类请求

剔除 `googlevideo.com`（YouTube 媒体流）后：

| 类别 | URL 数 | 请求数 |
|---|---|---|
| `.ts`（HLS 分片） | 23,105 | 37,259 |
| `.mp4` | 2,909 | 12,715 |
| `.m4s`（soundcloud 分片） | 4,842 | 9,534 |
| `.m3u8`（播放列表） | 4,770 | 7,058 |
| 其他（`ct:mp4`、`.mov`、`.webm`） | 29 | 156 |
| **合计** | **35,655** | **66,722**（74.7 GB） |

归一化后不同视频文件 **20,422 个**（`.ts` 12,882、`.m4s` 3,004、`.m3u8` 2,895、`.mp4` 1,626）。若一并计入 `googlevideo`，则为 542,680 次请求 / 103.7 GB。

请求数含 `eyeball`（客户端→siteproxy）与 `edgeWorkerFetch`（Worker→源站）两个视角，同一次取数计两次，实际拉取次数约为表中数字的一半。

### 8.2 成人内容统计

按域名与路径关键词命中 **128,099 次请求**（占窗口总请求 **2.85%**），其中视频文件 52,472 次 / 68.5 GB。

| 分组 | 代表域名 | URL 数 | 请求数 | 其中视频 | 字节 |
|---|---|---|---|---|---|
| Pornhub | `*.phncdn.com`、`jp.pornhub.com`、`i.pornhub.com` | 35,316 | 84,846 | 45,532 | 41.24 GB |
| 成人漫画 | `d-hentai.net` | 2,094 | 20,796 | 0 | 1.53 GB |
| DMM / FANZA | `cc3001.dmm.co.jp`、`pics.dmm.co.jp`、`doujin-assets.dmm.co.jp` | 3,246 | 5,937 | 1,250 | 11.17 GB |
| 成人广告网络 | `*.trafficjunky.net`、`tsyndicate.com`、`*.adtng.com` | 705 | 3,459 | 362 | 0.47 GB |
| JAV 站 | `javynow.com`、`topjav.tv`、`supjav.com`、`stream-48-cdn.b-cdn.net`（路径含 `topjav`） | 1,911 | 3,157 | 1,236 | 1.25 GB |
| TokyoMotion | `www32`~`www47.tokyomotion.net` | 414 | 2,225 | 1,216 | 14.67 GB |
| 其他成人 CDN | `*.bkcdn.net`、`*.sacdnssedge.com`、`*.vcmdiawe.com`、`playhubconnect.com` | 1,337 | 3,492 | 2,801 | 2.73 GB |
| 直播站 | `*.livejasmin.com` 等 | 66 | 142 | 0 | 0.00 GB |
| 其他零散命中 | — | 689 | 4,045 | 75 | 0.57 GB |

请求数最多的域名：`ev-h.phncdn.com` 24,556（16.82 GB）、`d-hentai.net` 20,662、`ei.phncdn.com` 19,065、`em-h.phncdn.com` 14,030（16.88 GB）、`jp.pornhub.com` 12,595（其中多为 `/_xa/ads_batch`、`/_xa/deep_pixel` 等广告信标）。单文件体积最大的是 TokyoMotion 的整段 MP4（最大 765 MB），其余为 Pornhub 的 HLS 分片与 DMM 的预览片。

### 8.3 失效链接反推

导出的 CDN 直链基本都带时效签名，事后打开多为 403 或 404。可按路径中的稳定 ID 反推稳定页，已实测：

| 站点 | 失效链接中的 ID | 反推出的稳定页 | 实测结果 |
|---|---|---|---|
| TokyoMotion | `/video/<md5>/<hash>/iphone/<数字ID>.mp4` | `https://www.tokyomotion.net/video/<数字ID>` | 34 条中 **30 条 200**、4 条 404（视频已被站方删除） |
| DMM / FANZA | `/pv/<token>/<cid>mhb.mp4`、`litevideo/freepv/x/xyz/<cid>/<cid>_mhb_w.mp4`、`pics.dmm.co.jp/digital/video/<cid>/<cid>pl.jpg` | `https://www.dmm.co.jp/digital/videoa/-/detail/=/cid=<cid>/` | 301 跳转至 FANZA 区域限制页（非日本 IP 不可用） |
| javynow | 日志中的 `/player/<数字ID>/` | `https://javynow.com/player/<数字ID>/` | 200 |
| d-hentai | `/ja/get/<数字ID>` | `https://d-hentai.net/ja/get/<数字ID>` | 200 |
| topjav | `/m3u8/<数字ID>.m3u8`、`/sdc48/topjav/<数字ID>/indexN.ts` | `https://topjav.tv/video/<数字ID>` | 403（站点 bot 防护，不代表失效） |
| supjav | `/ja/<数字ID>.html` | `https://supjav.com/ja/<数字ID>.html` | 403（同上） |

Pornhub **无法反推**：phncdn 分片路径里的数字 ID 是内部编号，公开页需要 `view_video.php?viewkey=ph<十六进制>`，而免费计划的 GraphQL 不返回 `clientRequestQuery` 维度，日志中没有查询串，viewkey 无从获得；实测 `https://www.pornhub.com/video/show?id=<数字ID>` 会被重定向到 `/video`。带 `key=...,end=...` 时效签名的链接（如 javynow）签名无法重算，只能退回页面级地址重新取流。

### 8.4 本节产出的文件

| 文件 | 内容 |
|---|---|
| `siteproxy-subrequests-by-minute.csv` | Worker 子请求逐分钟明细，2026-07-19 至 2026-10-16，357,490 行 / 4,544,490 次子请求 / 4,646 个主机 |
| `jp-requests.ndjson.gz` | zone 上日本来源请求逐分钟明细，2,541,447 行 / 70 MB |
| `jp-video-urls.csv`、`jp-video-requests.csv` | 视频类 URL 与明细（已剔除 googlevideo） |
| `jp-adult-urls.csv` | 成人相关内容 URL 汇总，45,778 行 |
| `jp-adult-reconstructed-links.csv` | 反推出的稳定页地址，120 条（含 `evidence` 与实测状态列） |
| `jp-client-ips.csv`、`jp-client-ip-hosts.csv` | 日本来源 IP 汇总 |
| `siteproxy-subrequests-verification.md`、`jp-export-verification.md` | 独立校验报告 |

这些文件含第三方逐条浏览记录，对外提交本报告时应按第九节处理。

## 九、数据范围与限制

- 第一至七节、第十节为**聚合数据**（按天/小时/来源 IP/目标主机计数），不含逐条访问 URL、查询串与时间戳。
- 第八节记录的补充导出（`jp-requests.ndjson.gz`、`jp-video-*.csv`、`jp-adult-*.csv`、`siteproxy-subrequests-by-minute.csv`）包含**逐条 URL 与分钟级时间戳**，与原报告正文分开存放；对外提交时建议只保留正文，或先删除、脱敏这些附件。
- 技术原因：Cloudflare 分析接口按维度聚合返回，单次结果有行数上限，且不保留查询串与逐条记录；查询串（如 Pornhub 的 `viewkey`）在本账号的套餐下不可取，故第八节也无法还原具体视频页。
- 隐私原因：逐条浏览记录属于第三方的个人数据。公开逐条 URL（尤其涉及成人内容）与该人滥用服务的程度不成比例，且会使公开方在个人信息保护法规下面临风险。故第八节的逐条数据建议仅作内部取证留存，不予公开。

## 十、可用于进一步举证的对象

| 对象 | 事由 |
|---|---|
| Cloudflare 支持 | 第三方盗用 Worker、导致账号级额度耗尽与 1027 中断 |
| SAKURA Internet（AS9370）滥用举报 | `163.43.139.100`、`163.43.136.36`、`163.43.134.149` 的租用主机持续盗用他人代理服务 |
| 被代理站点 | 若需说明流量来源 |
