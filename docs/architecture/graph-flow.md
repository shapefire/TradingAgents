# LangGraph 状态图工作流

> 核心编排层详解：节点、边、条件路由、辩论机制

## 1. 图编排器：TradingAgentsGraph

**文件**: `tradingagents/graph/trading_graph.py`

`TradingAgentsGraph` 是顶层编排器，负责组装所有组件并驱动图执行。

### 1.1 构造函数

```python
TradingAgentsGraph(
    selected_analysts=["market", "social", "news", "fundamentals"],
    debug=False,
    config=None,
    callbacks=None,
)
```

构造过程：
1. 创建两个 LLM 客户端：`deep_thinking_llm`（Research Manager + Portfolio Manager）和 `quick_thinking_llm`（其余所有 Agent）
2. 提取提供商特定参数（Google thinking_level、OpenAI reasoning_effort、Anthropic effort、temperature）
3. 创建 4 个 ToolNode（每个 Analyst 一个）
4. 实例化辅助组件：ConditionalLogic、GraphSetup、Propagator、Reflector、SignalProcessor
5. 调用 `setup_graph()` 构建未编译的 StateGraph
6. 编译图（无检查点）

### 1.2 公开方法

| 方法 | 用途 |
|------|------|
| `propagate(company_name, trade_date, asset_type)` | 主入口，可选启用检查点 |
| `_run_graph(company_name, trade_date, asset_type)` | 创建初始状态，调用图，记录结果 |
| `process_signal(full_signal)` | 从最终决策提取 5 级评级 |
| `_resolve_pending_entries(ticker)` | 获取历史决策的实际收益，生成反思 |
| `_resolve_benchmark(ticker)` | 选择基准 ticker（如 SPY）用于 alpha 计算 |

## 2. 状态模型：AgentState

**文件**: `tradingagents/agents/utils/agent_states.py`

### 2.1 AgentState（全局图状态）

继承自 LangGraph 的 `MessagesState`，包含 `messages: list` 字段。

| 字段 | 类型 | 用途 |
|------|------|------|
| `messages` | `list` | 共享消息列表，所有 Agent 读写 |
| `company_of_interest` | `str` | 被分析的 ticker |
| `asset_type` | `str` | `"stock"` 或 `"crypto"` |
| `instrument_context` | `str` | 运行开始时解析的标的身份 |
| `trade_date` | `str` | 被分析的日期 |
| `sender` | `str` | 最后写入的 Agent |
| `market_report` | `str` | Market Analyst 输出 |
| `sentiment_report` | `str` | Sentiment Analyst 输出 |
| `news_report` | `str` | News Analyst 输出 |
| `fundamentals_report` | `str` | Fundamentals Analyst 输出 |
| `investment_debate_state` | `InvestDebateState` | 多空辩论累积器 |
| `investment_plan` | `str` | Research Manager 输出 |
| `trader_investment_plan` | `str` | Trader 输出 |
| `risk_debate_state` | `RiskDebateState` | 三方风控辩论累积器 |
| `final_trade_decision` | `str` | Portfolio Manager 输出 |
| `past_context` | `str` | 历史记忆上下文 |

### 2.2 InvestDebateState（多空辩论状态）

| 字段 | 用途 |
|------|------|
| `bull_history` | 看多方论点累积 |
| `bear_history` | 看空方论点累积 |
| `history` | 完整辩论历史 |
| `current_response` | 最新论点（以 "Bull" 或 "Bear" 前缀，用于路由） |
| `judge_decision` | 裁决输出 |
| `count` | 回合计数器，达到 `2 * max_debate_rounds` 时终止 |

### 2.3 RiskDebateState（风控辩论状态）

| 字段 | 用途 |
|------|------|
| `aggressive_history` | 激进方论点累积 |
| `conservative_history` | 保守方论点累积 |
| `neutral_history` | 中立方论点累积 |
| `history` | 完整辩论历史 |
| `latest_speaker` | 最后发言者名称前缀（用于路由） |
| `current_*_response` | 各方最新论点 |
| `judge_decision` | 裁决输出 |
| `count` | 回合计数器，达到 `3 * max_risk_discuss_rounds` 时终止 |

## 3. 图构建：GraphSetup

**文件**: `tradingagents/graph/setup.py`

### 3.1 Analyst 执行计划

**文件**: `tradingagents/graph/analyst_execution.py`

每个 Analyst key 映射到一个 `AnalystNodeSpec`，包含四个命名节点：

| Key | agent_node | tool_node | clear_node | report_key |
|-----|-----------|-----------|------------|------------|
| `"market"` | `"Market Analyst"` | `"tools_market"` | `"Msg Clear Market"` | `"market_report"` |
| `"social"` | `"Sentiment Analyst"` | `"tools_social"` | `"Msg Clear Sentiment"` | `"sentiment_report"` |
| `"news"` | `"News Analyst"` | `"tools_news"` | `"Msg Clear News"` | `"news_report"` |
| `"fundamentals"` | `"Fundamentals Analyst"` | `"tools_fundamentals"` | `"Msg Clear Fundamentals"` | `"fundamentals_report"` |

### 3.2 节点注册

对于每个选中的 Analyst，注册 3 个节点：
1. **Agent 节点**：LLM 调用 + 工具绑定
2. **ToolNode**：执行该 Analyst 的数据获取工具
3. **Clear 节点**：清空消息列表，注入上下文锚定占位符

额外注册 8 个节点：Bull Researcher、Bear Researcher、Research Manager、Trader、Aggressive/Conservative/Neutral Analyst、Portfolio Manager。

### 3.3 完整节点表（默认 4 Analyst 配置）

| # | 节点名 | 类型 | LLM | 工具 |
|---|--------|------|-----|------|
| 1 | Market Analyst | Analyst | quick | get_stock_data, get_indicators, get_verified_market_snapshot |
| 2 | tools_market | ToolNode | — | 执行市场工具 |
| 3 | Msg Clear Market | 消息清理 | — | 清空消息 |
| 4 | Sentiment Analyst | Analyst | quick | 无（预取数据） |
| 5 | Msg Clear Sentiment | 消息清理 | — | 清空消息 |
| 6 | News Analyst | Analyst | quick | get_news, get_global_news, get_macro_indicators, get_prediction_markets |
| 7 | tools_news | ToolNode | — | 执行新闻工具 |
| 8 | Msg Clear News | 消息清理 | — | 清空消息 |
| 9 | Fundamentals Analyst | Analyst | quick | get_fundamentals, get_balance_sheet, get_cashflow, get_income_statement |
| 10 | tools_fundamentals | ToolNode | — | 执行基本面工具 |
| 11 | Msg Clear Fundamentals | 消息清理 | — | 清空消息 |
| 12 | Bull Researcher | Debater | quick | 无 |
| 13 | Bear Researcher | Debater | quick | 无 |
| 14 | Research Manager | Judge | deep | 无（结构化输出） |
| 15 | Trader | Producer | quick | 无（结构化输出） |
| 16 | Aggressive Analyst | Debater | quick | 无 |
| 17 | Conservative Analyst | Debater | quick | 无 |
| 18 | Neutral Analyst | Debater | quick | 无 |
| 19 | Portfolio Manager | Judge | deep | 无（结构化输出） |

## 4. 边拓扑（完整）

```
START ──→ Market Analyst
Market Analyst ──[has_tool_calls?]──→ tools_market / Msg Clear Market
tools_market ──→ Market Analyst
Msg Clear Market ──→ Sentiment Analyst

Sentiment Analyst ──[has_tool_calls?]──→ tools_social / Msg Clear Sentiment
(Sentiment Analyst 不使用工具，总是到 Msg Clear Sentiment)
Msg Clear Sentiment ──→ News Analyst

News Analyst ──[has_tool_calls?]──→ tools_news / Msg Clear News
tools_news ──→ News Analyst
Msg Clear News ──→ Fundamentals Analyst

Fundamentals Analyst ──[has_tool_calls?]──→ tools_fundamentals / Msg Clear Fundamentals
tools_fundamentals ──→ Fundamentals Analyst
Msg Clear Fundamentals ──→ Bull Researcher

Bull Researcher ──[count < 2*max_debate_rounds?]──→ Bear Researcher / Research Manager
Bear Researcher ──[count < 2*max_debate_rounds?]──→ Bull Researcher / Research Manager

Research Manager ──→ Trader
Trader ──→ Aggressive Analyst

Aggressive Analyst ──[count < 3*max_risk_rounds?]──→ Conservative Analyst / Portfolio Manager
Conservative Analyst ──[count < 3*max_risk_rounds?]──→ Neutral Analyst / Portfolio Manager
Neutral Analyst ──[count < 3*max_risk_rounds?]──→ Aggressive Analyst / Portfolio Manager

Portfolio Manager ──→ END
```

## 5. 工具循环机制（Analyst 节点）

每个 Analyst（Market、News、Fundamentals）遵循相同的模式：

```
┌─────────────────────────────────────────────┐
│  Analyst LLM 调用（bind_tools）              │
│  输入：共享 messages 列表                     │
└──────────────┬──────────────────────────────┘
               │
               ▼
        ┌──────────────┐
        │ 有 tool_calls? │
        └──────┬───────┘
          Yes  │  No
               │    │
               ▼    ▼
    ┌──────────┐  ┌──────────────┐
    │ ToolNode │  │ Msg Clear    │
    │ 执行工具  │  │ 清空消息      │
    └────┬─────┘  │ 注入占位符    │
         │        └──────┬───────┘
         │               │
         ▼               ▼
    回到 Analyst      下一个 Analyst
```

**消息清理节点**的作用：
- 删除所有 messages
- 注入单条上下文锚定的 HumanMessage
- 防止后续 Agent 被前序工具调用历史干扰

## 6. 辩论机制

### 6.1 多空辩论（Bull/Bear Debate）

**参与者**：Bull Researcher、Bear Researcher、Research Manager（裁决）

**流程**：
1. Bull Researcher 首先运行，读取 4 份 Analyst 报告，产出看多论点（前缀 `"Bull Analyst:"`）
2. 条件边检查：
   - `count >= 2 * max_debate_rounds`（默认 2）→ 终止到 Research Manager
   - `current_response` 以 `"Bull"` 开头 → 路由到 Bear Researcher
   - 否则 → 路由到 Bull Researcher
3. Bear Researcher 读取完整辩论历史和 Bull 的最新论点，产出看空论点（前缀 `"Bear Analyst:"`）
4. 循环直到计数器达到阈值

**Research Manager（裁决者）**：
- 使用 `deep_thinking_llm`
- 结构化输出 `ResearchPlan`（recommendation, rationale, strategic_actions）
- 写入 `investment_plan` 和 `investment_debate_state.judge_decision`

### 6.2 风控辩论（三方）

**参与者**：Aggressive Analyst、Conservative Analyst、Neutral Analyst、Portfolio Manager（裁决）

**轮转顺序**：Aggressive → Conservative → Neutral → Aggressive → ...

**流程**：
1. Trader 先运行（在 Research Manager 之后），产出具体交易提案
2. Aggressive Analyst 首先运行，读取所有报告 + Trader 决策
3. 条件边检查：
   - `count >= 3 * max_risk_discuss_rounds`（默认 3）→ 终止到 Portfolio Manager
   - `latest_speaker` 以 `"Aggressive"` 开头 → 路由到 Conservative
   - 以 `"Conservative"` 开头 → 路由到 Neutral
   - 否则 → 路由回 Aggressive
4. 每个参与者读取完整历史 + 其他两方的最新回复
5. 循环直到计数器达到阈值

**Portfolio Manager（最终裁决者）**：
- 使用 `deep_thinking_llm`
- 结构化输出 `PortfolioDecision`（rating, executive_summary, investment_thesis, price_target, time_horizon）
- 读取 Research Manager 计划、Trader 提案、风控辩论历史、`past_context`（历史记忆）
- 产出 `final_trade_decision`

## 7. 检查点/恢复机制

**文件**: `tradingagents/graph/checkpointer.py`

当 `config["checkpoint_enabled"]` 为 true 时：

1. 打开 per-ticker 的 `SqliteSaver`
2. 重新编译图：`self.graph = self.workflow.compile(checkpointer=saver)`
3. `thread_id` = `SHA256(f"{TICKER}:{DATE}")[:16]`（确定性计算）
4. 崩溃恢复：相同 ticker+date 从最后完成的节点继续
5. 成功完成后：清除检查点行，重新编译无检查点图

**存储位置**: `{data_cache_dir}/checkpoints/{TICKER}.db`

## 8. 反思与记忆系统

**文件**: `tradingagents/graph/reflection.py`

**Reflector** 类：
- 使用 `quick_thinking_llm` 生成 2-4 句反思
- 内容：方向正确性（含 alpha）、论题成败、具体教训

**记忆流程**：
1. `_run_graph()` 完成后 → `memory_log.store_decision()` 存储为 pending 条目
2. 下次运行同 ticker → `_resolve_pending_entries()` 获取实际收益，生成反思，批量更新
3. `memory_log.get_past_context()` 将历史教训注入 Portfolio Manager 的 prompt 作为 `past_context`

## 9. 信号处理

**文件**: `tradingagents/graph/signal_processing.py`

`SignalProcessor` 从 Portfolio Manager 的 markdown 输出提取 5 级评级：
- 委托给 `parse_rating()`（确定性启发式解析）
- 读取 `**Rating**: X` 格式的头部
- 无需 LLM 调用
- 评级：Buy / Overweight / Hold / Underweight / Sell

## 10. 配置默认值

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `max_debate_rounds` | 1 | 多空辩论轮数（实际 2 轮：1 bull + 1 bear） |
| `max_risk_discuss_rounds` | 1 | 风控辩论轮数（实际 3 轮：aggressive + conservative + neutral） |
| `max_recur_limit` | 100 | LangGraph 递归限制 |
| `analyst_concurrency_limit` | 1 | Analyst 顺序执行 |
| `checkpoint_enabled` | false | 检查点开关 |
