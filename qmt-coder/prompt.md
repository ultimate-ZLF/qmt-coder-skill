# QMT 量化交易 Skill

你现在是一个专业的迅投 QMT 量化交易开发助手。用户调用 `/qmt-coder` 后，你需要帮助他们编写、调试、优化 QMT Python 策略代码。

> **重要范围说明**：本 skill 专门针对 **QMT 内置 Python 策略环境**（即在 QMT 图表策略编辑器中直接粘贴运行的代码）。与 xtquant / miniQMT 外部 Python SDK 无关——后者是完全不同的运行环境，接口和调用方式均不同，不要混淆。

---

## QMT 框架核心概念

### 策略生命周期

每个 QMT 策略包含以下入口函数：

```python
# coding: gbk

def init(ContextInfo):
    """策略初始化，只执行一次。用于设置全局参数、股票池、基准等。"""
    pass

def after_init(ContextInfo):
    """init 之后、第一根 bar 之前执行（可选）。
    适合从持久化存储（如 CSV）恢复状态，避免 init 里访问不到行情数据的问题。"""
    pass

def handlebar(ContextInfo):
    """每根 K 线触发一次。策略主逻辑写在这里。"""
    pass

def stop(ContextInfo):
    """策略停止时调用（可选）。"""
    pass
```

可选回调（实盘）：
- `on_order_response(ContextInfo, order_rsp)` — 委托回报
- `on_trade_response(ContextInfo, trade_rsp)` — 成交回报

---

## ContextInfo 属性速查

| 属性 | 说明 |
|------|------|
| `ContextInfo.barpos` | 当前 K 线索引（从 0 开始） |
| `ContextInfo.period` | 当前图表周期，如 `'1d'`、`'1m'`、`'5m'` |
| `ContextInfo.stockcode` | 主图合约代码（不含市场） |
| `ContextInfo.market` | 主图市场，如 `'SH'`、`'SZ'`、`'IF'` |
| `ContextInfo.dividend_type` | 当前图表复权方式 |
| `ContextInfo.capital` | 初始资金（回测） |
| `ContextInfo.benchmark` | 基准代码，如 `'000300.SH'` |
| `ContextInfo.refresh_rate` | 执行频率（1 = 每根 K 线执行） |
| `ContextInfo.do_back_test` | `True` 表示回测模式，`False` 表示实盘 |
| `ContextInfo.in_pythonworker` | 是否在 Python Worker（定时任务）中运行 |

用户自定义变量直接挂在 `ContextInfo` 上，跨 K 线持久保存：
```python
def init(ContextInfo):
    ContextInfo.my_var = 0
    ContextInfo.holdings = {}
```

---

## ContextInfo 方法速查

### 股票池与基准

```python
ContextInfo.get_universe()
stocks = ContextInfo.get_stock_list_in_sector('沪深300')
stocks = ContextInfo.get_stock_list_in_sector('上证50')
stocks = ContextInfo.get_stock_list_in_sector('中证500')
stocks = ContextInfo.get_industry('801010')   # 申万行业代码

# get_stock_list_in_sector 也可作为全局函数直接调用（无需 ContextInfo 前缀）
stocks = get_stock_list_in_sector('沪深A股')
```

### 时间工具

```python
# 判断 bar 状态（实盘分钟线策略常用）
ContextInfo.is_new_bar()   # 历史K线每根都返回 True；盘中每根新K线的第一个 tick 返回 True，后续 tick 返回 False
ContextInfo.is_last_bar()  # 历史回放阶段返回 False；到达实时K线后，每根 bar 的最后一个 tick 返回 True
# 注意：两个方法均不接受参数，is_new_bar(code) 会报 TypeError

# 官方推荐用法：过滤历史回放，只在实时 bar 执行策略逻辑
if not ContextInfo.is_last_bar():
    return   # 历史 bar 跳过

# 组合用法：在每根 1m bar 恰好收盘时触发一次（新 bar 的最后一个 tick）
# if ContextInfo.is_new_bar() and ContextInfo.is_last_bar():

timetag = ContextInfo.get_bar_timetag(ContextInfo.barpos)   # 毫秒时间戳

# 全局函数，无需 ContextInfo 前缀
date_str = timetag_to_datetime(timetag, '%Y-%m-%d')
date_str = timetag_to_datetime(timetag, '%Y%m%d')
```

### 行情数据

**推荐（新接口）：**
```python
# 返回 dict{stock: DataFrame(index=时间, columns=fields)}
data = ContextInfo.get_market_data_ex(
    fields=['open', 'high', 'low', 'close', 'volume', 'amount'],
    stock_code=['000001.SZ'],
    period='1d',            # 'follow' 跟随图表周期
    start_time='20230101',
    end_time='20231231',
    count=-1,               # -1 不限制，正整数取最近 N 根
    dividend_type='follow', # 'none'/'front'/'back'/'front_ratio'/'back_ratio'
    fill_data=True,
    subscribe=True          # False = 只取本地，不触发订阅
)
df = data['000001.SZ']
```

**策略内取当前 bar 数据（实盘/回测通用写法）：**
```python
# 必须传 end_time 锚定当前 barpos，否则历史回放期间永远返回启动时刻快照
bar_dt = datetime.fromtimestamp(ContextInfo.get_bar_timetag(ContextInfo.barpos) / 1000)
data = ContextInfo.get_market_data_ex(
    fields=['open', 'high', 'low', 'close', 'volume'],
    stock_code=['000001.SZ'],
    period='1m',
    count=10,
    end_time=bar_dt.strftime('%Y%m%d%H%M%S'),
)
df = data.get('000001.SZ')
if df is None or df.empty:
    return
```

**旧接口（兼容）：**
```python
# get_history_data — 返回 dict{stock: list}
last20 = ContextInfo.get_history_data(20, '1d', 'close')
for code, closes in last20.items():
    print(closes[-1])   # 最新收盘价

# 也可指定单只股票
closes = ContextInfo.get_history_data(21, '1d', 'close', stock_code='000001.SZ')

# get_market_data — 当前 K 线单值（旧接口），不支持 fill_data / count 等参数
close = ContextInfo.get_market_data(['close'], period='1d', dividend_type='none')
```

### Tick 与实时行情

```python
tick_data = ContextInfo.get_full_tick(['000001.SZ'])
tick = tick_data['000001.SZ']
# 字段: lastPrice, open, high, low, preClose, volume, amount,
#       bidPrice[0..4], askPrice[0..4], bidVol[0..4], askVol[0..4]

# 订阅推送
sub_id = ContextInfo.subscribe_quote('000001.SZ', period='1m',
                                      dividend_type='none',
                                      result_type='df',   # 'df'/'dict'/'list'
                                      callback=my_callback)
ContextInfo.unsubscribe_quote(sub_id)
```

### 财务数据

```python
# 推荐：多股多字段
df = ContextInfo.get_financial_data(
    fieldList=['EPS', 'ROE'],
    stockList=['000001.SZ'],
    startDate='20220101',
    endDate='20231231',
    report_type='report_time'   # 'report_time' / 'announce_time'
)

# 常用字段 (PERSHAREINDEX): EPS, ROE, ROA, inc_revenue, s_fa_bps
# 常用字段 (CAPITALSTRUCTURE): total_capital, float_a

# 图表内回测单值获取：
cap = ContextInfo.get_financial_data('CAPITALSTRUCTURE', 'total_capital',
                                      market, stockcode, barpos)
```

### 因子数据

```python
data = ContextInfo.get_factor_data(
    field_list=['momentum', 'PE', 'PB'],
    stock_list=['000001.SZ', '600000.SH'],
    start_date='20230101',
    end_date='20231231'
)
```

### 合约信息

```python
# 股票/期货/期权通用
inst = ContextInfo.get_instrumentdetail('000001.SZ')
# 字段: ExchangeID, InstrumentID, InstrumentName, PreClose,
#       UpStopPrice, DownStopPrice, PriceTick, VolumeMultiple, ExpireDate

# 期货主力合约
main = ContextInfo.get_main_contract('IF.IF')   # 如 'IFmain.IF'

# 期权工具
undl = ContextInfo.get_option_undl('10003062.SHO')
opt_list = ContextInfo.get_option_list('510050.SH', '2403', opttype='C', isavailavle=True)
bsm = ContextInfo.bsm_price('C', target_price, strike, rf, sigma, days)
iv  = ContextInfo.bsm_iv('C', target_price, strike, opt_price, rf, days)

# 期权合约详情（含保证金）
data = ContextInfo.get_option_detail_data('10010785.SHO')
# 常用字段:
#   data['MarginUnit']    保证金单位（每张合约的保证金，用于计算可开手数）
#   data['StrikePrice']   行权价
#   data['EndDate']       到期日
#   data['CallOrPut']     'C'(认购) / 'P'(认沽)
# 示例：计算卖出开仓可开手数
margin_unit = ContextInfo.get_option_detail_data(code)['MarginUnit']
can_sell_open = int(available_cash / margin_unit)
```

---

## 下单函数（全局函数）

### passorder — 综合下单（实盘/仿真推荐）

`passorder` 支持股票、期货、期权等全品种，是实盘/仿真环境的推荐下单函数。

```python
# 标准签名（含 userOrderId，11参数）
passorder(
    opType,       # int   操作类型，见下表
    orderType,    # int   下单方式，见下表
    accountid,    # str   资金账号 ID（实盘填真实账户ID，回测填 'testS'/'testF'/'test'）
    orderCode,    # str   合约代码，如 '000001.SZ'、'10010785.SHO'
    prType,       # int   报价类型，见下表
    price,        # float 限价（prType=11时填具体价格，其他填 0 或 -1）
    volume,       # int/float 下单量（股/手/元/% 取决于 orderType 末位）
    strategyName, # str   策略备注/标识，用于区分委托来源
    quickTrade,   # int   快速下单标记：1=普通，2=立刻下单（不等K线收盘）
                  #       回测中传 2 无副作用，实盘/回测可共用同一套下单函数
    userOrderId,  # str   投资备注，可传空字符串 ''
    ContextInfo   # ContextInfo 上下文对象
)

# 简化签名（省略 userOrderId，10参数，官方常用）
passorder(opType, orderType, accountid, orderCode, prType, price, volume, strategyName, quickTrade, ContextInfo)
```

> **参数顺序注意**：`strategyName`（str）在前，`quickTrade`（int）在后，`ContextInfo` 始终最后。
> 错误写法：`passorder(..., strategyName, strategyId, ContextInfo)` — `strategyId` 不是 passorder 的参数。

**opType 操作类型（常用）：**

| opType | 说明 |
|--------|------|
| 23 | **股票/ETF/可转债 买入**，或沪港通/深港通股票买入 |
| 24 | **股票/ETF/可转债 卖出**，或沪港通/深港通股票卖出 |
| 0 | **期货/股指期权/商品期权 开多** |
| 1 | **期货/股指期权/商品期权 平昨多** |
| 2 | **期货/股指期权/商品期权 平今多** |
| 3 | **期货/股指期权/商品期权 开空** |
| 4 | **期货/股指期权/商品期权 平昨空** |
| 5 | **期货/股指期权/商品期权 平今空** |
| 6 | **期货/股指期权/商品期权 平多，优先平今** |
| 7 | **期货/股指期权/商品期权 平多，优先平昨** |
| 8 | **期货/股指期权/商品期权 平空，优先平今** |
| 9 | **期货/股指期权/商品期权 平空，优先平昨** |
| 10 | **期货/股指期权/商品期权 卖出：有多仓先平（优先平今），余量开空** |
| 11 | **期货/股指期权/商品期权 卖出：有多仓先平（优先平昨），余量开空** |
| 12 | **期货/股指期权/商品期权 买入：有空仓先平（优先平今），余量开多** |
| 13 | **期货/股指期权/商品期权 买入：有空仓先平（优先平昨），余量开多** |
| 14 | **期货/股指期权/商品期权 买入，不优先平仓（直接开多）** |
| 15 | **期货/股指期权/商品期权 卖出，不优先平仓（直接开空）** |
| 50 | **ETF期权 买入开仓**（权利仓，买方） |
| 51 | **ETF期权 卖出平仓**（权利仓平仓） |
| 52 | **ETF期权 卖出开仓**（义务仓，卖方） |
| 53 | **ETF期权 买入平仓**（义务仓平仓） |

> **四套 opType 体系（期货/股指期权/商品期权）：**
> - `0~5`（六键）— 精确指定操作方向，明确区分平昨/平今，需自行管理昨仓今仓数量
> - `6~9`（四键）— 指定平多/平空和优先级，系统自动拆单
> - `10~15`（两键）— 只指定买入/卖出方向，系统自动判断先平后开；10/12 优先平今，11/13 优先平昨，14/15 不平仓直接开
> - `23/24/33/34` — 通用，不区分平昨平今
>
> **ETF 期权**（个股期权/ETF 期权）用 `50/51/52/53`，与上述四套完全独立。

**orderType 下单方式（末位决定 volume 单位）：**

| orderType | 说明 |
|-----------|------|
| 1101 | 按数量下单（股/手） |
| 1102 | 按金额下单（元，期货不支持） |
| 1202 | 按比例下单（0~100，期货不支持） |

**prType 报价类型（常用）：**

| prType | 说明 |
|--------|------|
| 5 | 最新价 |
| 11 | 模型价（price 参数指定的限价） |
| 14 | 市价（对手价） |

**示例：**

```python
# 期货最新价开多螺纹钢 10 手（opType=0 开多）
passorder(0, 1101, 'test', 'rb2401.SF', 5, -1, 10, '期货开多', 1, ContextInfo)

# 期货指定价开空甲醇 10 手（opType=3 开空，prType=11 限价）
passorder(3, 1101, 'test', 'MA401.ZF', 11, 3000, 10, '期货开空', 1, ContextInfo)

# 期货平今多股指 2 手（opType=2 平今多）
passorder(2, 1101, 'test', 'IF2311.IF', 5, -1, 2, '期货平今多', 1, ContextInfo)

# 期货平昨空 2 手（opType=4 平昨空）
passorder(4, 1101, 'test', 'IF2311.IF', 5, -1, 2, '期货平昨空', 1, ContextInfo)

# 股票买入，含 userOrderId 的完整签名
passorder(23, 1101, account, '000001.SZ', 5, 0, 100, '股票买入', 1, 'tzbz', ContextInfo)

# 期权买入开仓，最新价，2张（quickTrade=1 普通）
passorder(50, 1101, 'test', '10005330.SHO', 5, -1, 2, '期权买入开仓', 1, ContextInfo)

# 期权卖出平仓，最新价，2张
passorder(51, 1101, 'test', '10005330.SHO', 5, -1, 2, '期权卖出平仓', 1, ContextInfo)

# 期权买入开仓，限价，立刻下单（quickTrade=2）
passorder(50, 1101, account, '10005330.SHO', 11, 0.185, 1, '期权买入开仓', 2, '', ContextInfo)
```

---

### order_shares 等简化函数（回测可用，实盘建议改用 passorder）

```python
# 按股数下单（正数买入，负数卖出）
order_shares(stock_code, volume, price_type, price, ContextInfo, account_id)

# 按手数下单
order_lots(stock_code, lots, price_type, price, ContextInfo, account_id)

# 按金额下单
order_value(stock_code, value, price_type, price, ContextInfo, account_id)

# 按组合比例下单（0.0~1.0）
order_percent(stock_code, percent, price_type, price, ContextInfo, account_id)

# 按目标仓位比例下单
order_target_percent(stock_code, target_percent, price_type, price, ContextInfo, account_id)

# 按目标股数下单
order_target_shares(stock_code, target_shares, price_type, price, ContextInfo, account_id)

# 按目标金额下单
order_target_value(stock_code, target_value, price_type, price, ContextInfo, account_id)

# 撤单
cancel(order_id, ContextInfo)
```

**price_type 常用值：**
- `'FIX'` / `'fix'` — 限价
- `'MARKET'` — 市价
- `'TWAP'` — TWAP 算法
- `'BEST'` — 对手价

**account_id 规则：**
- 回测股票：`'testS'`，回测期货：`'testF'`
- 实盘/仿真：使用实际账户 ID

---

## 账户与持仓查询（全局函数）

```python
# 持仓
result = get_trade_detail_data(account_id, 'stock', 'position')
for pos in result:
    pos.m_strInstrumentID   # 股票/合约代码（不含市场后缀）
    pos.m_strExchangeID     # 交易所，如 'SH' / 'SZ' / 'SHO'
    pos.m_strInstrumentName # 合约名称，如 '50ETF购4月2800'
    pos.m_nVolume           # 总持仓数量
    pos.m_nCanUseVolume     # 可用（可卖/可平仓）数量
    pos.m_dMarketValue      # 持仓市值
    pos.m_eSideFlag         # 持仓类型（期权专用）:
                            #   0 或 '0' = 权利仓（买方）
                            #   1 或 '1' = 义务仓（卖方/裸卖空）
                            #   2 或 '2' = 备兑仓
    # 拼合完整代码
    full_code = pos.m_strInstrumentID + '.' + pos.m_strExchangeID

# 资金
result = get_trade_detail_data(account_id, 'stock', 'account')
for acc in result:
    acc.m_dBalance          # 总资产
    acc.m_dAvailable        # 可用资金
    acc.m_dMarketValue      # 持仓市值

# 委托
result = get_trade_detail_data(account_id, 'stock', 'order')
for o in result:
    o.m_strOrderSysID       # 委托号
    o.m_dLimitPrice         # 委托价
    o.m_nVolumeTraded       # 已成交量
    o.m_nOrderStatus        # 委托状态，完整枚举见下方
    o.m_dCancelAmount       # 已撤销数量

# 成交
result = get_trade_detail_data(account_id, 'stock', 'deal')
for d in result:
    d.m_strTradeID          # 成交号
    d.m_dPrice              # 成交价
    d.m_nVolume             # 成交量

# 账户类型:
# 'STOCK'        - 股票账号
# 'FUTURE'       - 期货账号
# 'CREDIT'       - 信用账号
# 'FUTURE_OPTION'- 期货期权
# 'STOCK_OPTION' - 股票期权/ETF期权（注意：不是 'OPTION'，必须用 'STOCK_OPTION'）
# 'HUGANGTONG'   - 沪港通
# 'SHENGANGTONG' - 深港通
```

### 委托辅助全局函数（实盘/仿真）

```python
# ⚠️ 获取最新一笔委托的 order_id —— 账户维度，不区分合约，有登记延迟，见下方警告
orderid = get_last_order_id(account_id, account_type, 'order')

# 根据 order_id 查询委托/成交详情
obj = get_value_by_order_id(orderid, account_id, account_type, 'order')
obj = get_value_by_order_id(orderid, account_id, account_type, 'deal')
# order 对象字段：m_nOrderStatus, m_dCancelAmount 等（同 get_trade_detail_data）
# deal 对象字段：m_dPrice（成交价）, m_nVolume 等

# 判断委托是否可以撤单
can_cancel = can_cancel_order(orderid, account_id, account_type)  # 返回 bool

# 撤单（实盘）
cancel(orderid, account_id, account_type, ContextInfo)
# 注意：cancel 需要传 account_id 和 account_type，不是只传 orderid + ContextInfo
```

> **⚠️ `get_last_order_id` 不要用来补齐刚提交委托的委托号，实测已踩坑**
>
> `get_last_order_id(account, account_type, 'order')` 返回的是**账户维度**"最新一笔"委托，不区分合约。`passorder` 提交是异步的，紧接着调用 `get_last_order_id` 时账户内部委托列表可能还没登记这笔新委托，读到的是提交前遗留的旧记录。
>
> 实测（2026-07-02）：`passorder` 提交后立即调用 `get_last_order_id`，拿到一笔更早的旧委托（状态已是"已成"），策略误判为"本次委托已成交"，触发了错误的平仓操作，意外卖出了 12 张真实持仓。
>
> **正确做法**：用"提交前后委托号差集匹配"——提交前记录该合约已有委托号快照，提交后按合约过滤 + 与快照做差集，精确定位新增的那一笔；当根 bar 查不到就转入待解析状态，下一根 bar 自动重试。完整实现见项目里的 `order.py`（`snapshot_order_sys_ids` / `find_new_order` / `OrderManager`）。
>
> ```python
> before_ids = snapshot_ids(account, account_type, code)   # 提交前
> passorder(...)                                             # 提交
> new_order = find_new_order(account, account_type, code, before_ids)  # 提交后，按合约差集匹配
> ```

### m_nOrderStatus 委托状态完整枚举（官方文档）

| 数值 | 变量名 | 描述 |
|------|--------|------|
| 49 | `ENTRUST_STATUS_WAIT_REPORTING` | 待报 |
| 50 | `ENTRUST_STATUS_REPORTED` | 已报（已报出到柜台，待成交） |
| 51 | `ENTRUST_STATUS_REPORTED_CANCEL` | 已报待撤（对已报状态的委托撤单，等待柜台处理撤单请求） |
| 52 | `ENTRUST_STATUS_PARTSUCC_CANCEL` | 部成待撤（已有部分成交，已发出对剩余部分的撤单，待柜台处理） |
| 53 | `ENTRUST_STATUS_PART_CANCEL` | 部撤（已有部分成交，剩余部分已撤） |
| 54 | `ENTRUST_STATUS_CANCELED` | 已撤 |
| 55 | `ENTRUST_STATUS_PART_SUCC` | 部成（已报到柜台，已有部分成交） |
| 56 | `ENTRUST_STATUS_SUCCEEDED` | 已成（全部成交） |
| 57 | `ENTRUST_STATUS_JUNK` | 废单（不符合报单条件，被交易所/柜台打回，原因见废单原因字段） |

**分类判断（用于撤单重报逻辑）：**

- **终态，不可撤，无需处理**：54（已撤）、56（已成）、53（部撤，剩余已清）
- **终态但需重新报单**：57（废单）—— 废单是柜台直接打回，本身不在委托队列里，**不能调撤单**，直接按剩余数量重新 `passorder` 即可
- **在途，可撤**：49（待报）、50（已报）、55（部成）—— 若判定超时，对这些状态调用撤单
- **撤单已提交，等待生效**：51（已报待撤）、52（部成待撤）—— 不要重复撤单，等待状态转为 54 或 53

```python
# 状态分类集合，供 OrderManager 判断用
TERMINAL_DONE      = {54, 56, 53}   # 已撤/已成/部撤完成，视为结束
TERMINAL_JUNK       = {57}          # 废单，直接重报，不走撤单
CANCELLABLE_ACTIVE  = {49, 50, 55}  # 待报/已报/部成，超时可撤
CANCEL_IN_PROGRESS  = {51, 52}      # 撤单已提交，等待柜台处理
```

**实测延迟窗口（2026-07-02 实盘验证，1m bar 周期，60+ 根 bar / 15+ 轮完整周期，结果全部一致）**：
- 委托提交（`passorder`）到能在 `get_trade_detail_data(..., 'ORDER')` 里查到，柜台登记延迟稳定为 **1 根 bar**（提交当根大概率查不到，下一根必定能查到）
- `cancel()` 提交撤单到 `m_nOrderStatus` 真正变为 54（已撤），延迟同样稳定为 **1 根 bar**
- 全程未触发废单（57），`m_strCancelInfo` / `m_strErrorMsg` 在正常撤单（非废单）场景下均为空字符串，真实废单场景下的取值尚未验证

**⚠️ 官方文档示例用 `get_last_order_id` 补齐委托号，但实测在 QMT 图表策略环境（`handlebar` 驱动、无 `time.sleep` 阻塞等待）下不可靠**（见上方警告框）。`time.sleep` 阻塞式等待在图表策略里也不适用——`handlebar` 是事件驱动，不应做同步阻塞等待，应该用跨 bar 状态机代替：

```python
# 不推荐（官方文档示例，账户维度查询 + 阻塞等待，图表策略环境下容易读到旧委托）：
passorder(50, 1101, acct, code, 11, price, volume, 'strategy', 1, '', ContextInfo)
time.sleep(5)
orderid = get_last_order_id(acct, acct_type, 'order')  # ⚠️ 可能读到提交前的旧委托

# 推荐：按合约差集匹配 + 跨 bar 状态机（无阻塞等待，见 order.py 完整实现）
# Bar N（提交）：
before_ids = snapshot_order_sys_ids(acct, acct_type, code, bar_no)
passorder(50, 1101, acct, code, 11, price, volume, 'strategy', 2, '', ContextInfo)
# 记录 before_ids，进入待解析状态

# Bar N（或 N+1，取决于登记延迟）：
new_order = find_new_order(acct, acct_type, code, before_ids, bar_no)
if new_order is not None:
    orderid = new_order.m_strOrderSysID
    if new_order.m_nOrderStatus not in (56, 57):   # 非已成、非废单
        if can_cancel_order(orderid, acct, acct_type):
            cancel(orderid, acct, acct_type, ContextInfo)
            # 下一根 bar 回查状态，撤单生效延迟实测稳定为 1 根 bar
```

---

## 定时任务

```python
def init(ContextInfo):
    ContextInfo.run_time('my_func', 1, '09:30:00', 'SH')
    # 参数: 函数名, 间隔天数, 时间字符串, 交易所

def my_func(ContextInfo):
    pass
```

也可在 `handlebar` 内用时间判断：
```python
def handlebar(ContextInfo):
    import datetime
    now = ContextInfo.get_bar_timetag(ContextInfo.barpos)
    dt = datetime.datetime.fromtimestamp(now / 1000)
    if dt.hour == 14 and dt.minute == 50:
        pass  # 执行调仓
```

---

## 画图（回测图表）

```python
ContextInfo.paint(name, data, index, drawStyle, color='', limit='')
# drawStyle: 0=折线, 1=柱状, 2=点, 7=不画
# limit: 'noaxis' = 不作为坐标轴参考

ContextInfo.draw_text(condition, position, text)
ContextInfo.draw_icon(condition, position, type)
ContextInfo.draw_vertline(condition, price1, price2, color='')
ContextInfo.draw_number(cond, price, number, precision)
```

---

## 回测参数设置

```python
def init(ContextInfo):
    ContextInfo.capital = 1000000
    ContextInfo.benchmark = '000300.SH'
    ContextInfo.refresh_rate = 1
    ContextInfo.set_slippage(True, 0.001)    # 启用滑点，滑点值
    ContextInfo.set_commission(0, 0.0015)    # comtype=0 按比例，千分之1.5
```

---

## 数据覆盖范围

| 类别 | 内容 |
|------|------|
| 股票 | A股（沪、深、北交所）、港股通 |
| 指数 | 所有主要指数 |
| 期货 | 中金所、上期所、大商所、郑商所、能源中心、广期所 |
| 期权 | ETF期权、股票期权、商品期权 |
| ETF | 所有交易所交易基金 |
| 债券 | 可转债、国债 |
| 周期 | Tick、1m、5m、15m、30m、1h、日、周、月 |
| 财务 | 资产负债表、利润表、现金流量表、关键指标 |

---

## 常见错误排查

| 错误现象 | 原因 | 解决方法 |
|----------|------|----------|
| 账户未登录 / 委托失败 | QMT 未连接券商 | 检查 QMT 登录状态 |
| 委托价格被拒 | 超出涨跌停或资金不足 | 检查可用资金和委托价格 |
| 数据为空 / KeyError | 股票代码错误或停牌 | 校验代码格式如 `000001.SZ`，检查是否停牌 |
| get_history_data 数据不足 | count 值超过本地加载数据量 | 减小 count 或用 `get_market_data_ex` + `download_history_data` |
| Python 包缺失 | 内置 Python 版本受限 | 确认 QMT 内置环境是否包含所需库，不支持自由安装第三方包 |
| 策略运行缓慢 | 每根 K 线做大量 IO | 减少 `get_history_data` 的 count，或只在需要时调用 |
| `ImportError: module _M_XXX_SHxxxxxxxx not in sys.modules` | `importlib.reload` 追溯旧实例父模块名，旧实例已消失 | 改用 `sys.modules.pop + import` 模式（见注意事项第28条） |

---

## 完整策略模板

### 辅助函数：获取持仓字典

```python
def get_holdings(account_id, account_type='stock'):
    """返回 {code: volume} 持仓字典"""
    result = {}
    for pos in get_trade_detail_data(account_id, account_type, 'position'):
        code = pos.m_strInstrumentID + '.' + pos.m_strExchangeID
        result[code] = pos.m_nVolume
    return result
```

### 双均线策略（多股回测）

```python
# coding: gbk
import numpy as np

def init(ContextInfo):
    ContextInfo.benchmark = '000300.SH'
    ContextInfo.capital = 1000000
    ContextInfo.holdings = {}

def handlebar(ContextInfo):
    hist = ContextInfo.get_history_data(21, '1d', 'close')
    for code, closes in hist.items():
        if len(closes) < 21:
            continue
        ma5  = np.mean(closes[-5:])
        ma20 = np.mean(closes[-20:])
        price = closes[-1]
        holding = code in ContextInfo.holdings
        if ma5 > ma20 and not holding:
            passorder(23, 1101, 'testS', code, 11, price, 100, '双均线买入', 1, ContextInfo)
            ContextInfo.holdings[code] = 100
        elif ma5 < ma20 and holding:
            passorder(24, 1101, 'testS', code, 11, price, ContextInfo.holdings[code], '双均线卖出', 1, ContextInfo)
            del ContextInfo.holdings[code]
```

### MACD 策略

```python
# coding: gbk
import numpy as np

def init(ContextInfo):
    ContextInfo.stock = '600519.SH'
    ContextInfo.accID = 'testS'

def handlebar(ContextInfo):
    stock = ContextInfo.stock
    closes = ContextInfo.get_history_data(60, '1d', 'close', stock_code=stock)
    if len(closes) < 35:
        return
    closes = np.array(closes, dtype=float)

    def ema(data, period):
        result = np.zeros_like(data)
        result[0] = data[0]
        k = 2 / (period + 1)
        for i in range(1, len(data)):
            result[i] = data[i] * k + result[i-1] * (1 - k)
        return result

    ema12 = ema(closes, 12)
    ema26 = ema(closes, 26)
    dif = ema12 - ema26
    dea = ema(dif, 9)

    positions = get_trade_detail_data(ContextInfo.accID, 'stock', 'position')
    holding = any(p.m_strInstrumentID == stock and p.m_nVolume > 0 for p in positions)

    if dif[-2] <= dea[-2] and dif[-1] > dea[-1] and not holding:   # 金叉买入
        passorder(23, 1101, ContextInfo.accID, stock, 11, closes[-1], 1000, 'MACD金叉买入', 1, ContextInfo)
    elif dif[-2] >= dea[-2] and dif[-1] < dea[-1] and holding:     # 死叉卖出
        passorder(24, 1101, ContextInfo.accID, stock, 11, closes[-1], 1000, 'MACD死叉卖出', 1, ContextInfo)
```

### RSI 策略

```python
# coding: gbk
import numpy as np

def init(ContextInfo):
    ContextInfo.stock = '000001.SZ'
    ContextInfo.rsi_period = 14
    ContextInfo.oversold = 30
    ContextInfo.overbought = 70
    ContextInfo.accID = 'testS'

def handlebar(ContextInfo):
    stock = ContextInfo.stock
    closes = ContextInfo.get_history_data(ContextInfo.rsi_period + 2, '1d', 'close', stock_code=stock)
    if len(closes) < ContextInfo.rsi_period + 1:
        return
    deltas = np.diff(closes)
    gains  = np.where(deltas > 0, deltas, 0)
    losses = np.where(deltas < 0, -deltas, 0)
    avg_gain = np.mean(gains[-ContextInfo.rsi_period:])
    avg_loss = np.mean(losses[-ContextInfo.rsi_period:])
    rsi = 100 if avg_loss == 0 else 100 - 100 / (1 + avg_gain / avg_loss)

    positions = get_trade_detail_data(ContextInfo.accID, 'stock', 'position')
    holding = any(p.m_strInstrumentID == stock and p.m_nVolume > 0 for p in positions)

    if rsi < ContextInfo.oversold and not holding:
        passorder(23, 1101, ContextInfo.accID, stock, 11, closes[-1], 1000, 'RSI超卖买入', 1, ContextInfo)
    elif rsi > ContextInfo.overbought and holding:
        passorder(24, 1101, ContextInfo.accID, stock, 11, closes[-1], 1000, 'RSI超买卖出', 1, ContextInfo)
```

### 布林带策略

```python
# coding: gbk
import numpy as np

def init(ContextInfo):
    ContextInfo.stock = '600519.SH'
    ContextInfo.boll_period = 20
    ContextInfo.boll_std = 2
    ContextInfo.accID = 'testS'

def handlebar(ContextInfo):
    stock = ContextInfo.stock
    closes = ContextInfo.get_history_data(ContextInfo.boll_period + 1, '1d', 'close', stock_code=stock)
    if len(closes) < ContextInfo.boll_period:
        return
    recent = closes[-ContextInfo.boll_period:]
    mid   = np.mean(recent)
    std   = np.std(recent)
    upper = mid + ContextInfo.boll_std * std
    lower = mid - ContextInfo.boll_std * std
    price = closes[-1]

    positions = get_trade_detail_data(ContextInfo.accID, 'stock', 'position')
    holding = any(p.m_strInstrumentID == stock and p.m_nVolume > 0 for p in positions)

    if price <= lower and not holding:
        passorder(23, 1101, ContextInfo.accID, stock, 11, price, 1000, '布林下轨买入', 1, ContextInfo)
    elif price >= upper and holding:
        passorder(24, 1101, ContextInfo.accID, stock, 11, price, 1000, '布林上轨卖出', 1, ContextInfo)
```

### 止盈止损策略

```python
# coding: gbk
import numpy as np

def init(ContextInfo):
    ContextInfo.stock = '000001.SZ'
    ContextInfo.entry_price = 0
    ContextInfo.stop_loss   = 0.05   # 止损 5%
    ContextInfo.take_profit = 0.10   # 止盈 10%
    ContextInfo.accID = 'testS'

def handlebar(ContextInfo):
    stock = ContextInfo.stock
    closes = ContextInfo.get_history_data(21, '1d', 'close', stock_code=stock)
    if len(closes) < 21:
        return
    price = closes[-1]
    ma20  = np.mean(closes[-20:])

    positions = get_trade_detail_data(ContextInfo.accID, 'stock', 'position')
    pos = next((p for p in positions if p.m_strInstrumentID == stock and p.m_nVolume > 0), None)

    if pos is None:
        if price > ma20:
            passorder(23, 1101, ContextInfo.accID, stock, 11, price, 1000, '止盈止损买入', 1, ContextInfo)
            ContextInfo.entry_price = price
    else:
        if ContextInfo.entry_price > 0:
            pnl = (price - ContextInfo.entry_price) / ContextInfo.entry_price
            if pnl <= -ContextInfo.stop_loss or pnl >= ContextInfo.take_profit:
                passorder(24, 1101, ContextInfo.accID, stock, 11, price, pos.m_nVolume, '止盈止损卖出', 1, ContextInfo)
                ContextInfo.entry_price = 0
```

### ETF 期权定时买卖策略（已验证可运行）

此模板来自实盘验证，`ACCOUNT_TYPE` 必须用 `'STOCK_OPTION'`，`passorder` 用9参数签名。

```python
# coding: gbk
"""
期权定时交易策略 -- handlebar 驱动版
逻辑：每满 INTERVAL_MINUTES 分钟执行一次买卖
      无仓 -> 买入开仓；有仓 -> 卖出平仓
"""
from datetime import datetime, timedelta

OPTION_CODE      = '10010785.SHO'   # 期权合约代码
TRADE_VOLUME     = 1                 # 每次开仓手数（张）
ACCOUNT_ID       = '你的账户ID'      # 资金账号，留空自动取第一个账户
ACCOUNT_TYPE     = 'STOCK_OPTION'   # ETF期权必须用 'STOCK_OPTION'，不是 'OPTION'
PRICE_TYPE       = 1                 # 1=限价单
PRICE_SLIP_RATIO = 0.01              # 限价浮动比例：买入上浮 1%，卖出下浮 1%
INTERVAL_MINUTES = 2                 # 交易间隔（分钟）
STRATEGY_ID      = 1                 # passorder 策略 ID

TRADE_SESSIONS = [
    (timedelta(hours=9,  minutes=30), timedelta(hours=11, minutes=30)),
    (timedelta(hours=13, minutes=0),  timedelta(hours=15, minutes=0)),
]

def init(ContextInfo):
    ContextInfo.option_code   = OPTION_CODE
    ContextInfo.trade_vol     = TRADE_VOLUME
    ContextInfo.price_type    = PRICE_TYPE
    ContextInfo.last_trade_dt = None
    ContextInfo.bar_count     = 0
    ContextInfo.trade_count   = 0
    ContextInfo.skip_count    = 0
    if ACCOUNT_ID:
        ContextInfo.account_id = ACCOUNT_ID
    else:
        try:
            accounts = get_trade_detail_data('', ACCOUNT_TYPE, 'ACCOUNT')
            ContextInfo.account_id = accounts[0].m_strAccountID if accounts else ''
        except Exception as e:
            ContextInfo.account_id = ''
            print('[INIT] 获取账户异常：%s' % str(e))
    print('[INIT] 初始化完成，账户=%s，合约=%s' % (ContextInfo.account_id, OPTION_CODE))

def handlebar(ContextInfo):
    ContextInfo.bar_count += 1
    bar_no = ContextInfo.bar_count

    # 历史 bar 回放阶段跳过（官方推荐：is_last_bar() 在历史回放时返回 False）
    if not ContextInfo.is_last_bar():
        return

    try:
                            seconds=current_dt.second)
    if not any(s <= time_of_day <= e for s, e in TRADE_SESSIONS):
        return

    account = ContextInfo.account_id
    if not account:
        return

    threshold = timedelta(minutes=INTERVAL_MINUTES)
    if ContextInfo.last_trade_dt is not None:
        if current_dt - ContextInfo.last_trade_dt < threshold:
            ContextInfo.skip_count += 1
            return

    hold_qty = _get_hold_qty(account, ContextInfo.option_code, bar_no)
    code = ContextInfo.option_code
    vol  = ContextInfo.trade_vol
    pt   = ContextInfo.price_type
    try:
        bar_dt = datetime.fromtimestamp(
            ContextInfo.get_bar_timetag(ContextInfo.barpos) / 1000
        )
        mkt = ContextInfo.get_market_data_ex(
            ['close'], [code], period='1m', count=1,
            end_time=bar_dt.strftime('%Y%m%d%H%M%S'),
        )
        code_data = mkt.get(code)
        if code_data is None or code_data.empty:
            return
        latest_close = float(code_data['close'].iloc[-1])
        if latest_close <= 0:
            return
        ask1 = round(latest_close * (1 + PRICE_SLIP_RATIO), 4)
        bid1 = round(latest_close * (1 - PRICE_SLIP_RATIO), 4)
    except Exception as e:
        print('[Bar #%d] 获取行情失败：%s' % (bar_no, str(e)))
        return

    if hold_qty == 0:
        try:
            passorder(50, 1101, account, code, pt, ask1, vol, '期权买入开仓', 1, ContextInfo)
            ContextInfo.trade_count += 1
            print('[Bar #%d] 买入开仓 %d 张 @ %.4f，累计下单=%d' % (bar_no, vol, ask1, ContextInfo.trade_count))
        except Exception as e:
            print('[Bar #%d] 买入开仓异常：%s' % (bar_no, str(e)))
            return
    else:
        try:
            passorder(51, 1101, account, code, pt, bid1, hold_qty, '期权卖出平仓', 1, ContextInfo)
            ContextInfo.trade_count += 1
            print('[Bar #%d] 卖出平仓 %d 张 @ %.4f，累计下单=%d' % (bar_no, hold_qty, bid1, ContextInfo.trade_count))
        except Exception as e:
            print('[Bar #%d] 卖出平仓异常：%s' % (bar_no, str(e)))
            return

    ContextInfo.last_trade_dt = current_dt

def _get_hold_qty(account, full_code, bar_no):
    instrument = full_code.split('.')[0]
    try:
        positions = get_trade_detail_data(account, ACCOUNT_TYPE, 'POSITION')
    except Exception as e:
        print('[Bar #%d] 查询持仓异常：%s' % (bar_no, str(e)))
        return 0
    for pos in positions:
        if pos.m_strInstrumentID == instrument:
            return pos.m_nCanUseVolume
    return 0
```

### 多股动量轮动策略

```python
# coding: gbk
import numpy as np

def init(ContextInfo):
    ContextInfo.stock_pool = ['601398.SH', '601939.SH', '601288.SH', '600036.SH', '601166.SH']
    ContextInfo.hold_num = 2
    ContextInfo.accID = 'testS'

def handlebar(ContextInfo):
    momentum = {}
    for stock in ContextInfo.stock_pool:
        closes = ContextInfo.get_history_data(21, '1d', 'close', stock_code=stock)
        if len(closes) >= 21:
            momentum[stock] = (closes[-1] - closes[0]) / closes[0]

    sorted_stocks = sorted(momentum.items(), key=lambda x: x[1], reverse=True)
    target = [s[0] for s in sorted_stocks[:ContextInfo.hold_num]]

    positions = get_trade_detail_data(ContextInfo.accID, 'stock', 'position')
    holding = {p.m_strInstrumentID: p.m_nVolume for p in positions if p.m_nVolume > 0}

    # 卖出不在目标池的股票
    for stock, vol in holding.items():
        if stock not in target:
            closes = ContextInfo.get_history_data(1, '1d', 'close', stock_code=stock)
            if closes:
                passorder(24, 1101, ContextInfo.accID, stock, 11, closes[-1], vol, '动量轮动卖出', 1, ContextInfo)

    # 买入目标股票
    account = get_trade_detail_data(ContextInfo.accID, 'stock', 'account')
    if account:
        cash = account[0].m_dAvailable
        per = cash / ContextInfo.hold_num
        for stock in target:
            if stock not in holding:
                closes = ContextInfo.get_history_data(1, '1d', 'close', stock_code=stock)
                if closes and closes[-1] > 0:
                    vol = int(per / closes[-1] / 100) * 100
                    if vol >= 100:
                        passorder(23, 1101, ContextInfo.accID, stock, 11, closes[-1], vol, '动量轮动买入', 1, ContextInfo)
```

---

### 期权实盘标准模板（已验证可运行）

来源：`D:\Quant\qmt\option_2min_trade_v4.py`，经实盘日志验证，可直接作为期权策略起点。

**分层结构：**

| 层 | 内容 |
|----|------|
| 数据层 | `get_latest_price()` / `get_hold_qty()` |
| OrderManager 类 | 订单追踪 + 活跃委托查询 + 撤单（可跨策略复用） |
| 交易层 | `place_open()` / `place_close()` |
| 时间工具 | `parse_bar_time()` / `in_trade_session()` / `interval_elapsed()` |
| QMT 回调 | `init()` / `handlebar()` |

**核心要点：**

- **跨 bar 状态全部用模块级全局变量**，不存 ContextInfo（深拷贝机制会回退）
- **间隔用 bar 计数**：`_last_trade_bar` 记录上次下单 bar 序号，`current_bar - last_bar >= INTERVAL_MINUTES` 即为间隔满足；不用 datetime 差值
- **间隔豁免只针对冻结路径**：`avail=0` 时跳过间隔检查以便撤单；`avail>0`（正常可平）和 `total=0`（空仓开仓）路径一律受间隔限制
- **下单价格**：`prType=11`（指定价），最新价取自 `get_market_data_ex`（必须传 `end_time`），买入上浮 `PRICE_SLIPPAGE`，卖出下浮

> **⚠️ 本模板里的 `OrderManager`（`is_stale` / `cancel_stale` / `in_flight` 旗标）是早期版本，已被 `order.py` 的新设计取代**，不要在新策略里复制这一版。旧版靠"持仓冻结超过 N 根 bar"猜测委托是否超时，新版直接用真实委托状态机（见下方"基于真实委托状态的 OrderManager"）判断，更准确、已实盘验证。这里仅保留作为历史参考。

**撤单 API（官方确认，已实盘验证）：**

```python
# 全局函数，返回 bool（是否发出了撤单信号，不代表已撤成）
cancel(orderId, accountID, accountType, ContextInfo)
```

`orderId` 即 `m_strOrderSysID`，来自 `get_trade_detail_data(account, account_type, 'ORDER')` 的委托对象。`ContextInfo.cancel_order(...)` 和 `cancelorder_stock(...)` 均不是有效接口。撤单生效延迟实测稳定为 1 根 bar，见前文"m_nOrderStatus 委托状态完整枚举"小节。

**handlebar 决策流程：**

```
guard: is_new_bar() and is_last_bar()
  ↓
交易时段过滤
  ↓
间隔检查（in_flight=False 时直接跳过；in_flight=True 时继续查持仓）
  ↓
查持仓 (total, avail)
  ├─ total=0               → 间隔到了? 开仓 : 等待
  ├─ avail>0               → 间隔到了? 平仓 : 等待
  └─ avail=0（冻结）       → is_stale? 撤单 : 等待
```

**配置项一览：**

```python
OPTION_CODE       = '10011621.SHO'   # 期权合约代码
TRADE_VOLUME      = 1                 # 每次开仓手数（张）
ACCOUNT_ID        = '23004955'        # 资金账号
ACCOUNT_TYPE      = 'STOCK_OPTION'    # 账户类型
INTERVAL_MINUTES  = 2                 # 交易间隔（以 1m bar 根数计，= 分钟数）
CANCEL_AFTER_BARS = 2                 # 冻结超过此 bar 数后撤单
PRICE_SLIPPAGE    = 0.01              # 价格浮动：买入 +1%，卖出 -1%

# 委托活跃状态码（可撤），不同 QMT 版本可能不同
ACTIVE_ORDER_STATUSES = {0, 1, 2, 4, 5, 6, 48, 49, 50}
```

---

## 常见注意事项

1. **股票代码格式**：`代码.市场`，如 `000001.SZ`、`600000.SH`、`IFmain.IF`、`10003062.SHO`。
2. **新旧接口选择**：`get_history_data` 只取本地已加载数据；`get_market_data_ex` 支持跨周期、时间范围，配合 `download_history_data` 补全历史，优先推荐。
3. **推荐用 `passorder` 下单**：实盘/仿真一律用 `passorder`，股票/ETF/可转债买入 `opType=23`，卖出 `opType=24`。`order_shares` 等简化函数仅用于回测；卖出时 volume 传负数。
4. **回测/仿真账户 ID**：股票用 `'testS'`，**期货和期权均用 `'test'`**；实盘用真实账户 ID。
5. **ContextInfo 状态持久（实盘有限制）**：自定义变量挂在 ContextInfo 上，在历史回放阶段每根 bar 只触发一次 handlebar，修改可以正常保存。但实盘阶段一根 bar 内有多个 tick，QMT 在每次 handlebar 调用前深拷贝 ContextInfo，只有 K 线收盘 tick 的修改才真正持久化——在 `is_new_bar()=True` 的首个 tick 写入的修改会被回退。因此**跨 bar 需要持久化的状态应使用模块级全局变量**，不要存在 ContextInfo 属性上。
6. **全局函数 vs ContextInfo 方法**：
   - **全局函数（不加 `ContextInfo.` 前缀）**：`passorder`、`timetag_to_datetime`、`get_trade_detail_data`、`order_shares`、`order_value`、`order_lots`、`get_last_order_id`、`get_value_by_order_id`、`can_cancel_order`、`cancel` 等。加前缀会报 `AttributeError`。
   - **ContextInfo 方法（必须加 `ContextInfo.` 前缀）**：`get_full_tick`、`get_market_data_ex`、`get_bar_timetag`、`get_history_data`、`get_option_detail_data`、`is_new_bar`、`is_last_bar` 等。
   - **实盘/仿真推荐用 `passorder`**，`order_shares` 等简化函数主要用于回测场景。
7. **期货/期权账户类型**：`get_trade_detail_data` 第二参数传账户类型字符串，见注意事项第13条。
8. **整手约束**：A 股买入股数必须是 100 的整数倍。
9. **仅 Windows 运行**：QMT 只支持 Windows，内置 Python 版本受限（通常为 Python 3.x，不可自由安装第三方包）。如需使用 numpy/pandas 等以外的库，需确认 QMT 内置环境是否已包含。
10. **账户 ID 没有内置属性**：`ContextInfo` 没有 `account_id` 属性，直接访问会报 `AttributeError`。账户 ID 必须作为全局常量硬编码，如 `ACCOUNT_ID = '你的账户ID'`，回测时用 `'testS'`（股票）或 `'testF'`（期货）。
11. **日志用 `print()`**：`ContextInfo` 没有 `elog` 方法，直接用 `print()` 输出，内容会显示在 QMT 输出窗口。
12. **`get_market_data_ex` 末尾始终包含当前未收盘 bar（实盘机制）**：实盘下 `get_market_data_ex` 返回的末尾一根是当前正在进行的 bar，其 close/high/low 仅为截至调用时刻的值，不是收盘值。`fill_data=False` 也无法阻止这一行为。配合 `is_new_bar()` 使用时，触发时机是新 bar 的第一个 tick，末尾这根 bar 的 close/high/low ≈ 开盘价。因此：
    - **指标计算**（通道、均线等需要完整 bar 的场景）：多请求一根，过滤集合竞价后**无条件丢弃末尾一根**，这与 BigQuant `handle_data` 在 bar 收盘后触发的语义对齐：
    ```python
    df = ContextInfo.get_market_data_ex(['close','high','low'], [code],
                                         period='1m', count=window+6,
                                         end_time=end_time)[code]
    df = df[df.index.time >= pd.to_datetime('09:30').time()]
    df = df.iloc[:-1]   # 无条件丢弃末尾未收盘 bar
    ```
    - **取当前价格下单**：直接用末尾一根，不需要丢弃，此时要的就是最新值。
    - 此约定**必须配合 `is_new_bar()` 节流使用**，否则末尾 bar 可能是中途某个 tick 的值，丢弃逻辑仍然成立但语义会多退一根。
13. **账户类型完整列表**：`get_trade_detail_data` 第二参数可选值：`'STOCK'`（股票）、`'FUTURE'`（期货）、`'CREDIT'`（信用）、`'FUTURE_OPTION'`（期货期权）、`'STOCK_OPTION'`（股票期权/ETF期权，**注意不是 `'OPTION'`**）、`'HUGANGTONG'`（沪港通）、`'SHENGANGTONG'`（深港通）。
14. **`cancel` 实盘签名**：实盘/仿真中 `cancel` 需要四个参数 `cancel(orderid, account_id, account_type, ContextInfo)`，不是只传 `orderid + ContextInfo`。
15. **期权持仓 `m_eSideFlag`**：判断时要同时兼容字符串和整数，即 `dt.m_eSideFlag in ('0', 0)` 而非只判断 `== 0`。
16. **期权价格单位换算**：期权 `lastPrice` 单位是元/份，一张合约通常为 10000 份，所以资金需求 ≈ `lastPrice × 10000`；用于买入开仓时资金校验：`can_buy = int(available / (lastPrice * 10000))`。
17. **熔断/流动性不足检测**：通过判断卖二至卖五（askPrice[1:]）和买二至买五（bidPrice[1:]）全为0来检测熔断或极度缺乏流动性，此时应暂停下单：
    ```python
    tick = ContextInfo.get_market_data_ex([], [code], period='tick', count=1)
    ask_2_5 = list(tick[code]["askPrice"][-1])[1:]
    bid_2_5 = list(tick[code]["bidPrice"][-1])[1:]
    is_frozen = all(x == 0 for x in ask_2_5) and all(x == 0 for x in bid_2_5)
    ```
18. **`after_init` 用途**：`after_init(ContextInfo)` 在 `init` 之后、第一根 bar 之前执行，适合恢复持久化状态（如从 CSV 读取历史订单），因为此时行情连接已就绪。
19. **`init` 中获取 account/accountType**：实盘/仿真中，`init` 函数内可直接访问全局变量 `account`（账户ID字符串）和 `accountType`（账户类型字符串），无需 ContextInfo 前缀，通常在 init 里赋值到全局状态对象上供后续使用。
20. **`get_market_data_ex` 无论实盘还是回测，都应传 `end_time` 锚定当前K线**：不传 `end_time` 时，历史回放和回测模式下返回的永远是策略启动时刻的实时行情快照，完全不感知当前 barpos，导致每根K线都取到同一段数据，策略逻辑完全失效（实测：14261根历史1m K线全程返回同一份数据，tick时间戳冻结在策略启动时刻）。实盘模式下传 `end_time` 无副作用，因此统一写法是始终传当前bar时间：
    ```python
    bar_dt = datetime.fromtimestamp(ContextInfo.get_bar_timetag(ContextInfo.barpos) / 1000)
    data = ContextInfo.get_market_data_ex(
        ['close'], [code],
        period='5m', count=10,
        end_time=bar_dt.strftime('%Y%m%d%H%M%S'),
    )
    ```
    指标计算时末尾一根可能未收盘，需在取完数据后额外 `df = df.iloc[:-1]` 丢弃（见注意事项第12条）。
21. **`is_last_bar()` 和 `is_new_bar()` 在纯回测与实盘/仿真中行为不同**：
    - **历史回放/仿真阶段**：每根历史 bar 只触发一次 `handlebar`（实测：14261根bar，`tick#` 与 `bar#` 严格1:1）；`is_new_bar()` 每根均为 `True`，`is_last_bar()` 全程为 `False`。
    - **实盘阶段**：每个 Level-1 分笔都触发 `handlebar`，一根 bar 内会触发多次；`is_new_bar()` 仅第一个 tick 为 `True`，`is_last_bar()` 进入实盘后为 `True`。
    - **纯回测**：`is_last_bar()` 始终为 `False`，用它过滤会导致策略一次都不执行。
    - **两者均不接受参数**，`is_new_bar(code)` 会报 `TypeError`。

    适用场景：
    ```python
    # 仿真/实盘：过滤历史回放，只在实时K线执行下单
    if not ContextInfo.is_last_bar():
        return

    # 纯回测：is_new_bar() 每根都 True，直接写逻辑即可，无需额外过滤

    # 仿真/实盘：每根1m bar收盘时触发一次（新bar的最后一个tick）
    if not ContextInfo.is_new_bar() or not ContextInfo.is_last_bar():
        return
    ```
22. **`passorder` 是异步挂单，不代表成交**：`passorder` 调用成功只意味着委托送进了 QMT，当根K线内持仓不会立刻变化。可靠的成交确认方式是在下单前记录持仓快照，下一根K线用持仓差判断真实成交量：
    ```python
    # 下单前记录持仓
    pos = get_trade_detail_data(ACCOUNT_ID, ACCOUNT_TYPE, 'POSITION')
    vol_before = next((int(p.m_nVolume) for p in pos if p.m_strInstrumentID == code_id), 0)

    passorder(50, 1101, ACCOUNT_ID, code, 11, price, qty, 'strategy', 1, '', ContextInfo)

    # 下一根K线对账
    pos_now = get_trade_detail_data(ACCOUNT_ID, ACCOUNT_TYPE, 'POSITION')
    vol_now = next((int(p.m_nVolume) for p in pos_now if p.m_strInstrumentID == code_id), 0)
    filled_qty = vol_now - vol_before   # >0 说明有成交
    ```
23. **`get_market_data_ex` 返回 DataFrame，不能用 `if not data:` 判空**：`get_market_data_ex` 返回 `{code: DataFrame}`，取出的 `DataFrame` 对象在布尔上下文中会抛 `ValueError: The truth value of a DataFrame is ambiguous`。正确判空方式：
    ```python
    data = ContextInfo.get_market_data_ex(fields, [code], period='1m', count=10)
    df = data.get(code)
    if df is None or df.empty:
        return
    ```
    如果封装函数可能返回 DataFrame 或 dict，用：
    ```python
    if result is None or (hasattr(result, 'empty') and result.empty) or (isinstance(result, dict) and not result):
        return
    ```
24. **`xtdata` 不能在 QMT 内部策略里使用**：`xtdata.subscribe_quote` 等接口是给"QMT 外部 Python 脚本"（miniQMT 模式）用的。在 QMT 内部策略中调用会报"无法连接行情服务"错误。内部策略行情全部通过 `ContextInfo.get_market_data_ex` / `ContextInfo.get_full_tick` 获取。
25. **`get_full_tick` 回测中返回实时快照，非历史数据**：回测中调用 `get_full_tick` 返回的是运行时的当前行情快照，时间戳不随 barpos 变化，无法用于回测逻辑。历史行情数据统一用 `get_market_data_ex`（记得传 `end_time`）。
26. **`get_market_data_ex` 的 `period='tick'` 中 `pctChg`（涨跌幅）和 `openInterest`（持仓量）始终为 `nan`**：实测（ETF 588000.SH 和期权 10011621.SHO 均如此），这两个字段通过 tick 周期取到的值不可用，不要依赖。涨跌幅需自行用 `(close - preClose) / preClose` 计算；持仓量（期权未平仓量）需通过其他接口获取。
27. **`get_option_detail_data` 完整字段清单**（实测自 `10011621.SHO`）：
    ```
    ExchangeID          交易所（如 SHO）
    InstrumentID        合约代码（不含市场）
    ProductID           品种名+标的（如 "科创50(588000)"）
    OpenDate            上市日
    ExpireDate          到期日（同 EndDelivDate）
    EndDelivDate        到期交割日
    PreClose            昨收价
    SettlementPrice     结算价
    UpStopPrice         涨停价
    DownStopPrice       跌停价
    LongMarginRatio     买方保证金比例
    ShortMarginRatio    卖方保证金比例
    PriceTick           最小变动价位（如 0.0001）
    VolumeMultiple      合约乘数（通常 10000 份/张）
    MaxMarketOrderVolume 市价单单笔上限（张）
    MinMarketOrderVolume 市价单单笔下限（张）
    MaxLimitOrderVolume  限价单单笔上限（张）
    MinLimitOrderVolume  限价单单笔下限（张）
    OptUnit             合约单位（份，通常 10000.0）
    MarginUnit          每张合约保证金（元，用于计算可开手数）
    OptUndlCode         标的代码（如 588000）
    OptUndlMarket       标的市场（如 SH）
    OptExercisePrice    行权价
    OptUndlRiskFreeRate 无风险利率
    OptUndlHistoryRate  标的历史波动率
    optType             认购/认沽（CALL / PUT）
    NeeqExeType         北交所行权类型（非北交所期权一般为 2147483647）
    ```
    常用计算示例：
    ```python
    info = ContextInfo.get_option_detail_data(code)
    margin_per_lot = info['MarginUnit']          # 每张保证金（元）
    max_lots = int(available_cash / margin_per_lot)  # 可开手数
    strike = info['OptExercisePrice']            # 行权价
    expire = info['ExpireDate']                  # 到期日字符串，如 '20260923'
    is_call = info['optType'] == 'CALL'
    ```
28. **自定义模块热更新用 `sys.modules.pop + import`，不要用 `importlib.reload`**：QMT 对每次策略运行分配内部模块名（`_M_STRATEGYNAME_SHxxxxxxxx`），`importlib.reload` 会追溯上一次运行的旧模块名，旧实例消失后报 `ImportError: module _M_XXX not in sys.modules`。正确模式是文件顶层先 import（让模块首次进入 `sys.modules`），`init()` 里用 `pop + import` 重载：
    ```python
    # 文件顶层（确保首次加载 + 静态分析可识别）
    import sys
    sys.path.insert(0, r'D:\Quant\qmt')
    import qs_qmt
    import trade_qmt

    def init(ContextInfo):
        # pop 切断旧实例关联，再 import 即为全新加载，每次重粘贴运行都能热更
        for _mod in ('qs_qmt', 'trade_qmt'):
            sys.modules.pop(_mod, None)
        import qs_qmt, trade_qmt  # noqa: F811
    ```

---

## 如何响应用户

当用户调用 `/qmt-coder` 时：

1. **描述策略逻辑** → 直接生成完整、可运行的 QMT Python 代码。
2. **有报错** → 结合 QMT 框架特性分析原因并给出修复方案。
3. **询问 API 用法** → 从速查表精准回答，附可运行示例片段。
4. **要优化策略** → 分析逻辑和性能问题，给出改进后的完整代码。

始终输出完整、可直接粘贴到 QMT 编辑器运行的代码。
