# Agent 定义与角色

> 所有 LLM Agent 的详细定义：角色、工具、提示词结构、输出格式

## 1. 构建模式

所有 Agent 通过闭包工厂函数构建：

```python
def create_<role>(llm):
    # 绑定 LLM 到闭包
    def node(state: AgentState) -> dict:
        # 使用 llm 处理 state，返回状态更新
        return {"field": result}
    return node
```

**无类设计**：没有 Agent 类，全部使用闭包。唯一例外是 Trader 使用 `functools.partial` 绑定 `name="Trader"` 参数。

## 2. 结构化输出机制

**文件**: `tradingagents/agents/utils/structured.py`

### 2.1 绑定阶段（Agent 创建时）

```python
structured_llm = bind_structured(llm, Schema, agent_name)
```

调用 `llm.with_structured_output(schema)`。如果不支持（`NotImplementedError` 或 `AttributeError`），返回 `None` 并记录警告。

### 2.2 调用阶段（Agent 执行时）

```python
result = invoke_structured_or_freetext(
    structured_llm, plain_llm, prompt, render, agent_name
)
```

优先尝试结构化 LLM；如果失败（畸形 JSON、提供商错误），回退到 `plain_llm.invoke(prompt)`。保证管道永不阻塞于结构化输出。

### 2.3 设计哲学

> 主产物是散文。结构化输出叠加在其上，使字段描述成为模型的输出指令，渲染辅助函数将 Pydantic 实例转回 markdown 形状。

## 3. Pydantic Schema

**文件**: `tradingagents/agents/schemas.py`

### 3.1 枚举

| 枚举 | 值 | 使用者 |
|------|-----|--------|
| `PortfolioRating` | Buy, Overweight, Hold, Underweight, Sell | Research Manager, Portfolio Manager |
| `TraderAction` | Buy, Hold, Sell | Trader |
| `SentimentBand` | Bullish, Mildly Bullish, Neutral, Mixed, Mildly Bearish, Bearish | Sentiment Analyst |

### 3.2 Schema 定义

**ResearchPlan**（Research Manager 输出）：
- `recommendation`: PortfolioRating — 投资建议评级
- `rationale`: str — 推理依据
- `strategic_actions`: str — 战略行动

**TraderProposal**（Trader 输出）：
- `action`: TraderAction — 交易动作
- `reasoning`: str — 推理依据
- `entry_price`: Optional[float] — 入场价格
- `stop_loss`: Optional[float] — 止损价格
- `position_sizing`: Optional[str] — 仓位大小

**PortfolioDecision**（Portfolio Manager 输出）：
- `rating`: PortfolioRating — 最终评级
- `executive_summary`: str — 执行摘要
- `investment_thesis`: str — 投资论题
- `price_target`: Optional[float] — 目标价格
- `time_horizon`: Optional[str] — 时间范围

**SentimentReport**（Sentiment Analyst 输出）：
- `overall_band`: SentimentBand — 整体情绪带
- `overall_score`: float (0-10) — 整体分数
- `confidence`: Literal[low, medium, high] — 置信度
- `narrative`: str — 叙述

## 4. Analysts（5 个，顺序执行）

所有 Analyst 共享提示词模板结构：
- 系统消息：关于与其他助手协作
- 工具列表 partial
- 标的上下文 partial
- `MessagesPlaceholder` 用于对话历史

### 4.1 Market Analyst

**文件**: `tradingagents/agents/analysts/market_analyst.py`

| 属性 | 值 |
|------|-----|
| 工厂 | `create_market_analyst(llm)` |
| 角色 | 分析金融市场，选择最多 8 个技术指标，撰写趋势报告 |
| 工具 | `get_stock_data`, `get_indicators`, `get_verified_market_snapshot` |
| 输出 | 自由格式 markdown → `market_report` |
| LLM | quick_thinking_llm |

**提示词要点**：
- 包含 12 个命名指标的完整目录（close_50_sma, close_200_sma, close_10_ema, macd, macds, macdh, rsi, boll, boll_ub, boll_lb, atr, vwma）
- 指示先调用 `get_stock_data`，再用 `get_indicators` 获取特定指标
- 要求调用 `get_verified_market_snapshot` 作为精确 OHLCV/价格声明的来源
- 要求末尾附加 Markdown 汇总表

### 4.2 Sentiment Analyst

**文件**: `tradingagents/agents/analysts/sentiment_analyst.py`

| 属性 | 值 |
|------|-----|
| 工厂 | `create_sentiment_analyst(llm)` |
| 角色 | 多源情绪分析（Yahoo Finance 新闻 + StockTwits + Reddit） |
| 工具 | **无**（数据预取后注入 prompt） |
| 输出 | 结构化输出 `SentimentReport` → 渲染为 markdown → `sentiment_report` |
| LLM | quick_thinking_llm |

**特殊设计**：
- 不使用工具循环
- 数据在 LLM 调用前直接获取：`get_news.func()`、`fetch_stocktwits_messages()`、`fetchReddit_posts()`
- 注入为结构化块：`<start_of_news>`、`<start_of_stocktwits>`、`<start_of_reddit>`
- 每个数据源优雅降级，返回字符串而非抛出异常
- 重设计原因：旧版本在只有 Yahoo Finance 新闻工具时会伪造 Reddit/X 内容（issues #557, #796）

### 4.3 News Analyst

**文件**: `tradingagents/agents/analysts/news_analyst.py`

| 属性 | 值 |
|------|-----|
| 工厂 | `create_news_analyst(llm)` |
| 角色 | 分析近期新闻、世界事务、宏观经济指标、预测市场 |
| 工具 | `get_news`, `get_global_news`, `get_macro_indicators`, `get_prediction_markets` |
| 输出 | 自由格式 markdown → `news_report` |
| LLM | quick_thinking_llm |

**提示词要点**：
- 根据 `asset_type` 调整语言（"company" vs "asset"）
- 列出 FRED 可用的宏观指标：cpi, core_pce, unemployment, fed_funds_rate, 10y_treasury, yield_curve
- 列出预测市场主题
- 要求 Markdown 汇总表

### 4.4 Fundamentals Analyst

**文件**: `tradingagents/agents/analysts/fundamentals_analyst.py`

| 属性 | 值 |
|------|-----|
| 工厂 | `create_fundamentals_analyst(llm)` |
| 角色 | 分析公司基本面信息（财务文档、公司概况、基础财务、财务历史） |
| 工具 | `get_fundamentals`, `get_balance_sheet`, `get_cashflow`, `get_income_statement` |
| 输出 | 自由格式 markdown → `fundamentals_report` |
| LLM | quick_thinking_llm |

### 4.5 Social Media Analyst（已弃用）

**文件**: `tradingagents/agents/analysts/social_media_analyst.py`

向后兼容 shim，从 `sentiment_analyst.py` 重新导出，带弃用警告。

## 5. Research Team（多空辩论）

### 5.1 Bull Researcher

**文件**: `tradingagents/agents/researchers/bull_researcher.py`

| 属性 | 值 |
|------|-----|
| 工厂 | `create_bull_researcher(llm)` |
| 角色 | 主张投资 — 强调增长潜力、竞争优势、积极指标 |
| 工具 | 无 |
| 输出 | 追加到 `investment_debate_state.history` 和 `bull_history` |
| LLM | quick_thinking_llm |

**提示词输入**：4 份 Analyst 报告、辩论历史、Bear 最新论点、标的上下文。

### 5.2 Bear Researcher

**文件**: `tradingagents/agents/researchers/bear_researcher.py`

| 属性 | 值 |
|------|-----|
| 工厂 | `create_bear_researcher(llm)` |
| 角色 | 反对投资 — 强调风险、竞争劣势、消极指标 |
| 工具 | 无 |
| 输出 | 追加到 `investment_debate_state.history` 和 `bear_history` |
| LLM | quick_thinking_llm |

## 6. Research Manager（裁决者）

**文件**: `tradingagents/agents/managers/research_manager.py`

| 属性 | 值 |
|------|-----|
| 工厂 | `create_research_manager(llm)` |
| 角色 | 裁决多空辩论，产出结构化投资计划 |
| 工具 | 无 |
| 输出 | 结构化输出 `ResearchPlan` → `investment_plan` + `investment_debate_state.judge_decision` |
| LLM | **deep_thinking_llm** |

**评级指导**：
- Buy：强买入信号
- Overweight：适度积极
- Hold：证据平衡
- Underweight：适度消极
- Sell：强卖出信号

## 7. Trader

**文件**: `tradingagents/agents/trader/trader.py`

| 属性 | 值 |
|------|-----|
| 工厂 | `create_trader(llm)` |
| 角色 | 将投资计划转化为具体交易提案 |
| 工具 | 无 |
| 输出 | 结构化输出 `TraderProposal` → `trader_investment_plan` + `messages` |
| LLM | quick_thinking_llm |

**特殊构造**：使用 `functools.partial(trader_node, name="Trader")` 绑定名称参数。

**提示词结构**：双消息（system + user），而非 ChatPromptTemplate。

## 8. Risk Management Team（三方辩论）

### 8.1 Aggressive Debator

**文件**: `tradingagents/agents/risk_mgmt/aggressive_debator.py`

| 属性 | 值 |
|------|-----|
| 工厂 | `create_aggressive_debator(llm)` |
| 角色 | 拥护高回报、高风险机会 |
| 输出 | `latest_speaker: "Aggressive"` |

### 8.2 Conservative Debator

**文件**: `tradingagents/agents/risk_mgmt/conservative_debator.py`

| 属性 | 值 |
|------|-----|
| 工厂 | `create_conservative_debator(llm)` |
| 角色 | 保护资产，最小化波动，确保稳定增长 |
| 输出 | `latest_speaker: "Conservative"` |

### 8.3 Neutral Debator

**文件**: `tradingagents/agents/risk_mgmt/neutral_debator.py`

| 属性 | 值 |
|------|-----|
| 工厂 | `create_neutral_debator(llm)` |
| 角色 | 平衡视角，权衡收益和风险 |
| 输出 | `latest_speaker: "Neutral"` |

所有三方辩论者接收相同输入：标的上下文、4 份 Analyst 报告、Trader 决策、辩论历史、其他两方的最新回复。

## 9. Portfolio Manager（最终裁决者）

**文件**: `tradingagents/agents/managers/portfolio_manager.py`

| 属性 | 值 |
|------|-----|
| 工厂 | `create_portfolio_manager(llm)` |
| 角色 | 综合风控辩论产出最终交易决策 |
| 工具 | 无 |
| 输出 | 结构化输出 `PortfolioDecision` → `final_trade_decision` + `risk_debate_state.judge_decision` |
| LLM | **deep_thinking_llm** |

**读取内容**：
- Research Manager 投资计划
- Trader 交易提案
- 完整风控辩论历史
- `past_context`（历史记忆教训）

## 10. Agent 工具总览

| 工具 | 文件 | 参数 | 用途 |
|------|------|------|------|
| `get_stock_data` | core_stock_tools.py | symbol, start_date, end_date | OHLCV 价格数据 |
| `get_indicators` | technical_indicators_tools.py | symbol, indicator, curr_date, look_back_days=30 | 单个技术指标 |
| `get_fundamentals` | fundamental_data_tools.py | ticker, curr_date | 综合基本面数据 |
| `get_balance_sheet` | fundamental_data_tools.py | ticker, freq, curr_date | 资产负债表 |
| `get_cashflow` | fundamental_data_tools.py | ticker, freq, curr_date | 现金流量表 |
| `get_income_statement` | fundamental_data_tools.py | ticker, freq, curr_date | 利润表 |
| `get_news` | news_data_tools.py | ticker, start_date, end_date | 标的特定新闻 |
| `get_global_news` | news_data_tools.py | curr_date, look_back_days, limit | 全球宏观新闻 |
| `get_insider_transactions` | news_data_tools.py | ticker | 内部交易活动 |
| `get_macro_indicators` | macro_data_tools.py | indicator, curr_date, look_back_days | FRED 宏观时间序列 |
| `get_prediction_markets` | prediction_markets_tools.py | topic, limit | Polymarket 隐含概率 |
| `get_verified_market_snapshot` | market_data_validation_tools.py | symbol, curr_date, look_back_days | 确定性 OHLCV 快照 |

## 11. Agent 使用摘要

| Agent | 工具 | 结构化输出 | 自由文本调用 | 辩论风格 |
|-------|------|-----------|-------------|----------|
| Fundamentals Analyst | 4 工具 (bind_tools) | 否 | 否（工具循环） | N/A |
| Market Analyst | 3 工具 (bind_tools) | 否 | 否（工具循环） | N/A |
| News Analyst | 4 工具 (bind_tools) | 否 | 否（工具循环） | N/A |
| Sentiment Analyst | 0（预取） | 是 (SentimentReport) | 回退 | N/A |
| Bull Researcher | 0 | 否 | 是 | Bull vs Bear |
| Bear Researcher | 0 | 否 | 是 | Bear vs Bull |
| Research Manager | 0 | 是 (ResearchPlan) | 回退 | 裁决辩论 |
| Trader | 0 | 是 (TraderProposal) | 回退 | N/A |
| Aggressive Debator | 0 | 否 | 是 | 三方风控 |
| Conservative Debator | 0 | 否 | 是 | 三方风控 |
| Neutral Debator | 0 | 否 | 是 | 三方风控 |
| Portfolio Manager | 0 | 是 (PortfolioDecision) | 回退 | 裁决风控辩论 |

## 12. Agent 工具函数

**文件**: `tradingagents/agents/utils/agent_utils.py`

| 函数 | 用途 |
|------|------|
| `get_language_instruction()` | 非英语输出时追加语言指令 |
| `resolve_instrument_identity(ticker)` | LRU 缓存（256 条）yfinance 查询，返回公司名称、行业、交易所等 |
| `build_instrument_context(ticker, asset_type, identity)` | 构建确定性身份字符串，防止 Agent 幻觉错误公司身份 |
| `get_instrument_context_from_state(state)` | 从 state 读取 instrument_context 或回退构建 |
| `create_msg_delete()` | 返回 LangGraph 节点函数，清空消息并注入上下文锚定占位符 |

## 13. 记忆/反思系统

**文件**: `tradingagents/agents/utils/memory.py`

`TradingMemoryLog` — 追加式 markdown 决策日志：

**Phase A（写入）**：
- `store_decision(ticker, trade_date, final_trade_decision)`
- 格式：`[date | ticker | rating | pending]`
- 幂等保护：检查现有 pending 条目

**Phase B（更新）**：
- `update_with_outcome(ticker, trade_date, raw_return, alpha_return, holding_days, reflection)`
- 替换 tag 为实际收益：`[date | ticker | rating | +X.X% | +X.X% | Nd]`
- 追加 `REFLECTION:` 部分
- 原子写入（临时文件 + `os.replace()`）

**读取**：
- `get_past_context(ticker, n_same=5, n_cross=3)`
- 返回格式化上下文字符串用于 prompt 注入
- 最多 5 条同 ticker 历史 + 3 条跨 ticker 历史

**轮转**：超过 `memory_log_max_entries` 时丢弃最旧的已解析条目，保留 pending 条目。

## 14. 评级系统

**文件**: `tradingagents/agents/utils/rating.py`

- `RATINGS_5_TIER`：`("Buy", "Overweight", "Hold", "Underweight", "Sell")`
- `parse_rating(text, default="Hold")`：两遍启发式解析
  1. 查找显式 `"Rating: X"` 标签（容忍 markdown 粗体和冒号/连字符分隔符）
  2. 回退到文本中找到的第一个 5 级评级词
