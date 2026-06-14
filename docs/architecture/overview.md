# TradingAgents 架构概览

> 版本: 0.2.5 | Python >= 3.10 | 基于 LangGraph 的多 Agent LLM 金融交易框架

## 1. 项目定位

TradingAgents 是由 Tauric Research 开发的多 Agent LLM 金融交易框架。它模拟真实交易公司的运作方式，部署多个专业化 LLM Agent 协作评估市场条件，最终产出交易决策（Buy/Hold/Sell）。

核心设计理念：
- **多角色协作**：分析师、研究员、交易员、风控、投资组合经理各司其职
- **辩论驱动决策**：多空辩论 + 三方风控辩论，避免单一视角偏差
- **厂商无关数据层**：统一接口路由到 yfinance / Alpha Vantage / FRED 等数据源
- **多 LLM 提供商**：支持 17+ LLM 提供商，深度推理与快速推理分离

## 2. 顶层目录结构

```
TradingAgents/
├── tradingagents/          # 核心库
│   ├── agents/             # 所有 LLM Agent 定义
│   ├── dataflows/          # 数据厂商路由与实现
│   ├── graph/              # LangGraph 工作流编排
│   └── llm_clients/        # 多提供商 LLM 抽象层
├── cli/                    # 交互式 CLI（Typer + Rich）
├── tests/                  # 测试套件（44 个测试文件）
├── scripts/                # 辅助脚本
├── assets/                 # 文档图片
├── main.py                 # 编程入口（演示）
├── pyproject.toml          # 构建配置、依赖
├── Dockerfile              # 多阶段 Docker 构建
└── docker-compose.yml      # 3 个服务定义
```

## 3. 核心模块关系

```
┌─────────────────────────────────────────────────────────┐
│                     CLI / 编程入口                        │
│              cli/main.py  |  main.py                     │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│              TradingAgentsGraph                          │
│         graph/trading_graph.py (顶层编排器)               │
│                                                         │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌───────────┐  │
│  │Propagator│ │GraphSetup│ │Reflector │ │SignalProc. │  │
│  └──────────┘ └──────────┘ └──────────┘ └───────────┘  │
└──────────────────────┬──────────────────────────────────┘
                       │ 构建并编译 LangGraph StateGraph
                       ▼
┌─────────────────────────────────────────────────────────┐
│                LangGraph StateGraph                      │
│                                                         │
│  Analysts ──→ Bull/Bear Debate ──→ Research Manager      │
│       ──→ Trader ──→ Risk Debate ──→ Portfolio Manager   │
└───────┬──────────────┬──────────────────┬───────────────┘
        │              │                  │
        ▼              ▼                  ▼
┌──────────────┐ ┌──────────┐ ┌────────────────────┐
│  agents/     │ │  agents/ │ │   llm_clients/     │
│  工具定义     │ │  Agent   │ │   LLM 客户端抽象    │
│  & schemas   │ │  闭包工厂 │ │   17+ 提供商       │
└──────┬───────┘ └──────────┘ └────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────┐
│               dataflows/ 数据层                           │
│                                                         │
│  interface.py (路由门面)                                  │
│    ├── y_finance.py        (无需 API key)                │
│    ├── alpha_vantage_*.py  (需要 AV_API_KEY)            │
│    ├── fred.py             (需要 FRED_API_KEY)           │
│    ├── polymarket.py       (无需 API key)                │
│    ├── reddit.py           (无需 API key, RSS)           │
│    ├── stocktwits.py       (无需 API key)                │
│    └── yfinance_news.py    (无需 API key)                │
└─────────────────────────────────────────────────────────┘
```

## 4. 数据流全景

### 4.1 LLM 客户端初始化流

```
用户配置 (config dict)
  → DEFAULT_CONFIG + TRADINGAGENTS_* 环境变量覆盖
  → set_config() 推送到 dataflows 模块
  → _get_provider_kwargs() 提取提供商特定参数
  → create_llm_client() × 2（deep_thinking + quick_thinking）
  → 工厂懒加载对应提供商客户端
  → get_llm() 返回 Normalized* LangChain Chat 实例
```

### 4.2 Agent 执行流

```
用户调用 propagate(ticker, date)
  → 解析标的身份（yfinance LRU 缓存）
  → 构建初始 AgentState
  → 可选：SQLite 检查点恢复
  → LangGraph 编译图执行
  → 4 个 Analyst 顺序执行（工具循环）
  → Bull/Bear 辩论（默认 2 轮）
  → Research Manager 裁决（结构化输出）
  → Trader 生成交易提案（结构化输出）
  → 三方风控辩论（默认 3 轮）
  → Portfolio Manager 最终决策（结构化输出）
  → 信号处理提取评级
  → 存储决策到记忆日志
  → 返回最终决策
```

### 4.3 数据请求流

```
Agent LLM 调用工具（如 get_stock_data）
  → @tool 函数调用 route_to_vendor()
  → 解析工具类别 → 确定厂商链
  → 按链顺序尝试厂商实现
  → 成功：返回格式化数据字符串
  → 限流/未配置：跳到下一个厂商
  → 无数据：返回 NO_DATA_AVAILABLE: 哨兵字符串
  → 全部失败：抛出第一个真实错误
```

## 5. 关键设计决策

| 决策 | 选择 | 原因 |
|------|------|------|
| Agent 构建模式 | 闭包工厂函数 `create_<role>(llm)` | 无类开销，LLM 绑定在闭包中 |
| 结构化输出 | Pydantic schema + 渲染回 markdown | 主产物是散文，结构化用于指令和验证 |
| 辩论终止 | 计数器阈值（`count >= 2*rounds`） | 简单可控，避免无限循环 |
| 消息隔离 | 每个 Analyst 后清空 messages | 防止后续 Agent 被前序工具调用历史干扰 |
| 数据回退 | 有序厂商链，不静默回退 | 用户显式选择的数据源优先级受尊重 |
| 检查点 | SQLite per-ticker + SHA256 thread_id | 崩溃恢复，相同 ticker+date 总是恢复同一点 |
| 记忆系统 | 追加式 markdown 日志 + 延迟反思 | 简单持久化，下次运行时生成反思注入上下文 |

## 6. 入口点

| 入口 | 位置 | 运行方式 |
|------|------|----------|
| CLI（主要） | `cli/main.py` | `tradingagents` 或 `python -m cli.main` |
| 编程演示 | `main.py` | `python main.py` |
| Docker | `Dockerfile` | `docker compose run --rm tradingagents` |
| 结构化输出冒烟测试 | `scripts/smoke_structured_output.py` | `python scripts/smoke_structured_output.py <provider>` |

## 7. 支持的 LLM 提供商（17+）

| 提供商 | 客户端类 | 特殊处理 |
|--------|----------|----------|
| OpenAI | OpenAIClient | Responses API（原生端点） |
| Anthropic | AnthropicClient | effort 参数（Opus 4.5+/Sonnet 4.5+） |
| Google | GoogleClient | thinking_level 映射（Gemini 3 vs 2.5） |
| Azure OpenAI | AzureOpenAIClient | 部署名称配置 |
| Amazon Bedrock | BedrockClient | AWS 凭证链，可选 [bedrock] extra |
| DeepSeek | OpenAIClient | reasoning_content 往返 |
| MiniMax | OpenAIClient | reasoning_split 注入 |
| Qwen | OpenAIClient | 国际/中国双区域 |
| GLM (智谱) | OpenAIClient | 国际/中国双区域 |
| xAI | OpenAIClient | - |
| OpenRouter | OpenAIClient | - |
| Mistral | OpenAIClient | - |
| Kimi/Moonshot | OpenAIClient | - |
| Groq | OpenAIClient | - |
| NVIDIA NIM | OpenAIClient | - |
| Ollama | OpenAIClient | 本地部署，无需 key |
| OpenAI 兼容 | OpenAIClient | vLLM/LM Studio/llama.cpp |

## 8. 测试与 CI

- **测试框架**: pytest，标记: `unit`、`integration`、`smoke`
- **测试文件**: 44 个，覆盖数据层、Agent、配置、提供商、CLI
- **CI**: GitHub Actions，Python 3.10-3.13 矩阵，ruff lint
- **Lint**: ruff（line-length=100，target py310）
