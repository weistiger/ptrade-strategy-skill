---
name: ptrade-strategy
description: |
  国金 PTRADE 量化平台策略生成技能。根据用户需求，
  生成可在 PTRADE 平台运行的 Python 量化交易策略代码。
  触发关键词：PTRADE、ptrade、量化策略、量化交易、生成策略、写策略代码。
metadata:
  openclaw:
    emoji: "📈"
---

# PTRADE 量化策略生成 Skill

## 概述

本 Skill 从知识库「国金量化」中读取 PTRADE API 文档，为用户生成可在 PTRADE 平台运行的量化交易策略代码。

**知识库文档结构（`个人知识库 → 国金量化`）：**
| 文件 | 说明 |
|------|------|
| `PTrade新版API帮助文档.pdf` | API 帮助文档 |
| `PTrade研究模块简要使用说明.pdf` | 研究模块使用说明 |
| `PTrade系统量化交易与回测的区别.pdf` | 回测 vs 实盘 |
| `PTrade新版API帮助文档.pdf` | API 完整参考 |
| `PTrade API 完整详细文档.docx` | 函数与变量大全 |
| `PTrade API 函数与变量大全.docx` | API 速查 |
| `demo-单因子策略` | 单因子示例 |
| `demo-双均线策略` | 双均线示例 |
| `demo-小市值日线交易策略` | 小市值示例 |
| `demo-三因子策略` | 三因子示例 |
| `demo-日内交易策略` | 日内示例 |
| `demo-协整配对策略` | 配对交易示例 |
| `demo-二八轮动策略` | 轮动策略示例 |
| `demo-阿隆指标` | 技术指标示例 |
| `demo-猛犸策略` | 高级策略示例 |
| `Ptrade 量化交易 API接口文档` | 在线文档（web URL） |

---

## 核心 API 参考

### 常用对象

```python
# 获取上下文
context = get_json('context')
g = context.get('g', {})         # 全局对象
security = g.get('security')    # 证券行情
trade   = g.get('trade')        # 交易模块
log     = g.get('log')          # 日志模块

# 常用函数
log.info("消息")                # 打印日志
log.error("错误消息")
get_json('context')             # 获取上下文
```

### 行情数据

```python
# 获取历史 K 线
security.get_bars(n: int, end_date: str, fields: list, count: int, period: str)
# period: "1d"(日线) "1m"(1分钟) "5m"/"15m"/"30m"/"60m"/"1h"/"4h"
# fields: ["open", "high", "low", "close", "volume"]

# 获取财务数据
security.get_financial_data(start_date, end_date, count, fields)
# fields: "pubDate","totalShare","flowShare","closePrice","PE","PB","PCF","PS"
# fields: "netProfit","grossProfit","operatingProfit","totalRevenue","debtAssetRatio"

# 获取成分股
security.get_index_stocks(index_code: str)
# index_code: "000300.XSHG"(沪深300) "000016.XSHG"(上证50) "000852.XSHG"(中证1000)

# 获取当前持仓
security.get_positions()

# 获取当前账户信息
security.get_account_info()

# 获取主力资金流向
security.get_money_flow(start_date, end_date, count, fields)
```

### 交易下单

```python
# 买入（回测模式）
trade.open_long(security: str, volume: int, price: float = 0, price_type: str = "")

# 卖出（回测模式）
trade.close_long(security: str, volume: int, price: float = 0, price_type: str = "")

# 撮合价格类型
# market: 以开盘价成交
# latest: 以最新价成交（需订阅实时行情）
# 指定价格: 以指定价格成交（需订阅Level2）
```

### 财务因子

```python
# 获取财务因子
security.get_误符串(n: int, fields: list, count: int, end_date: str)
# 示例: security.get_financial_data(count=10, end_date="", fields=["PE","PB","PCF"])
```

### 支持的库

- **数据分析**：`numpy`, `pandas`, `scipy`
- **机器学习**：`sklearn`（不支持深度学习框架）
- **时间序列**：`statsmodels`
- **因子研究**：`alphalens`, `ta-lib`
- **数据库**：`pymongo`, `redis`

---

## 策略模板

### 模板 1：日线双均线策略

```python
def initialize(context):
    """初始化策略参数"""
    context.params = {
        "security": "000001.XSHG"         # 交易标的
        "short_window": 5                  # 短期均线
        "long_window": 20                 # 长期均线
    }
    context.g.security = get_json("context").get("g", {}).get("security")
    context.g.trade    = get_json("context").get("g", {}).get("trade")

def handle_data(context):
    security = context.g.security
    trade    = context.g.trade

    # 获取近 N 根 K 线
    bars = security.get_bars(
        n=context.params["long_window"],
        end_date="",
        fields=["open", "high", "low", "close", "volume"],
        count=context.params["long_window"],
        period="1d"
    )

    close = bars["close"]
    short_ma = close[-5:].mean()  # 5 日均线
    long_ma  = close[-20:].mean() # 20 日均线

    # 金叉买入
    if short_ma > long_ma and close[-2] <= close[-3] * 0.995:
        trade.open_long(context.params["security"], 100, price=0, price_type="market")

    # 死叉卖出
    if short_ma < long_ma:
        trade.close_long(context.params["security"], 100, price=0, price_type="market")
```

### 模板 2：财务因子选股策略

```python
def initialize(context):
    context.params = {
        "index": "000300.XSHG",   # 选股范围：沪深300
        "n": 10,                  # 持仓数量
        "factor": "PE",           # 估值因子：PE/PB/PCF/PS
        "ascending": True,        # True=低估值优先
    }
    context.g.security = get_json("context").get("g", {}).get("security")
    context.g.trade    = get_json("context").get("g", {}).get("trade")

def handle_data(context):
    security = context.g.security
    trade    = context.g.trade

    # 获取指数成分股
    stocks = security.get_index_stocks(context.params["index"])

    # 批量获取财务数据
    financial_data = security.get_financial_data(
        start_date="", end_date="",
        count=1,
        fields=[context.params["factor"]]
    )

    # 按因子排序
    sorted_stocks = sorted(stocks, key=lambda s: financial_data.get(s, {}).get(context.params["factor"], float("inf")))
    if context.params["ascending"]:
        sorted_stocks = sorted_stocks[::-1]

    # 取前N只，等权配置
    target_stocks = sorted_stocks[:context.params["n"]]
    for stock in target_stocks:
        trade.open_long(stock, 100, price=0, price_type="market")
```

### 模板 3：日内交易策略

```python
def initialize(context):
    context.params = {
        "security": "000001.XSHG",
        "upper_limit": 0.05,   # 涨幅上限
        "lower_limit": -0.03,   # 跌幅下限
        "volume_threshold": 500000  # 成交量阈值
    }
    context.g.security = get_json("context").get("g", {}).get("security")
    context.g.trade    = get_json("context").get("g", {}).get("trade")

def handle_data(context):
    security = context.g.security
    trade    = context.g.trade

    # 获取5分钟K线
    bars = security.get_bars(n=20, end_date="", fields=["close", "volume"], count=20, period="5m")
    close  = bars["close"]
    volume = bars["volume"]

    last_close = close[-1]
    prev_close = close[-2]
    change = (last_close - prev_close) / prev_close if prev_close > 0 else 0

    # 放量突破
    avg_vol = volume[:-1].mean()
    if volume[-1] > avg_vol * 1.5 and change > context.params["upper_limit"]:
        trade.open_long(context.params["security"], 100, price=0, price_type="market")

    # 止损
    if change < context.params["lower_limit"]:
        trade.close_long(context.params["security"], 100, price=0, price_type="market")
```

---

## 生成策略流程

1. **理解需求**：确认策略类型（选股/择时/套利/事件驱动）、标的、周期、信号
2. **确认 API**：参考上方 API 参考，选择合适的行情/交易函数
3. **套用模板**：从上方模板库中选择最接近的模板，填入参数
4. **代码生成**：输出完整的 `initialize` + `handle_data` 策略代码
5. **说明要点**：附上 PTRADE 平台运行说明（参数配置、回测 vs 实盘差异）

## 注意事项

- PTRADE 不支持实时 tick 数据回测，需订阅 Level2 才能用指定价格撮合
- `open_long`/`close_long` 的 volume 为股数（100 股整数倍）
- 回测撮合规则：默认 market 以当日开盘价成交；需订阅实时行情才能用 latest
- 因子选股建议用沪深300/中证500/中证1000成分股作为股票池过滤
- 组合持仓建议最多不超过20只，单只权重不超过10%
