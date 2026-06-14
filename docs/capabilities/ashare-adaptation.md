# A 股市场改造指南

> 将 TradingAgents 改造为中国 A 股市场的完整路线图、改造项、实施建议。

---

## 1. 改造优先级矩阵

| 优先级 | 改造项 | 影响度 | 工作量 | 说明 |
|--------|--------|--------|--------|------|
| P0 | 数据厂商接入 | 🔴 关键 | 中 | 无中国数据源则无法分析 A 股 |
| P0 | 符号规范化 | 🔴 关键 | 小 | 无法识别 A 股 ticker |
| P0 | 宏观数据接入 | 🔴 关键 | 中 | 无中国宏观数据则 News Analyst 无法分析 |
| P1 | 情绪数据源 | 🟡 重要 | 大 | Reddit/StockTwits 对 A 股无效 |
| P1 | 技术指标扩展 | 🟡 重要 | 中 | 缺少中国常用指标 |
| P1 | 新闻源接入 | 🟡 重要 | 中 | 英文新闻对 A 股覆盖不足 |
| P2 | LLM 提供商扩展 | 🟢 一般 | 小 | 已有 DeepSeek/Qwen/GLM，可补充 |
| P2 | Schema 微调 | 🟢 一般 | 小 | 适配 A 股交易规则 |
| P3 | 交易日历 | 🟢 一般 | 中 | 中国节假日和交易时间 |

---

## 2. P0：数据厂商接入

### 2.1 推荐方案：AKShare

**理由**：
- 无需 API key（免费开源）
- 覆盖 A 股全市场（沪深主板、创业板、科创板、北交所）
- 提供 OHLCV、财务报表、宏观数据、新闻、资金流向
- 社区活跃，维护良好
- Python 接口友好

**需要创建的文件**：

```
tradingagents/dataflows/
├── akshare_adapter.py        # AKShare 数据适配器
├── china_macro.py            # 中国宏观数据（央行利率、CPI、M2 等）
└── china_calendar.py         # 中国交易日历
```

**需要修改的文件**：

| 文件 | 修改内容 |
|------|----------|
| `interface.py` | 在 `VENDOR_METHODS` 中注册 `akshare` 厂商 |
| `default_config.py` | 添加 `data_vendors` 的 A 股默认配置 |
| `symbol_utils.py` | 添加 A 股符号映射 |

**AKShare 适配器接口设计**：

```python
# tradingagents/dataflows/akshare_adapter.py

def get_stock_data_akshare(symbol: str, start_date: str, end_date: str) -> str:
    """获取 A 股 OHLCV 数据"""
    # symbol: "600519" 或 "600519.SS"
    # 调用 ak.stock_zh_a_hist()

def get_fundamentals_akshare(ticker: str, curr_date: str) -> str:
    """获取 A 股基本面数据"""
    # 调用 ak.stock_financial_analysis_indicator()

def get_balance_sheet_akshare(ticker: str, freq: str, curr_date: str) -> str:
    """获取 A 股资产负债表"""

def get_cashflow_akshare(ticker: str, freq: str, curr_date: str) -> str:
    """获取 A 股现金流量表"""

def get_income_statement_akshare(ticker: str, freq: str, curr_date: str) -> str:
    """获取 A 股利润表"""
```

### 2.2 符号规范化改造

**当前状态**：`symbol_utils.py` 不处理 A 股符号

**需要添加的映射**：

```python
# A 股符号规范化
_CHINA_INDEX_ALIASES = {
    "上证": "000001.SS",
    "深证": "399001.SZ",
    "创业板": "399006.SZ",
    "科创50": "000688.SS",
    "沪深300": "000300.SS",
    "中证500": "000905.SS",
}

def normalize_symbol(symbol: str) -> str:
    # 新增：SH600519 → 600519.SS
    # 新增：SZ000858 → 000858.SZ
    # 新增：600519 → 600519.SS（自动检测交易所）
    # 新增：000858 → 000858.SZ
```

**交易所后缀规则**：
- `.SS` — 上海证券交易所（60xxxx, 68xxxx, 000xxx 指数）
- `.SZ` — 深圳证券交易所（00xxxx, 30xxxx, 399xxx 指数）
- 60xxxx → `.SS`（上海主板）
- 00xxxx → `.SZ`（深圳主板）
- 30xxxx → `.SZ`（创业板）
- 68xxxx → `.SS`（科创板）

### 2.3 中国宏观数据接入

**需要接入的数据**：

| 数据 | 来源 | 用途 |
|------|------|------|
| PBOC LPR 利率 | AKShare | 货币政策方向 |
| PBOC MLF 利率 | AKShare | 中期政策利率 |
| 中国 CPI/PPI | AKShare/NBS | 通胀趋势 |
| M2 货币供应 | AKShare/PBOC | 流动性 |
| SHIBOR | AKShare | 银行间利率 |
| 中国 GDP | AKShare/NBS | 经济增长 |
| 社会融资规模 | AKShare/PBOC | 信用环境 |
| 北向资金流向 | AKShare | 外资动向 |
| 融资融券余额 | AKShare | 杠杆情绪 |

**示例实现**：

```python
# tradingagents/dataflows/china_macro.py

CHINA_MACRO_SERIES = {
    "lpr_1y": {"name": "1年期LPR", "func": "ak.macro_china_lpr()"},
    "lpr_5y": {"name": "5年期LPR", "func": "ak.macro_china_lpr()"},
    "cpi": {"name": "中国CPI", "func": "ak.macro_china_cpi_yearly()"},
    "ppi": {"name": "中国PPI", "func": "ak.macro_china_ppi_yearly()"},
    "m2": {"name": "M2货币供应", "func": "ak.macro_china_money_supply()"},
    "shibor": {"name": "SHIBOR", "func": "ak.rate_interbank()"},
    "gdp": {"name": "中国GDP", "func": "ak.macro_china_gdp()"},
    "north_flow": {"name": "北向资金", "func": "ak.stock_hsgt_north_net_flow_in_em()"},
}
```

---

## 3. P1：情绪数据源接入

### 3.1 东方财富股吧

**作用**：替代 Reddit/StockTwits，获取 A 股散户情绪

**需要创建**：`tradingagents/dataflows/eastmoney_guba.py`

**功能**：
- 按 ticker 获取股吧讨论帖
- 提取看多/看空情绪标签
- 统计多空比例
- 获取热度/阅读量

```python
def fetch_eastmoney_guba(symbol: str, limit: int = 50) -> str:
    """获取东方财富股吧讨论"""
    # 调用东方财富股吧 API
    # 返回格式化的情绪分析文本
```

### 3.2 雪球（Xueqiu）

**作用**：替代 StockTwits，获取高质量投资讨论

**需要创建**：`tradingagents/dataflows/xueqiu.py`

**功能**：
- 获取雪球讨论帖
- 用户持仓和观点
- 大 V 观点摘要

### 3.3 财联社（CLS）

**作用**：获取实时财经快讯，替代部分新闻功能

**需要创建**：`tradingagents/dataflows/cls_news.py`

**功能**：
- 实时财经快讯
- 市场影响标记
- 公司公告速递

---

## 4. P1：技术指标扩展

**当前支持的 12 个指标**：SMA 50/200, EMA 10, MACD, RSI, Bollinger, ATR, VWMA, MFI

**需要添加的 A 股常用指标**：

| 指标 | 说明 | 中国使用频率 |
|------|------|-------------|
| KDJ | 随机指标 | ⭐⭐⭐⭐⭐ 极高 |
| OBV | 能量潮 | ⭐⭐⭐⭐ 高 |
| WR | 威廉指标 | ⭐⭐⭐⭐ 高 |
| DMI/ADX | 趋向指标 | ⭐⭐⭐⭐ 高 |
| BIAS | 乖离率 | ⭐⭐⭐ 中高 |
| PSY | 心理线 | ⭐⭐⭐ 中高 |
| TRIX | 三重指数平滑 | ⭐⭐⭐ 中 |
| ROC | 变动速率 | ⭐⭐⭐ 中 |
| SAR | 抛物线转向 | ⭐⭐⭐ 中 |
| CR | 能量指标 | ⭐⭐ 低中 |

**修改文件**：
- `tradingagents/dataflows/stockstats_utils.py` — 添加新指标计算
- `tradingagents/dataflows/y_finance.py` — 在 `best_ind_params` 中注册
- `tradingagents/dataflows/market_data_validator.py` — 在验证快照中添加

---

## 5. P1：新闻源接入

**当前状态**：只支持 Yahoo Finance 新闻（英文，A 股覆盖不足）

**需要接入的中国新闻源**：

| 新闻源 | 类型 | API 方式 | 特点 |
|--------|------|----------|------|
| 财联社 (CLS) | 快讯 | Web API | 实时性强，市场影响大 |
| 新华财经 | 综合 | Web API | 权威性强 |
| 21世纪经济报道 | 深度 | Web API | 分析深度好 |
| 东方财富新闻 | 综合 | Web API | 覆盖面广 |
| 同花顺资讯 | 综合 | Web API | 与行情结合紧密 |

**需要创建**：`tradingagents/dataflows/china_news.py`

**修改文件**：
- `tradingagents/default_config.py` — 添加 `global_news_queries` 的中国版本

```python
# 中国宏观新闻查询词
china_news_queries = [
    "央行货币政策 LPR 利率",
    "中国GDP 经济数据 PMI",
    "北向资金 外资流向",
    "监管政策 IPO 注册制",
    "行业政策 新能源 半导体 人工智能",
]
```

---

## 6. P2：LLM 提供商扩展

**已支持**：DeepSeek, Qwen, GLM, MiniMax, Kimi, Ollama

**建议补充**：

| 提供商 | 优先级 | 说明 |
|--------|--------|------|
| 百度文心 (ERNIE) | 中 | 国内主流 LLM |
| 豆包 (Doubao) | 中 | 字节跳动，可通过 OpenAI 兼容端点使用 |
| 讯飞星火 (Spark) | 低 | 可通过 OpenAI 兼容端点使用 |
| 百川 (Baichuan) | 低 | 可通过 OpenAI 兼容端点使用 |

**注意**：大部分中国 LLM 已支持 OpenAI 兼容 API，可通过 `openai_compatible` 提供商直接使用。

---

## 7. P2：Schema 微调

**TraderProposal 适配**：

```python
class TraderProposal(BaseModel):
    action: TraderAction
    reasoning: str
    entry_price: Optional[float]
    stop_loss: Optional[float]
    position_sizing: Optional[str]
    # 新增 A 股特定字段
    lot_size: Optional[str] = "100股（1手）"  # A 股最小交易单位
    price_limit_note: Optional[str] = None     # 涨跌停说明（10%/20%）
```

**PortfolioDecision 适配**：

```python
class PortfolioDecision(BaseModel):
    rating: PortfolioRating
    executive_summary: str
    investment_thesis: str
    price_target: Optional[float]
    time_horizon: Optional[str]
    # 新增 A 股特定字段
    settlement_note: Optional[str] = "T+1 交割"  # 交割说明
```

---

## 8. P3：中国交易日历

**需要创建**：`tradingagents/dataflows/china_calendar.py`

**功能**：
- 识别中国法定节假日（春节、国庆等）
- 识别调休交易日（周末补班时可能开盘）
- 计算实际交易日（用于回测和指标计算）

```python
def is_china_trading_day(date: str) -> bool:
    """判断是否为中国A股交易日"""

def get_china_trading_days(start: str, end: str) -> list[str]:
    """获取区间内的交易日列表"""
```

**数据源**：AKShare 提供 `ak.tool_trade_date_hist_sina()` 获取交易日历。

---

## 9. 改造后的完整数据流

```
用户输入: "600519" (贵州茅台)
  │
  ▼
符号规范化: 600519 → 600519.SS
  │
  ▼
数据获取（AKShare 为主）:
  ├── OHLCV: ak.stock_zh_a_hist("600519")
  ├── 基本面: ak.stock_financial_analysis_indicator("600519")
  ├── 宏观: ak.macro_china_lpr(), ak.macro_china_cpi_yearly()
  ├── 新闻: cls_news API, eastmoney news
  ├── 情绪: eastmoney guba, xueqiu
  └── 北向资金: ak.stock_hsgt_north_net_flow_in_em()
  │
  ▼
Agent 分析（中文输出）:
  ├── Market Analyst: KDJ, MACD, RSI, BIAS 等指标分析
  ├── Sentiment Analyst: 东方财富股吧情绪 + 雪球讨论
  ├── News Analyst: 财联社快讯 + 央行政策 + 行业新闻
  └── Fundamentals Analyst: 茅台财务数据 + 估值分析
  │
  ▼
多空辩论 → Research Manager 裁决
  │
  ▼
Trader: 入场价、止损、仓位（考虑 100 股手数、涨跌停限制）
  │
  ▼
三方风控辩论 → Portfolio Manager 最终决策
  │
  ▼
输出: 中文交易报告 + 信号（Buy/Hold/Sell）
```

---

## 10. 分阶段实施建议

### 阶段 1（2-3 周）：基础可用

- 接入 AKShare 数据源
- 修复符号规范化
- 接入中国宏观数据
- 验证基本分析流程可用

### 阶段 2（2-3 周）：分析质量提升

- 接入东方财富股吧情绪
- 接入财联社新闻
- 添加 KDJ、OBV 等技术指标
- 扩展宏观新闻查询词

### 阶段 3（1-2 周）：精细化

- 接入雪球数据
- Schema 微调（涨跌停、手数）
- 中国交易日历
- 补充 LLM 提供商

---

## 11. 测试策略

```bash
# 阶段 1 测试
pytest tests/test_akshare_adapter.py -m unit
pytest tests/test_china_symbol.py -m unit
pytest tests/test_china_macro.py -m integration

# 阶段 2 测试
pytest tests/test_eastmoney_guba.py -m integration
pytest tests/test_cls_news.py -m integration
pytest tests/test_china_indicators.py -m unit

# 端到端测试
python main.py  # 测试 600519.SS 分析
```

---

## 12. 配置示例

```python
# A 股分析配置
config = {
    "llm_provider": "qwen-cn",
    "deep_think_l": "qwen3.7-max",
    "quick_think_l": "qwen3.6-flash",
    "output_language": "Chinese",
    "data_vendors": {
        "core_stock_apis": "akshare,yfinance",
        "technical_indicators": "akshare,yfinance",
        "fundamental_data": "akshare,yfinance",
        "news_data": "cls_news,eastmoney",
        "macro_data": "china_macro",
        "prediction_markets": "polymarket",
    },
    "benchmark_ticker": "000300.SS",  # 沪深300作为基准
    "max_debate_rounds": 2,
    "max_risk_discuss_rounds": 2,
}

graph = TradingAgentsGraph(config=config)
result = graph.propagate("600519", "2025-06-14", "stock")
```

---

## 13. 现有能力无需改造的部分

以下能力天然支持 A 股，无需改造：

| 能力 | 说明 |
|------|------|
| 多空辩论 | 机制通用，无需改造 |
| 三方风控 | 机制通用，无需改造 |
| 延迟反思 | 机制通用，基准已配置 .SS/.SZ |
| 崩溃恢复 | 机制通用，无需改造 |
| 多语言输出 | 已支持 Chinese |
| CLI 交互 | 机制通用，无需改造 |
| 资产自适应 | 机制通用，无需改造 |
| 多数据源路由 | 架构已支持新厂商接入 |
| LLM 提供商 | 已支持 DeepSeek/Qwen/GLM |
