# TradingAgents 交易能力深度分析

> 系统梳理项目的核心能力、每个能力对交易的具体帮助、使用示例。

---

## 能力 1：多层级结构化交易决策

**是什么**：系统通过三个决策 Agent 产出层层递进的交易决策，从抽象研究到具体执行参数。

**对交易的帮助**：
- 不是简单给出 Buy/Hold/Sell，而是提供完整的交易方案：入场价、止损位、仓位大小、目标价、持仓周期
- 5 级评级（Buy/Overweight/Hold/Underweight/Sell）比简单的 3 级更精细
- 结构化输出确保每个字段都有明确含义，不会产生歧义

**示例输出**：

```
评级: Buy
执行摘要: 建议在 $185 附近建仓，止损设在 $172（-7%），目标价 $220（+19%），
          建议仓位不超过总资金的 5%，持仓周期 3-6 个月。
投资论题: 基于技术面 MACD 金叉、基本面 P/E 低于行业平均、
         消息面即将发布新品等多重利好因素...
价格目标: $220
持仓周期: 3-6 个月
```

**涉及模块**：
- `tradingagents/agents/schemas.py` — ResearchPlan, TraderProposal, PortfolioDecision
- `tradingagents/agents/trader/trader.py` — 生成入场价、止损、仓位
- `tradingagents/agents/managers/portfolio_manager.py` — 最终决策

---

## 能力 2：对抗式多空辩论

**是什么**：Bull Researcher 和 Bear Researcher 进行多轮辩论，各自引用数据支持论点，最后由 Research Manager 裁决。

**对交易的帮助**：
- 避免单一视角偏差（只看多或只看空）
- 过程中引用具体数据（技术指标、财务数据、新闻事件），不是空洞的观点
- Research Manager 作为独立裁判，必须明确表态支持哪一方
- 辩论过程本身就是一份完整的投资研究报告

**示例场景**：

```
Bull: "MACD 在上周五形成金叉，RSI 从超卖区回升至 45，
       同时公司 Q3 营收同比增长 23%，远超市场预期..."
Bear: "但 RSI 仍在 50 以下，成交量持续萎缩，
       而且公司最大客户刚刚宣布转向竞争对手产品..."
Bull: "客户转型的影响已在股价中反映，当前 P/E 仅 12x，
       低于 5 年平均的 18x，估值具备安全边际..."
Research Manager: "综合辩论，看多方在估值和基本面数据上更有说服力，
                 但看空方提出的客户风险值得警惕。建议 Overweight，但控制仓位。"
```

**涉及模块**：
- `tradingagents/agents/researchers/bull_researcher.py`
- `tradingagents/agents/researchers/bear_researcher.py`
- `tradingagents/agents/managers/research_manager.py`
- `tradingagents/graph/conditional_logic.py` — 辩论轮次控制

---

## 能力 3：三方风控辩论

**是什么**：Aggressive、Conservative、Neutral 三个风险分析师从不同角度评估 Trader 的交易提案，轮番辩论后由 Portfolio Manager 做最终决策。

**对交易的帮助**：
- 激进方确保不错过高回报机会
- 保守方确保充分考虑尾部风险和下行保护
- 中立方防止任何极端观点主导决策
- 三方轮流发言，每方都能回应其他两方的论点，形成真正的辩证

**示例场景**：

```
Aggressive: "这个标的近期涨幅强劲，技术面突破关键阻力位，
            建议大胆加仓，目标价可以看高 30%..."
Conservative: "但日线 RSI 已达 78，严重超买，
              而且大盘整体处于下降趋势中，建议等回调再入场，
              或者严格控制仓位在 2% 以内..."
Neutral: "两方都有道理。技术面确实超买，但基本面支撑强劲。
         建议分批建仓：先用 3% 仓位试探，回调后再加仓至 5%，
         同时设置 8% 的止损..."
Portfolio Manager: "综合三方意见，采纳 Neutral 的分批建仓策略，
                  评级 Overweight，仓位 3-5%，止损 8%..."
```

**涉及模块**：
- `tradingagents/agents/risk_mgmt/aggressive_debator.py`
- `tradingagents/agents/risk_mgmt/conservative_debator.py`
- `tradingagents/agents/risk_mgmt/neutral_debator.py`
- `tradingagents/agents/managers/portfolio_manager.py`

---

## 能力 4：专业分析师团队

**是什么**：4 个专业分析师从不同维度收集和分析数据，每个分析师使用专属工具获取数据。

**对交易的帮助**：
- **Market Analyst**：技术分析，识别趋势、支撑阻力、指标信号
- **Sentiment Analyst**：情绪分析，综合新闻、社交媒体、散户情绪
- **News Analyst**：新闻和宏观分析，识别事件驱动因素
- **Fundamentals Analyst**：基本面分析，评估公司财务健康度

**每个分析师的工具和职责**：

| 分析师 | 工具 | 分析维度 |
|--------|------|----------|
| Market | get_stock_data, get_indicators, get_verified_market_snapshot | 技术面：趋势、指标、价格模式 |
| Sentiment | 预取新闻+StockTwits+Reddit | 情绪面：散户情绪、多空比例、社交热度 |
| News | get_news, get_global_news, get_macro_indicators, get_prediction_markets | 消息面：公司新闻、宏观事件、市场预期 |
| Fundamentals | get_fundamentals, get_balance_sheet, get_cashflow, get_income_statement | 基本面：财务数据、估值、盈利质量 |

**示例**：分析 AAPL 时，Market Analyst 发现 MACD 金叉，Sentiment Analyst 发现 Reddit 上看多情绪高涨，News Analyst 发现苹果即将发布新品，Fundamentals Analyst 发现 P/E 低于行业平均 — 四个维度的信号被后续辩论环节综合评估。

---

## 能力 5：延迟反思与记忆系统

**是什么**：系统在做出交易决策后，等待实际结果出来，然后生成反思，注入到未来的决策中。

**对交易的帮助**：
- **不是马后炮**：反思基于实际收益数据，不是决策时的乐观预测
- **避免重复错误**：Portfolio Manager 会看到"过去的教训"
- **跨标的学习**：A 股的反思可以影响 B 股的决策（模式迁移）
- **反馈闭环**：系统越用越聪明

**示例流程**：

```
2025-01-15: 分析 AAPL，决策 Buy，评级 Overweight
            → 存储为 pending 条目

2025-04-15: 再次分析 AAPL
            → 检测到 1 月 15 日的 pending 条目
            → 获取实际收益：+12.5%，Alpha +8.3%（vs SPY）
            → 生成反思："方向正确，但目标价过于保守。
              实际涨幅超出预期，主要驱动力是新品发布超预期，
              而非基本面改善。下次应更重视产品周期因素。"
            → 注入 Portfolio Manager 的 prompt 作为历史教训

2025-04-15: Portfolio Manager 看到反思后：
            "参考过去 AAPL 的分析经验，本次更重视产品周期因素..."
```

**涉及模块**：
- `tradingagents/agents/utils/memory.py` — TradingMemoryLog
- `tradingagents/graph/reflection.py` — Reflector
- `tradingagents/graph/trading_graph.py` — _resolve_pending_entries()

---

## 能力 6：多数据源厂商路由

**是什么**：统一的数据接口，支持多个数据厂商，有序回退，不静默降级。

**对交易的帮助**：
- 一个数据源限流或不可用时，自动切换到下一个
- 用户可以显式选择数据源优先级
- 不会因为数据源问题导致分析中断
- 支持不同数据源的组合（如 yfinance 获取价格，FRED 获取宏观数据）

**示例配置**：

```python
data_vendors = {
    "core_stock_apis": "yfinance,alpha_vantage",  # 价格数据：优先 yfinance，回退 AV
    "technical_indicators": "yfinance",             # 技术指标：只用 yfinance
    "fundamental_data": "yfinance,alpha_vantage",   # 基本面：优先 yfinance
    "news_data": "yfinance",                        # 新闻：只用 yfinance
    "macro_data": "fred",                           # 宏观：只用 FRED
    "prediction_markets": "polymarket",             # 预测市场：只用 Polymarket
}
```

**涉及模块**：
- `tradingagents/dataflows/interface.py` — route_to_vendor()
- `tradingagents/dataflows/errors.py` — 错误分类体系

---

## 能力 7：17+ LLM 提供商支持

**是什么**：支持 17 种 LLM 提供商，包括中国提供商（DeepSeek、Qwen、GLM、MiniMax、Kimi），深度推理和快速推理分离。

**对交易的帮助**：
- 可以使用最适合的模型：深度推理用强模型（如 Claude Opus），快速分析用轻量模型
- 中国用户可以直接使用国产 LLM，无需翻墙
- 双区域支持（国际/中国），不同的 API key
- 本地部署支持（Ollama），数据不出境

**已支持的中国 LLM**：

| 提供商 | 国际端点 | 中国端点 | 模型示例 |
|--------|----------|----------|----------|
| DeepSeek | deepseek | — | deepseek-chat, deepseek-reasoner |
| Qwen (阿里) | qwen | qwen-cn | qwen3.7-max, qwen3.6-flash |
| GLM (智谱) | glm | glm-cn | glm-5, glm-4.7 |
| MiniMax | minimax | minimax-cn | MiniMax-M2.7 |
| Kimi (月之暗面) | kimi | — | moonshot-v1 |
| Ollama | ollama | — | qwen3:latest (本地) |

---

## 能力 8：崩溃恢复检查点

**是什么**：基于 SQLite 的检查点系统，支持从上次中断的地方恢复。

**对交易的帮助**：
- 长时间运行的分析（深度模式可能需要 10+ 分钟）不会因为网络波动或进程中断而白费
- 相同 ticker+date 的分析总是从最后成功的节点继续
- 分析完成后自动清理检查点，避免陈旧状态

**示例**：

```bash
# 启用检查点
tradingagents --checkpoint

# 分析过程中断（Ctrl+C 或网络问题）
# 再次运行相同 ticker+date
tradingagents --checkpoint
# → 自动从上次中断的节点继续，不需要重新分析所有 Agent

# 清理所有检查点
tradingagents --clear-checkpoints
```

**涉及模块**：
- `tradingagents/graph/checkpointer.py` — SQLite 检查点

---

## 能力 9：防幻觉数据验证

**是什么**：确定性的 ground-truth 验证工具，防止 LLM 编造数据。

**对交易的帮助**：
- Market Analyst 可能会"编造"技术指标数值
- `get_verified_market_snapshot` 提供经过验证的真实数据
- 明确指示 LLM："以此快照为准，如果其他工具输出与此冲突，标记差异而非编造数字"
- 防止基于错误数据做出交易决策

**示例**：

```
Market Analyst 调用 get_verified_market_snapshot("AAPL", "2025-06-14")

返回：
最新 OHLCV: Open=$185.20, High=$187.50, Low=$184.80, Close=$186.90, Volume=52.3M
10日 EMA: $184.32
50日 SMA: $179.85
200日 SMA: $172.40
RSI(14): 62.5
MACD: 1.23 (Signal: 0.89, Histogram: 0.34)
布林带: 中轨=$183.50, 上轨=$189.20, 下轨=$177.80

⚠️ 使用此快照作为精确 OHLCV 和指标声明的来源。
   如果其他工具输出与此冲突，标记差异而非编造数字。
```

**涉及模块**：
- `tradingagents/dataflows/market_data_validator.py`
- `tradingagents/agents/utils/market_data_validation_tools.py`

---

## 能力 10：多语言输出

**是什么**：支持 12+ 种语言的分析报告输出。

**对交易的帮助**：
- 中文用户可以获得中文分析报告
- 中国 LLM + 中文输出 = 完整的中文交易分析体验
- 每个 Agent 的提示词自动追加语言指令

**支持的语言**：English, Chinese, Japanese, Korean, Hindi, Spanish, Portuguese, French, German, Arabic, Russian + 自定义

**示例配置**：

```python
config = {
    "llm_provider": "qwen-cn",
    "deep_think_l": "qwen3.7-max",
    "quick_think_l": "qwen3.6-flash",
    "output_language": "Chinese",
}
```

---

## 能力 11：CLI 交互式分析

**是什么**：完整的交互式命令行界面，8 步引导式配置，实时仪表板。

**对交易的帮助**：
- 非技术人员也能使用：回答问题即可完成配置
- 实时看到每个 Agent 的进度和输出
- 分析完成后自动保存报告到文件
- 支持检查点恢复

**CLI 流程**：

```
Step 1: 输入 ticker（如 AAPL、BTC-USD、600519.SS）
Step 2: 选择分析日期
Step 3: 选择输出语言
Step 4: 选择分析师团队
Step 5: 选择研究深度（浅/中/深）
Step 6: 选择 LLM 提供商
Step 7: 选择模型
Step 8: 配置提供商特定参数
→ 开始分析，实时显示进度
→ 分析完成，保存报告
```

---

## 能力 12：资产类型自适应

**是什么**：自动识别股票/加密货币，调整分析师团队和提示词。

**对交易的帮助**：
- 加密货币自动排除 Fundamentals Analyst（无公司基本面）
- 仪表盘和提示词自动适配资产类型
- 标的身份解析防止 Agent 幻觉错误公司

**示例**：

```python
# 股票：4 个分析师全部启用
graph.propagate("AAPL", "2025-06-14", "stock")

# 加密货币：自动排除 Fundamentals Analyst
graph.propagate("BTC-USD", "2025-06-14", "crypto")
# → Fundamentals Analyst 被跳过，其余 3 个分析师正常运行
```

---

## 能力总结

| 能力 | 对交易的帮助 |
|------|-------------|
| 多层级决策 | 提供完整交易方案（入场/止损/仓位/目标） |
| 多空辩论 | 避免单一视角，引用数据论证 |
| 三方风控 | 全面评估风险，防止极端观点 |
| 专业分析团队 | 4 维度分析（技术/情绪/消息/基本面） |
| 延迟反思 | 从历史决策学习，越用越准 |
| 多数据源路由 | 数据源容错，支持厂商组合 |
| 多 LLM 支持 | 灵活选择模型，国产 LLM 直接可用 |
| 崩溃恢复 | 长时间分析不白费 |
| 防幻觉验证 | 防止基于错误数据决策 |
| 多语言输出 | 中文分析报告 |
| CLI 交互 | 非技术人员可用 |
| 资产自适应 | 自动调整分析师团队 |
