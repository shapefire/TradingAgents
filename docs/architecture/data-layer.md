# 数据层架构

> 厂商路由门面、数据适配器实现、错误处理、前瞻防护

## 1. 数据厂商路由系统

**核心文件**: `tradingagents/dataflows/interface.py`

### 1.1 工具类别定义

数据层定义了 6 个工具类别，每个类别包含一组数据获取方法：

| 类别 | 方法 | 说明 |
|------|------|------|
| `core_stock_apis` | `get_stock_data` | OHLCV 价格数据 |
| `technical_indicators` | `get_indicators` | 技术分析指标 |
| `fundamental_data` | `get_fundamentals`, `get_balance_sheet`, `get_cashflow`, `get_income_statement` | 公司基本面 |
| `news_data` | `get_news`, `get_global_news`, `get_insider_transactions` | 新闻与内部交易 |
| `macro_data` | `get_macro_indicators` | 宏观经济指标 |
| `prediction_markets` | `get_prediction_markets` | 预测市场概率 |

### 1.2 厂商实现矩阵

每个方法支持哪些厂商：

| 方法 | yfinance | Alpha Vantage | FRED | Polymarket |
|------|----------|---------------|------|------------|
| `get_stock_data` | ✓ | ✓ | — | — |
| `get_indicators` | ✓ | ✓ | — | — |
| `get_fundamentals` | ✓ | ✓ | — | — |
| `get_balance_sheet` | ✓ | ✓ | — | — |
| `get_cashflow` | ✓ | ✓ | — | — |
| `get_income_statement` | ✓ | ✓ | — | — |
| `get_news` | ✓ | ✓ | — | — |
| `get_global_news` | ✓ | ✓ | — | — |
| `get_insider_transactions` | ✓ | — | — | — |
| `get_macro_indicators` | — | — | ✓ | — |
| `get_prediction_markets` | — | — | — | ✓ |

### 1.3 厂商选择优先级

1. **工具级覆盖**（最高优先级）：`config["tool_vendors"][method_name]`
2. **类别级默认**：`config["data_vendors"][category]`
3. **兜底**：`"default"` 哨兵值，使用所有可用厂商

### 1.4 回退链逻辑

`route_to_vendor()` 将配置值视为逗号分隔的有序厂商链：

```
配置值 = "yfinance,alpha_vantage"
  → 先尝试 yfinance
  → 失败则尝试 alpha_vantage
  → 全部失败则抛出错误或返回 NO_DATA_AVAILABLE 哨兵
```

**设计原则**（引用 issues #988/#289）：不静默回退到用户未选择的厂商。

## 2. 错误分类体系

**文件**: `tradingagents/dataflows/errors.py`

```
VendorError (基类)
  ├── NoMarketDataError       -- 标的空/过时（携带 symbol, canonical, detail）
  ├── VendorRateLimitError    -- 临时限流，跳到下一个厂商
  └── VendorNotConfiguredError -- 缺少 API key（同时继承 ValueError）
```

- `VendorNotConfiguredError` 同时继承 `VendorError` 和 `ValueError`，保持向后兼容
- `NoMarketDataError` 携带结构化信息：symbol、canonical 形式、detail 原因
- 错误消息不泄露敏感数据（如 API key 值）

## 3. 厂商适配器实现

### 3.1 Alpha Vantage

**文件**: `alpha_vantage_common.py`, `alpha_vantage_stock.py`, `alpha_vantage_fundamentals.py`, `alpha_vantage_indicator.py`, `alpha_vantage_news.py`

| 特性 | 实现 |
|------|------|
| API Key | `ALPHA_VANTAGE_API_KEY` 环境变量 |
| 超时 | 30 秒网络超时 |
| 限流检测 | 检查 JSON 响应中的 "Information" 和 "Note" 字段 |
| 技术指标 | 支持 12 个（SMA 50/200, EMA 10, MACD/Signal/Hist, RSI, Bollinger bands, ATR, VWMA） |
| 前瞻防护 | `_filter_reports_by_date()` 过滤 `curr_date` 之后的财务期间 |
| 股票数据 | 使用 `TIME_SERIES_DAILY_ADJUSTED`，compact/full outputsize 选择 |

### 3.2 yFinance

**文件**: `y_finance.py`, `yfinance_news.py`, `stockstats_utils.py`

| 特性 | 实现 |
|------|------|
| API Key | 无需 |
| 符号规范化 | `symbol_utils.py` 映射经纪商符号到 Yahoo Finance 约定 |
| 限流重试 | `yf_retry()` 指数退避（最多 3 次，基础延迟 2s） |
| OHLCV 缓存 | 下载 5 年数据，保存到 CSV，后续调用复用 |
| 过时检测 | `_assert_ohlcv_not_stale()` 拒绝最新行 >10 天前的帧 |
| 前瞻防护 | `load_ohlcv()` 过滤 `curr_date` 之后的行；`filter_financials_by_date()` 过滤之后的财务期间列 |
| 新闻 | `Ticker.get_news()` + `yf.Search()`；`_in_news_window()` 排除历史窗口中的无日期文章 |

**符号映射示例**：

| 经纪商符号 | Yahoo Finance 符号 |
|-----------|-------------------|
| XAUUSD | GC=F |
| EURUSD | EURUSD=X |
| BTCUSD | BTC-USD |
| ^GSPC | ^GSPC (S&P 500) |

### 3.3 FRED（联邦储备经济数据）

**文件**: `fred.py`

| 特性 | 实现 |
|------|------|
| API Key | `FRED_API_KEY` 环境变量 |
| 超时 | 30 秒 |
| 别名表 | `MACRO_SERIES` 映射友好名称到 FRED 系列 ID |
| 输出格式 | Markdown：系列标题、单位、频率、最新值、窗口变化、最近观测表（最多 40 行） |

**可用宏观数标**：cpi, core_pce, unemployment, fed_funds_rate, 10y_treasury, yield_curve 等。

### 3.4 Polymarket（预测市场）

**文件**: `polymarket.py`

| 特性 | 实现 |
|------|------|
| API Key | 无需（公共 Gamma API） |
| 搜索 | 前瞻性开放市场，过滤已关闭/已解决 |
| 排序 | 按交易量排序 |
| 输出 | 隐含概率 + 1 周价格变动 |

### 3.5 Reddit

**文件**: `reddit.py`

| 特性 | 实现 |
|------|------|
| API Key | 无需 |
| 协议 | RSS/Atom feed（JSON 端点被 WAF 封锁） |
| 搜索范围 | wallstreetbets, stocks, investing 子版块 |
| 限流处理 | 429 时退避一次，尊重 Retry-After 头 |
| 降级 | 返回占位符字符串而非抛出异常 |

### 3.6 StockTwits

**文件**: `stocktwits.py`

| 特性 | 实现 |
|------|------|
| API Key | 无需 |
| API | 公共 symbol-stream API |
| 输出 | 最近消息 + 情绪标签（Bullish/Bearish） |
| 降级 | 任何失败时优雅降级 |

### 3.7 Market Data Validator

**文件**: `market_data_validator.py`

确定性验证工具，无 LLM 参与：
- 构建 ground-truth 快照：最新 OHLCV 行、10 个常用指标、近期收盘价
- 作为 Market Analyst LLM 精确数值声明的来源，防止幻觉

## 4. 工具系统

### 4.1 工具定义模式

所有工具使用 `@tool` 装饰器定义，是 `route_to_vendor()` 的薄包装：

```python
@tool
def get_stock_data(symbol: str, start_date: str, end_date: str) -> str:
    """获取 OHLCV 价格数据"""
    return route_to_vendor("get_stock_data", symbol, start_date, end_date)
```

### 4.2 ToolNode 组装

在 `trading_graph.py` 的 `_create_tool_nodes()` 中组装：

| ToolNode | 包含工具 |
|----------|----------|
| `market` | `get_stock_data`, `get_indicators`, `get_verified_market_snapshot` |
| `social` | `get_news` |
| `news` | `get_news`, `get_global_news`, `get_insider_transactions`, `get_macro_indicators`, `get_prediction_markets` |
| `fundamentals` | `get_fundamentals`, `get_balance_sheet`, `get_cashflow`, `get_income_statement` |

### 4.3 特殊处理

- `get_indicators`：分割 LLM 输出的逗号分隔指标字符串，逐个处理
- `get_verified_market_snapshot`：确定性快照，用于验证 LLM 的精确数值声明

## 5. 前瞻防护机制

所有数据适配器实现前瞻防护，确保回测场景下不泄露未来数据：

| 适配器 | 防护点 |
|--------|--------|
| yfinance OHLCV | `load_ohlcv()` 过滤 `curr_date` 之后的行 |
| yfinance 财务 | `filter_financials_by_date()` 过滤 `curr_date` 之后的财务期间列 |
| yfinance 新闻 | `_in_news_window()` 排除历史窗口中的无日期文章 |
| Alpha Vantage 财务 | `_filter_reports_by_date()` 过滤 `curr_date` 之后的财务期间 |
| yfinance 过时检测 | `_assert_ohlcv_not_stale()` 拒绝最新行 >10 天前的帧 |

## 6. 符号规范化

**文件**: `tradingagents/dataflows/symbol_utils.py`

将经纪商符号映射到 Yahoo Finance 约定：
- 外汇对：`EURUSD` → `EURUSD=X`
- 加密货币：`BTCUSD` → `BTC-USD`
- 商品：`XAUUSD` → `GC=F`
- 指数：添加 `^` 前缀（如 `SPX` → `^GSPC`）

## 7. 数据流全景

```
Agent LLM 调用工具
  → @tool 函数
    → route_to_vendor()
      → 解析工具类别
      → 确定厂商链（tool_vendors > data_vendors > default）
      → 按链顺序尝试厂商实现
        → yfinance: 符号规范化 → API 调用 → 缓存/重试 → 格式化输出
        → alpha_vantage: API key 验证 → HTTP 请求 → 限流检测 → 格式化输出
        → fred: 别名解析 → API 请求 → Markdown 格式化
        → polymarket: Gamma API → 市场搜索 → 概率格式化
        → reddit: RSS feed → 消息解析 → 情绪格式化
        → stocktwits: symbol-stream API → 消息解析 → 情绪格式化
      → 成功: 返回格式化数据字符串
      → 限流/未配置: 跳到下一个厂商
      → 无数据: 返回 NO_DATA_AVAILABLE: 哨兵
      → 全部失败: 抛出第一个真实错误
    → Agent LLM 接收数据用于分析
```
