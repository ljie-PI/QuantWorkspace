# A 股全市场 1 分钟历史与实时数据可得性

> 调研日期：2026-09-03
> 范围：内部使用的沪、深、北 A 股全市场 1 分钟历史与实时 bar；只陈述证据与首版约束，不选择最终架构。
> “未核实”表示在可公开访问的一手资料中没有找到明确承诺，不能按“支持”处理。

## 结论摘要

1. **AKShare 不能单独满足完整历史库**：东方财富 1 分钟接口只给最近 5 个交易日且不复权；新浪接口实际请求滚动 1,970 根 bar。两者都是对公开网页接口的封装，没有 SLA、修订流或稳定配额，且 AKShare 明示数据仅限学术研究、不可商业使用。
2. **Tushare Pro 是公开资料中规格最明确的托管分钟 API**：A 股历史分钟从 2009 年起，1/5/15/30/60 分钟，个人历史权限 ¥2,000/年；实时分钟 ¥1,000/月。机构价格为个人 10 倍。历史数据收盘后 17:00–21:00 处理，实时股票分钟文档只说明拉取 API，没有 WebSocket 或数值延迟承诺。
3. **Qlib 是数据格式、研究和回放框架，不是生产行情源**：微软官方可下载数据集目前明确“临时停用”；Yahoo 1 分钟采集器默认只取约 29 天，当前中国股票映射只覆盖 6 开头沪市和 0/3 开头深市代码，不覆盖北交所。
4. **miniQMT/xtdata 同时具备本地历史缓存与实时订阅能力**，但可得历史深度、北交所全推、非 VIP 精确订阅上限、延迟、修订政策和 SLA 均未公开承诺；能力和券商终端/行情站点/权限一致，不能把“某家券商可用”外推为通用事实。
5. **商业数据商和交易所官方产品是真正可签约的备选**。Wind WDS 官方说明覆盖任意周期 K 线、Tick、逐笔、委托队列、复权因子并提供低延迟传输，但公开页不披露 A 股分钟起始日、字段级规格、价格或 SLA。上交所经 CIIS 提供的官方 Level-1 日内快照历史数据含 snapshot 和 K-line，自 2014-01-02 起，US$1,250/3 个月，交易日 21:00 更新；它只解决沪市历史的一部分。
6. **JQData、RQData 是可采购/试用的托管数据 API**。公开一手代码可确认分钟历史、复权和实时/当前行情能力，但公开页面不足以确认北交所完整性、实时 bar 延迟、全市场并发配额、正式 SLA、修订流和当前合同价格。
7. 对首版最强的约束不是 SDK，而是：**北交所是否完整、内部公司使用是否合法、实时是否能在一分钟收盘后稳定到达、历史与实时 bar 定义是否一致、停牌/零成交如何表达、复权因子是否会回溯变化、以及供应商如何发布纠错**。

## 比较表

| 来源 | 字段与市场覆盖 | 历史深度与复权 | 实时可用性 | 稳定性、配额、成本与条款 | 纠错与离线回放 |
|---|---|---|---|---|---|
| **AKShare：东方财富 / 新浪** | 分钟字段含时间、OHLC、成交量、成交额；东方财富 1 分钟另有均价。文档称沪深京 A 股，但单接口/单标的完整性需实测。 | 东方财富 1 分钟仅最近 5 个交易日且固定不复权；新浪请求最近 1,970 根，可选不复权/qfq/hfq。新浪分钟复权是用每日最后一根分钟 close 对齐相应复权日线 close 后按日缩放，不是交易所原生复权分钟。 | 可轮询最近 bar；未发现推送、数值延迟或 bar-final 标志。新浪全市场现货函数警告重复运行会临时封 IP。 | SDK 为 MIT，但项目另行声明接口和数据只供学术研究、不可商业使用；无 token 的接口也不等于无上游限流。无 SLA，接口可能移除；近期 changelog 曾修复新浪分钟接口。 | 无正式纠错/版本政策。可自行落盘，但历史浅、上游可变，不足以形成可审计的全历史回放源。 |
| **Tushare Pro** | `stk_mins` / `rt_min`：代码、时间、OHLC、成交量（股）、成交额（元），1/5/15/30/60 分钟。实时页称“全 A 股票”；北交所分钟完整性未单独列明。 | 历史起始 2009 年，超过 10 年；单次 8,000 行。分钟接口无复权参数，`pro_bar` 明确复权目前只支持日线，因此分钟应按不复权处理。 | `rt_min` 拉取一个或多个代码，单次最多 1,000 行；`rt_min_daily` 拉取单只股票当日开盘以来全部分钟。未见股票分钟 WebSocket、数值延迟或最终 bar 标志。 | 个人：历史 ¥2,000/年、500 次/分钟；实时 ¥1,000/月、500 次/分钟，价格表称单次最多 300 家公司。机构价为 10 倍。需要账号、token、独立权限。分钟数据页称仅供策略研究和学习，不允许商业目的；机构内部使用是否获许可必须书面确认。无 SLA。 | 历史分钟每日收盘后 17:00–21:00 处理。未找到正式回溯修订/撤销通知机制。下载后适合自建 bar 回放，但供应商不提供版本化重放。 |
| **Microsoft Qlib 数据集/Provider** | Qlib 数据至少要求 adjusted open/close/high/low/volume 和 `factor`，可加入 `money`；停牌时这些字段按约定置 NaN。当前采集器的沪深股票代码映射不含北交所。 | 微软官方数据集目前停用。Yahoo 采集器 1 分钟默认窗口约 29 天；原官方 1 分钟包的精确日期范围未核实。Qlib 将每只股票首个交易日归一到 1，通常 `factor = adjusted / original`，原始 close 为 `$close / $factor`。 | 不提供官方实时行情源；只能处理用户自行送入的新数据。 | 框架 MIT、无框架调用费；这不授予底层行情数据权利。微软明确提醒 Yahoo 示例数据可能不完美。社区替代数据集的来源、许可和北交所覆盖未核实。 | 本地 `.bin`、特征读取、缓存和回测适合离线 bar 回放；确定性依赖冻结数据快照。没有行情供应商级纠错流或 tick 播放语义。 |
| **券商 MiniQMT / xtdata** | 1m/5m/1d K 线字段：time、OHLC、volume、amount、preClose、suspendFlag，另有结算价和持仓量字段。全推 tick 可订阅市场代码 `SH`、`SZ`；文档已增加北交所板块，但北交所全推/分钟订阅是否完整未核实。 | 支持历史下载、批量下载、增量下载和本地读取；公开文档未承诺 A 股 1 分钟最早日期。K 线支持 none/front/back/front_ratio/back_ratio；tick 不适用。`fill_data` 默认 True，会向后填补空缺。 | 单股周期订阅可推 1 分钟；VIP 文档称支持全市场指定周期 K 线。全推 tick 适合高订阅数。无数值延迟或 bar-final 承诺。 | 必须连接 MiniQMT/券商 QMT 或迅投数据中心，能力与终端使用的行情服务器及权限相同。建议单股订阅不超过 50；非 VIP 数量受限但精确值未披露。VIP token 单点访问，可由一个本地服务进程转发给其他进程。券商开户、资产门槛和价格因券商而异；迅投产品页价格需登录/询价。无公开 SLA。 | 订阅数据通常进入本地缓存；`get_local_data` 可批量读取本地 Level-1 历史，离线 bar 回放条件较好。Level-2 不保存跨日历史。未找到历史文件修订、校验和或版本政策。 |
| **JQData** | 官方 SDK 的 `get_price` 默认字段为 OHLC、volume、money，还支持 factor、涨跌停价、均价、前收和停牌标志；`get_current_tick(s)` 提供当前 tick。官方 README 称覆盖国内二级市场可交易品种及分钟/Tick bar。 | `get_price` 支持 minute，复权 `fq=pre/post/none`；`get_bars` 用 `fq_ref_date` 指定复权基准日。公开可访问资料未核实 A 股 1 分钟最早日期和北交所逐日完整性。 | 有当前 tick API；实时 1 分钟的刷新、延迟、最终状态和全市场批量限制未核实。 | SDK MIT；数据权利和商业内部使用受服务合同约束。README 的 5ms 响应、24 小时不中断、灾备等是厂商陈述，不是公开 SLA。配额可通过 `get_query_count()` 查询；当前套餐价格和正式额度未能从无需登录的一手页面核实。 | 厂商称有秒级异常响应和多源比对，但未找到公开修订流/历史版本说明。可下载落盘后回放。 |
| **RQData / RQAlpha 扩展 API** | 官方仓库称 A 股行情自 2005 年至今并含实时行情；`get_price` 支持日/分钟、OHLC、涨跌停价、成交额、成交量等。公开资料主要明确沪深代码体系，北交所未核实。 | `get_price` 支持 `pre/post/none`；`skip_suspended=False` 时会用停牌前数据补齐。分钟数据最早日期是否也等于 2005 年需合同/试用确认。 | 官方称含实时行情（连续竞价时段）；实时 bar/tick 接口、延迟和全市场吞吐未在已访问公开文档中充分核实。 | 需要 RQData license，可申请试用或咨询私有化部署。公开报价、配额和 SLA 未核实。RQAlpha 许可明确：法人或组织任何用途均需米筐授权；数据许可另以合同为准。 | 可本地 API 获取并自行快照；未找到公开纠错流和版本承诺。停牌默认补齐会影响事件回放，采集层必须显式记录参数。 |
| **Wind WDS** | 官方页称提供历史 Tick、任意周期 K 线、日 K、逐笔成交、委托、委托队列、复权因子，并汇聚多个交易所。精确 A 股 1 分钟字段和北交所范围需产品清单确认。 | 历史起始日和复权算法未公开。 | 官方称统一标准、低延迟、高质量传输，服务机构内部研究、交易、算法交易等；无公开数值延迟。 | 商业合同/询价；支持数据库、API、文件同步等。官网“全面、稳定”是服务陈述，公开 SLA、额度和价格未找到。 | 官方称有历史行情智能检测和修复工具；修订通知、旧版本保留和重放包格式需合同确认。 |
| **交易所官方历史产品** | CIIS 的上交所 Level-1 日内历史产品含 snapshot 和 K-line；这是官方源，但只覆盖上交所。深交所/北交所需分别通过其授权渠道补齐。 | 上交所 Level-1 日内历史自 2014-01-02，US$1,250/3 个月。精确 1 分钟字段、复权因子和更早数据需查产品手册/样例。 | 该产品是历史交付，不是实时 feed。 | 持续订阅每个交易日 21:00 更新；云端只保留最近两周且每个交易日最多下载 3 次，较大历史通过介质或 SFTP。正式使用需订单/许可。 | 官方来源最适合做基准校验；公开产品页未说明替换文件、版本保留和纠错通知。全市场仍需深交所、北交所对应合同。 |

## 关键证据

### AKShare

- 东方财富分钟实现写明 1 分钟只返回最近 5 个交易日，且 1 分钟不支持复权；5/15/30/60 分钟可传 qfq/hfq。
  [AKShare `stock_hist_em.py`（固定版本）](https://github.com/akfamily/akshare/blob/8e95744b79ae22326308ccd2b4e62650c5b53c55/akshare/stock_feature/stock_hist_em.py#L1071-L1167)
- 新浪分钟实现把 `datalen` 固定为 1,970；qfq/hfq 是按每日最后一根分钟 close 与相应复权日线 close 的比例缩放该日 OHLC。
  [AKShare `stock_zh_a_sina.py`](https://github.com/akfamily/akshare/blob/8e95744b79ae22326308ccd2b4e62650c5b53c55/akshare/stock/stock_zh_a_sina.py#L359-L445)
- 软件 MIT 与数据使用限制是两件事：
  [MIT License](https://github.com/akfamily/akshare/blob/8e95744b79ae22326308ccd2b4e62650c5b53c55/LICENSE#L1-L21)；[数据仅限学术研究、接口可能移除](https://github.com/akfamily/akshare/blob/8e95744b79ae22326308ccd2b4e62650c5b53c55/docs/special.md#L77-L83)。

### Tushare Pro

- [股票历史分钟行情](https://tushare.pro/document/2?doc_id=370)：字段、频率、8,000 行上限、超过 10 年历史。
- [权限与价格](https://tushare.pro/document/2?doc_id=290)：历史从 2009 年、历史/实时价格、频次、机构 10 倍价格。若内部系统按机构计价，页面标价对应约 **¥20,000/年历史 + ¥10,000/月实时**，是否含税及是否授予相应用途权利仍需合同确认。
- [A 股实时分钟](https://tushare.pro/document/2?doc_id=374) 与 [当日分钟累计](https://tushare.pro/document/2?doc_id=457)：均为拉取接口。
- [分钟使用说明](https://tushare.pro/document/1?doc_id=234)：仅限研究学习、收盘后 17:00–21:00 更新。
- [`pro_bar`](https://tushare.pro/document/2?doc_id=109)：复权目前只支持日线。

### Microsoft Qlib

- 官方 README 当前明确写明数据集因更严格的数据安全政策而临时停用，并提醒 Yahoo 数据可能不完美：
  [README data preparation](https://github.com/microsoft/qlib/blob/79633dd9506ea689e5400dea0197717b5b3d74b7/README.md#L211-L254)。
- Qlib 格式、调整语义、停牌 NaN 和至少字段要求：
  [data.rst](https://github.com/microsoft/qlib/blob/79633dd9506ea689e5400dea0197717b5b3d74b7/docs/component/data.rst#L52-L69)，[字段说明](https://github.com/microsoft/qlib/blob/79633dd9506ea689e5400dea0197717b5b3d74b7/docs/component/data.rst#L149-L198)。
- 当前采集器仅将 `6...` 映射到沪市，将 `0...`/`3...` 映射到深市：
  [collector utils](https://github.com/microsoft/qlib/blob/79633dd9506ea689e5400dea0197717b5b3d74b7/scripts/data_collector/utils.py#L178-L265)。
- 1 分钟默认采集窗口约 29 天：
  [collector base](https://github.com/microsoft/qlib/blob/79633dd9506ea689e5400dea0197717b5b3d74b7/scripts/data_collector/base.py#L20-L29)。

### MiniQMT / xtdata

- [xtdata 运行逻辑、周期、请求限制、历史下载、本地读取、字段与复权](https://dict.thinktrader.net/nativeApi/xtdata.html)
- [新版订阅说明](https://dict.thinktrader.net/nativeApi/xtdata_new.html)：非 VIP 限制订阅数量，VIP 支持全市场指定周期 K 线；能力与 MiniQMT 行情服务器一致。
- [VIP 行情和 token 使用](https://dict.thinktrader.net/dictionary/)：token 单点访问、本地数据服务转发、券商 QMT 配置。

### 其他可采购来源

- JQData 官方仓库：
  [产品能力与厂商稳定性陈述](https://github.com/JoinQuant/jqdatasdk/blob/ba61143a6504bb1f797b16b8f9d23a21909c4f60/README.md)；[`get_price`、`get_current_tick(s)`、`get_query_count`](https://github.com/JoinQuant/jqdatasdk/blob/ba61143a6504bb1f797b16b8f9d23a21909c4f60/jqdatasdk/api.py)；[SDK MIT License](https://github.com/JoinQuant/jqdatasdk/blob/ba61143a6504bb1f797b16b8f9d23a21909c4f60/LICENSE)。
- RQData / RQAlpha：
  [官方 README 的 2005 年至今及实时行情说明](https://github.com/ricequant/rqalpha/blob/7c9305ad1f8d40049b526e097cd2e6c106bdb481/README.rst#rqdata数据本地化服务)；[`get_price` 分钟、字段、复权与停牌语义](https://github.com/ricequant/rqalpha/blob/7c9305ad1f8d40049b526e097cd2e6c106bdb481/rqalpha/apis/api_rqdatac.py)；[试用/license 前置](https://github.com/ricequant/rqalpha/blob/7c9305ad1f8d40049b526e097cd2e6c106bdb481/docs/source/api/extend_api.rst)；[法人/组织授权限制](https://github.com/ricequant/rqalpha/blob/7c9305ad1f8d40049b526e097cd2e6c106bdb481/LICENSE)。
- Wind WDS：
  [实时行情服务](https://www.wind.com.cn/portal/zh/WDS/marketdata.html)；[WDS 数据服务](https://www.wind.com.cn/portal/zh/WDS/index.html)；[历史行情、K 线、逐笔和修复能力](https://www.wind.com.cn/portal/en/WDS/marketdata.html)。
- 上交所官方历史数据经 CIIS：
  [产品、起始日、价格、更新与交付](https://www.ciis.com.hk/hongkong/en/historicaldata1/index.shtml)；[2026-08-19 产品手册](https://www.ciis.com.hk/hongkong/en/uploadfiles/202608/24/2026082405145816225281.pdf)。
- 深交所和北交所官方授权入口已定位，但页面在本次环境中无法稳定抓取正文，因此只作为待核实入口：
  [深圳证券信息有限公司历史数据](http://www.szsi.cn/cpfw/overseas/market/historical/)；[北交所境内行情授权指南](https://www.bse.cn/application/guide.html)。

## 首版设计受限事实

1. **“全 A”必须逐交易所验收。** 不能用“沪深 A 股”或泛称“A 股”替代沪、深、北全市场；尤其 Qlib 当前采集器明确缺北交所，miniQMT、Tushare、JQData、RQData 的北交所分钟完整性需要样本和合同双重确认。
2. **历史源和实时源不能默认同口径。** 需要验证分钟时间戳是区间起点还是终点、是否含 09:30 集合竞价 bar、午间如何处理、零成交分钟是否生成、15:00 如何归属。Tushare 示例包含 09:30 bar 和零成交分钟，已说明供应商会做自己的日内网格处理。
3. **停牌/缺失默认值会改变回放。** xtdata `fill_data=True`、JQData 可填充停牌、RQData `skip_suspended=False` 都可能复制前值；Qlib 则约定停牌为 NaN。采集时必须保存供应商原始返回和请求参数，不能只保存统一后的 OHLCV。
4. **复权不能当作原始事实。** AKShare 新浪复权是派生算法；Tushare 分钟无复权；Qlib 首日归一化；JQData、RQData、xtdata 又有不同基准日/前后复权语义。至少应把不复权 bar 与公司行动/复权因子分开审计，再生成版本化视图。
5. **实时“可取”不等于可驱动生产分钟闭合。** 除 miniQMT 推送外，公开确认的 Tushare 股票实时分钟是轮询 API；所有候选均缺公开数值延迟和最终 bar 标志。首版上线前必须用交易日实测 p50/p95/p99 到达延迟、重连补洞和重复/乱序。
6. **开源许可证不授予数据权利。** AKShare、Qlib、JQData SDK 的 MIT 只覆盖代码。AKShare 和 Tushare 对数据用途有额外限制；RQAlpha 对法人/组织使用要求授权。内部使用仍可能属于商业/组织使用，必须以书面合同为准。
7. **纠错是独立功能需求。** 除 Wind 对“检测和修复”的概括性陈述外，未找到候选的公开、机器可读 correction feed。首版数据层必须能识别同一 provider/symbol/minute 的后到修订，并保留原值、修订值、抓取时间和来源版本。
8. **请求规模需要容量验证。** 约 5,000+ 股票 × 240/241 bar/日会快速形成十亿级记录。Tushare 8,000 行/请求适合按标的分段回填；miniQMT 建议超过 50 个单股订阅改全推；AKShare 网页接口不适合并发全市场长期抓取。

## 设计含义（非架构选择）

- 建立供应商无关的最小原始字段集合：`provider`、`symbol`、`exchange`、`bar_start/end`、OHLC、volume、amount、trade_count（若有）、suspend/zero-trade 状态、adjustment mode、ingested_at、source_revision。
- 将 **raw/unadjusted**、调整因子、调整后视图分层；禁止把不同供应商的 qfq/hfq 值直接拼接。
- 历史回填、当日实时和次日核对视为三个不同阶段；记录实时值是否被收盘后历史值替换。
- 回放数据集必须冻结清单、文件 hash、交易日历、证券主数据和修订版本。Qlib 可承载这种离线格式和回测，但不能替代行情采购。
- 在采购或实现前先做小规模验收：沪主板、科创板、深主板、创业板、北交所各取正常交易、停牌、除权、零成交和退市样本，比较至少 20 个交易日。
- SLA、用途权利、北交所、修订通知和实时延迟没有书面答案前，不应把任何候选标为“已满足生产要求”。

## 未解决事实

1. miniQMT/xtdata 对沪深北股票 1 分钟历史的实际最早日期，以及不同券商、普通/VIP 行情站点是否一致。
2. miniQMT 北交所全推 tick 和全市场 1 分钟 K 线订阅的正式支持范围。
3. Tushare、JQData、RQData 的北交所分钟起始日、退市股票保留、当日修订时间和 bar-final 语义。
4. Tushare “仅供研究学习、不可商业”与机构 10 倍价格之间，内部公司投研/交易的合同许可边界。
5. JQData/RQData 当前套餐价格、全市场批量额度、并发连接、实时延迟、正式 SLA 和数据修订政策。
6. Wind 的具体 A 股字段字典、沪深北起始日、复权算法、价格、延迟分位数、历史替换文件与 SLA。
7. 深交所、北交所官方历史 1 分钟产品的字段、起始日、价格、交付和修订机制；本次只能稳定核实上交所 CIIS 产品页。
8. 所有候选的成交量单位、集合竞价归属、停牌填充、价格精度和公司行动生效时点仍需同日交叉样本验证。

## 资料可靠性说明

- 优先使用供应商官方文档、官方 GitHub 固定提交、交易所授权机构页面和许可证文本。
- 厂商页面中的“低延迟”“稳定”“5ms”“24 小时不中断”等属于厂商陈述，除非合同给出测量方法、赔偿和可用性目标，否则不按 SLA 处理。
- 未采用博客、论坛报价或第三方转载作为结论证据。需要登录、销售提供或页面无法稳定访问的事实均标为未核实。
