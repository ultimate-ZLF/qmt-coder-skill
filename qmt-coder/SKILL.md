---
name: qmt-coder
display_name: QMT 量化交易助手
description: 帮你用迅投 QMT 框架编写回测和实盘交易策略，包含完整 API 参考与策略模板。
invocation: qmt-coder
global: true
---

# QMT 量化交易助手

帮你用迅投 **MiniQMT / XtQuant** 框架编写回测和实盘交易策略。

## 使用方式

输入 `/qmt-coder` 后描述你的需求，例如：

- "帮我写一个沪深300成分股的双均线回测策略"
- "如何在实盘中获取当前持仓？"
- "我的 passorder 报错了，帮我看看"
- "写一个网格交易策略，标的是 000001.SZ"

## 覆盖内容

- `init` / `handlebar` 策略结构
- `ContextInfo` 完整 API 速查（行情、财务、合约、订阅）
- 下单函数：`passorder`（实盘/仿真推荐）
- 持仓/资金查询：`get_trade_detail_data`
- 定时任务：`run_time`
- 图表绘制：`paint` / `draw_text`
- 回测参数设置：资金、基准、滑点、手续费
- 常见错误排查与注意事项
- 完整可运行策略模板

---

## 项目惯例（D:\Quant\qmt）

### 策略分层结构

新策略应将逻辑拆成四层，`handlebar` 只做流程调度：

| 层 | 内容 | 职责 |
|---|---|---|
| 数据层 | `get_latest_price()` `get_hold_qty()` | 所有 QMT API 调用集中在此 |
| OrderManager | 订单追踪 + 活跃委托查询 + 撤单 | 可跨策略复用的类，不依赖策略参数 |
| 交易层 | `place_open()` `place_close()` | 封装 `passorder`，返回 `True/False` |
| 时间工具 | `parse_bar_time()` `in_trade_session()` `interval_elapsed()` | 纯函数，无副作用 |

### 日志规范

使用 `logging` 模块，**不用 `print`**。日志目录由调用方传入。

```python
import logging
import os
from datetime import datetime

def _setup_logger(name='strategy', log_dir=None):
    """
    log_dir: 日志目录绝对路径，为 None 时只输出到控制台。
    幂等：若 logger 已有 handler 直接返回，防止 QMT 重复 import 时重复添加。
    """
    _logger = logging.getLogger(name)
    if _logger.handlers:
        return _logger
    _logger.setLevel(logging.DEBUG)
    fmt = logging.Formatter(
        fmt='%(asctime)s  %(levelname)-8s  %(message)s',
        datefmt='%Y-%m-%d %H:%M:%S'
    )
    ch = logging.StreamHandler()
    ch.setLevel(logging.DEBUG)
    ch.setFormatter(fmt)
    _logger.addHandler(ch)

    if log_dir:
        os.makedirs(log_dir, exist_ok=True)
        log_path = os.path.join(log_dir,
                                'strategy_%s.log' % datetime.now().strftime('%Y%m%d'))
        fh = logging.FileHandler(log_path, encoding='utf-8')
        fh.setLevel(logging.DEBUG)
        fh.setFormatter(fmt)
        _logger.addHandler(fh)
        _logger.info('日志文件路径: %s', log_path)
    return _logger

# 调用时传入实际路径，例如：
# logger = _setup_logger('option_strategy', log_dir=r'D:\Quant\qmt\logs')
logger = _setup_logger('option_strategy')
```

日志级别约定：
- `DEBUG`  — 遍历持仓条目、原始返回 keys 等细节
- `INFO`   — 正常流程节点（时间、价格、下单结果）
- `WARNING`— 数据缺失、账户为空等可恢复情况
- `ERROR`  — 接口抛异常

### handlebar 节流模式

```python
def handlebar(ContextInfo):
    # 两个方法均不接受参数，传参会 TypeError
    if not ContextInfo.is_new_bar() or not ContextInfo.is_last_bar():
        return  # 同一根 K 线只取第一个 tick，且跳过历史回放阶段
    ...
```

- `is_new_bar()` — 同一根 K 线只取第一个 tick，过滤后续 tick（**无参，传参报 TypeError**）
- `is_last_bar()` — **历史回放阶段返回 False，进入实盘阶段后返回 True**；用于挡住历史 K 线回放防止误触发（**无参，传参报 TypeError**）

### 间隔控制（用 bar 计数，不用秒数）

```python
# 全局状态
_last_trade_bar = -1   # 上次下单时的 bar 序号（-1 = 尚未交易）

def interval_elapsed(last_bar, current_bar):
    if last_bar < 0:
        return True
    return (current_bar - last_bar) >= INTERVAL_MINUTES

# handlebar 中
if not interval_elapsed(_last_trade_bar, _bar_count):
    ...  # 跳过

# 下单成功后
_last_trade_bar = _bar_count
```

`INTERVAL_MINUTES` 直接等于 bar 根数，策略运行在 1m bar 上时即等于分钟数。**不要用 datetime 差值或 `total_seconds()`**，bar 计数更精确且不受时钟偏移影响。

**间隔豁免规则**：持仓冻结（`avail=0`）需要继续运行撤单逻辑，可豁免间隔检查。`avail>0`（正常可平）和 `total=0`（空仓开仓）路径**一律受间隔限制，不豁免**。

### OrderManager（基于真实委托状态的撤单重报，可复用，已实盘验证）

> **设计原则**：不沿用 BigQuant 版 `order_mgr is not None` 的"自维护在途旗标"思路，也不是 v4 早期版本自己猜 `in_flight`。QMT 有真实的委托状态查询接口（`get_trade_detail_data(..., 'ORDER')`），直接以交易所返回的 `m_nOrderStatus` 为准，不用策略自己维护一套影子状态。完整实现见 `order.py`（模块）+ `test_order.py`（实盘验证脚本），下方内容均已用真实委托跑通。

**委托号获取：不要用 `get_last_order_id`，用"提交前后差集匹配"**

`get_last_order_id(account, account_type, 'order')` 返回的是**账户维度**"最新一笔"委托，不区分合约。`passorder` 提交是异步的，紧接着调用 `get_last_order_id` 时账户内部委托列表可能还没登记这笔新委托，读到的是提交前遗留的旧记录。**实测踩坑**：拿到一笔更早的旧委托（状态已是"已成"），误判为"本次测试委托已成交"，触发了错误的平仓操作，意外卖出了真实持仓。

正确做法：提交前记录该合约已有委托号快照，提交后按合约过滤 + 与快照做差集，精确定位新增的那一笔；当根 bar 查不到就转入待解析状态，下一根 bar 自动重试。

```python
# order.py 提供的底层函数
before_ids = order.snapshot_order_sys_ids(account, account_type, full_code, bar_no)  # 提交前
passorder(...)                                                                        # 提交
new_order = order.find_new_order(account, account_type, full_code, before_ids, bar_no)  # 提交后
# new_order is None：无新增（柜台登记延迟，下一根 bar 重试）或新增多笔（歧义，不猜测选择）
```

**实测延迟窗口（2026-07-02 实盘验证，合约 10011641.SHO，60+ 根 1m bar，15+ 轮完整周期，全部一致）**：
- **委托登记延迟**：稳定为 1 根 bar。Bar N 提交，Bar N 内查询大概率查不到，Bar N+1 必定能通过差集匹配查到
- **撤单生效延迟**：稳定为 1 根 bar。`cancel()` 返回 `True` 后，Bar N 状态仍是 50（已报），Bar N+1 状态变为 54（已撤）
- 全程未触发废单（57），废单原因字段（`m_strCancelInfo` / `m_strErrorMsg`）在正常撤单场景下均为空字符串，真实废单场景下的取值尚未验证

**基本流程（`OrderManager.check_and_act`）**：

1. 每根 `handlebar` **开头**（而不是下单之后）先尝试解析待处理委托，再查询已追踪委托的状态——此时看到的是**上一根 bar** 提交/操作的委托
2. 按 `m_nOrderStatus` 分类决策（完整枚举见 `prompt.md` 里的 `m_nOrderStatus` 表）：
   - **待解析（差集匹配未命中）**：柜台登记延迟，什么都不做，下一根 bar 自动重试
   - **废单（57）/ 已撤（54）/ 部撤（53）**：按剩余量（原报量 − 已成交量）重新报单
   - **超时未成交**（状态在 49/50/55 且挂单已跨过 1 根 bar）：调撤单，下一根 bar 状态会变为已撤
   - **撤单已提交待生效（51/52）**：什么都不做，等下一根 bar
   - **终态（56 已成）**：本轮委托已结束，清空追踪

```python
om = OrderManager(account=ACCOUNT_ID, full_code=OPTION_CODE, account_type=ACCOUNT_TYPE,
                   requeue_fn=my_requeue_fn)  # requeue_fn 由调用方提供，决定重报的价格/方式

# 下单前后
before_ids = om.before_submit(bar_no)
passorder(...)
om.on_submit(before_ids, submit_qty, 'open' | 'close', bar_no)

# handlebar 开头，无条件调用
action, order_obj = om.check_and_act(ContextInfo, bar_no)
# action: 'no_order' | 'resolving' | 'succeeded' | 'requeued' | 'cancelling' | 'pending' | 'waiting'
```

**撤单 API（官方确认，已验证）**：全局函数 `cancel(orderId, accountID, accountType, ContextInfo)`，返回 `bool` —— 表示"是否发出了取消委托信号"，**不代表撤单已成功**，真正是否撤成还要在下一根 bar 回查 `m_nOrderStatus` 是否变为 54（已撤）/53（部撤）。实测撤单生效延迟稳定为 1 根 bar。

```python
order_sys_id = order.m_strOrderSysID   # 从 get_trade_detail_data(..., 'ORDER') 拿到的委托对象
ok = cancel(order_sys_id, ACCOUNT_ID, ACCOUNT_TYPE, ContextInfo)
# ok=True 只说明撤单请求送达，不能当作撤单已完成的确认
```

`ContextInfo.cancel_order(...)` 和 `cancelorder_stock(...)` 都不是有效接口，不要使用。

**多策略共用账户时的合约隔离**：差集匹配按"合约"去重，不区分策略来源。若两个策略（如本项目的 `dual_qmt.py` 和某个测试脚本）同时对同一合约下单，差集匹配可能把对方策略提交的委托误判为己方新增，导致状态判断/撤单/平仓作用在错误的委托上。**测试脚本应使用一个当前没有任何策略在交易的合约**，不要用正式策略正在用的合约做探索性测试。

### 自定义模块 import 与热更新

QMT 在首次加载策略时执行顶层 `import`，之后重新运行策略不会重载已缓存的模块。
如果策略依赖同目录的自定义模块（如 `qs_qmt.py`），必须：

1. 在文件顶部将策略目录加入 `sys.path`
2. 在 `init()` 里用 `importlib.reload` 强制重载，并用 `globals().update` 刷新引用

```python
import sys
sys.path.insert(0, r'D:\Quant\qmt')

from your_module import SomeClass, some_func  # 首次加载用

def init(ContextInfo):
    import importlib, your_module
    importlib.reload(your_module)
    from your_module import SomeClass, some_func
    globals().update({
        'SomeClass': SomeClass,
        'some_func': some_func,
    })
    # 同步子模块的 logger handler（如有）
    import logging
    logging.getLogger('your_module').handlers = logger.handlers
    logging.getLogger('your_module').setLevel(logging.DEBUG)
    ...
```

顶层的 `import` 只是为了让编辑器/静态分析不报错，实际生效的是 `init()` 里 reload 后的版本。

### QMT 内置函数注入子模块

`passorder`、`get_trade_detail_data` 等 QMT 内置函数只在主策略文件的全局命名空间里可用，`import` 进来的模块有独立的 `__dict__`，直接调用会 `NameError`。

**解决方案**：每个子模块暴露一个 `init(ns)` 函数；主策略在 reload 之后立即调用，把自己的全局字典注入子模块。

子模块（`trade_qmt.py` / `utils_qmt.py` 等）：

```python
def init(ns):
    """将主策略文件的全局命名空间（含 QMT 内置函数）注入本模块。
    主策略 init() 里调用：import trade_qmt; trade_qmt.init(globals())
    """
    globals().update(ns)
```

主策略文件 `init(ContextInfo)` 里，reload 之后、使用子模块符号之前：

```python
def init(ContextInfo):
    import importlib, trade_qmt, utils_qmt, qs_qmt
    importlib.reload(trade_qmt)
    importlib.reload(utils_qmt)
    importlib.reload(qs_qmt)

    # reload 后立即注入，顺序不能颠倒
    trade_qmt.init(globals())
    utils_qmt.init(globals())
    qs_qmt.init(globals())   # 当前不用内置函数，统一注入备扩展

    from trade_qmt import buy_option, sell_option
    from utils_qmt import get_total_asset, print_all_positions
    ...
```

**注意事项**：
- `globals().update(ns)` 是全量覆盖，`init()` 里传的是调用时快照，之后主策略再修改全局变量不会同步到子模块——如果子模块需要的是动态值，改用 `ContextInfo` 传参而不是全局变量。
- 仅用于注入内置函数，不要用这个机制传递业务状态（会污染子模块命名空间）。
- 对于只做纯计算、不调用 QMT 内置函数的模块（如纯指标计算），可以不加 `init()`，加上也无害。

### passorder 期权常用 optype

| optype | 含义 |
|---|---|
| 50 | 买入开仓 |
| 51 | 卖出平仓 |
| 52 | 买入平仓 |
| 53 | 卖出开仓 |

`ordertype=1101` 为实时单，QMT 仿真/实盘推荐。

### passorder prType（价格类型）枚举

| prType | 描述 |
|--------|------|
| -1 | 无效（仅对 `algo_passorder` 起作用） |
| 0 | 卖5价 |
| 1 | 卖4价 |
| 2 | 卖3价 |
| 3 | 卖2价 |
| 4 | 卖1价 |
| 5 | 最新价 |
| 6 | 买1价 |
| 7 | 买2价（组合不支持） |
| 8 | 买3价（组合不支持） |
| 9 | 买4价（组合不支持） |
| 10 | 买5价（组合不支持） |
| 11 | 指定价（仅单股支持，组合不支持） |
| 12 | 涨跌停价（对手方最远端价格） |
| 13 | 挂单价（本方一档价格） |
| 14 | 对手价（对方一档价格） |
| 18 | 市价最优价 \[郑商所\]\[期货\]（不支持模拟交易） |
| 19 | 市价即成剩撤 \[大商所\]\[期货\]（不支持模拟交易） |
| 20 | 市价全额成交或撤 \[大商所\]\[期货\]（不支持模拟交易） |
| 21 | 市价最优一档即成剩撤 \[中金所\]\[期货\]（不支持模拟交易） |
| 22 | 市价最优五档即成剩撤 \[中金所\]\[期货\]（不支持模拟交易） |
| 23 | 市价最优一档即成剩转 \[中金所\]\[期货\]（不支持模拟交易） |
| 24 | 市价最优五档即成剩转 \[中金所\]\[期货\]（不支持模拟交易） |
| 26 | 限价即时全部成交否则撤单 \[上交所\]\[深交所\]\[期权\]（不支持模拟交易） |
| 27 | 市价即成剩撤 \[上交所\]\[期权\]（不支持模拟交易） |
| 28 | 市价即全成否则撤 \[上交所\]\[期权\]（不支持模拟交易） |
| 29 | 市价剩转限价 \[上交所\]\[期权\]（不支持模拟交易） |
| 42 | 最优五档即时成交剩余撤销 \[上交所\]\[北交所\]\[股票\]（不支持模拟交易） |
| 43 | 最优五档即时成交剩转限价 \[上交所\]\[北交所\]\[股票\]（不支持模拟交易） |
| 44 | 对手方最优价格委托 \[上交所\]\[深交所\]\[北交所\]\[股票\]\[期权\]（不支持模拟交易） |
| 45 | 本方最优价格委托 \[上交所\]\[深交所\]\[北交所\]\[股票\]\[期权\]（不支持模拟交易） |
| 46 | 即时成交剩余撤销委托 \[深交所\]\[股票\]\[期权\]（不支持模拟交易） |
| 47 | 最优五档即时成交剩余撤销委托 \[深交所\]\[股票\]\[期权\]（不支持模拟交易） |
| 48 | 全额成交或撤销委托 \[深交所\]\[股票\]\[期权\]（不支持模拟交易） |
| 49 | 盘后定价 |

**期权实盘常用组合：**
- `prType=5, price=-1` — 最新价，price 传 -1
- `prType=11, price=<指定价>` — 限价单，price 传实际报价
- `prType=14, price=-1` — 对手价（买入用卖1，卖出用买1），price 传 -1

### 合约代码格式

- A 股 ETF：`{6位代码}.SH` / `.SZ`，如 `588000.SH`
- 上交所期权：`{8位合约编码}.SHO`，如 `10011621.SHO`
- 持仓查询时需拆分：`instrument = full_code.split('.')[0]`，与 `pos.m_strInstrumentID` 比对

### get_market_data_ex 完整签名与 period 枚举

```python
ContextInfo.get_market_data_ex(
    fields=[],          # 字段列表，如 ['close', 'high', 'low', 'open', 'volume']
    stock_code=[],      # 合约代码列表
    period='follow',    # 数据周期（见下表）
    start_time='',      # 起始时间，格式 'YYYYMMDDHHMMSS'，与 count 二选一
    end_time='',        # 结束时间，格式同上；必须传当前 bar 时间（见下方说明）
    count=-1,           # 取最近 N 根，-1 表示取全部；与 start_time 二选一
    dividend_type='follow',  # 复权方式：'none' / 'front' / 'back' / 'follow'
    fill_data=True,     # True：补齐缺失 bar；False：不补
    subscribe=True,     # 是否订阅实时推送
)
# 返回 {code: DataFrame}，DataFrame 行索引为时间戳，列为 fields
```

> **⚠️ 必须传 `end_time`，始终使用当前 bar 的时间戳**
>
> 不传 `end_time` 时返回的是"当前最新"数据，在回测中会导致所有 bar 取到相同快照，
> 在实盘中也可能取到未来 bar。任何情况下都应传入当前 bar 时间：
>
> ```python
> timetag  = ContextInfo.get_bar_timetag(ContextInfo.barpos)
> dt       = datetime.fromtimestamp(timetag / 1000.0)
> end_time = dt.strftime('%Y%m%d%H%M%S')
>
> raw = ContextInfo.get_market_data_ex(
>     fields, [code], period='1m', count=N,
>     end_time=end_time,
>     dividend_type='none', fill_data=False,
> )
> ```

**period 可选值：**

| period | 含义 |
|--------|------|
| `'tick'` | Tick 数据 |
| `'1m'` | 1分钟线 |
| `'5m'` | 5分钟线 |
| `'15m'` | 15分钟线 |
| `'30m'` | 30分钟线 |
| `'1h'` | 小时线 |
| `'1d'` | 日线 |
| `'1w'` | 周线 |
| `'1mon'` | 月线 |
| `'1q'` | 季线 |
| `'1hy'` | 半年线 |
| `'1y'` | 年线 |
| `'l2quote'` | Level2 行情快照 |
| `'l2quoteaux'` | Level2 行情快照补充 |
| `'l2order'` | Level2 逐笔委托 |
| `'l2transaction'` | Level2 逐笔成交 |
| `'l2transactioncount'` | Level2 大单统计 |
| `'l2orderqueue'` | Level2 委买委卖队列 |

**关键特性：API 原生支持周期聚合**，直接传 `period='5m'`/`'30m'`/`'1d'` 等即可获得对应周期的 OHLC，**无需自行 resample**。

