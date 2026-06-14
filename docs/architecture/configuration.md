# 配置系统

> 默认配置、环境变量覆盖、数据流配置、提供商特定参数

## 1. 配置架构概览

```
DEFAULT_CONFIG (default_config.py)
  │
  ├── TRADINGAGENTS_* 环境变量覆盖
  │
  ▼
TradingAgentsGraph.__init__(config=...)
  │
  ├── set_config(config) → dataflows 模块级单例
  │
  ├── _get_provider_kwargs() → 提供商特定参数
  │
  └── create_llm_client() × 2
```

## 2. 默认配置

**文件**: `tradingagents/default_config.py`

`DEFAULT_CONFIG` 字典在模块加载时填充，通过 `_apply_env_overrides()` 支持环境变量覆盖。

### 2.1 核心配置项

| 配置键 | 默认值 | 类型 | 说明 |
|--------|--------|------|------|
| `llm_provider` | `"openai"` | str | LLM 提供商 |
| `deep_think_l` | `"gpt-5"` | str | 深度推理模型 |
| `quick_think_l` | `"gpt-5-mini"` | str | 快速推理模型 |
| `backend_url` | `""` | str | 提供商后端 URL |
| `output_language` | `"english"` | str | 输出语言 |
| `max_debate_rounds` | `1` | int | 多空辩论轮数 |
| `max_risk_discuss_rounds` | `1` | int | 风控辩论轮数 |
| `max_recur_limit` | `100` | int | LangGraph 递归限制 |
| `analyst_concurrency_limit` | `1` | int | Analyst 并发限制 |
| `checkpoint_enabled` | `false` | bool | 检查点开关 |
| `benchmark_ticker` | `"SPY"` | str | 基准 ticker |
| `temperature` | `None` | float | LLM 温度 |
| `data_vendors` | `{...}` | dict | 数据厂商配置 |
| `tool_vendors` | `{}` | dict | 工具级厂商覆盖 |

### 2.2 数据厂商默认配置

```python
data_vendors = {
    "core_stock_apis": "yfinance",
    "technical_indicators": "yfinance",
    "fundamental_data": "yfinance",
    "news_data": "yfinance",
    "macro_data": "fred",
    "prediction_markets": "polymarket",
}
```

### 2.3 提供商特定思考配置

| 配置键 | 默认值 | 适用提供商 |
|--------|--------|-----------|
| `google_thinking_level` | None | Google Gemini |
| `openai_reasoning_effort` | None | OpenAI 推理模型 |
| `anthropic_effort` | None | Anthropic Claude |

## 3. 环境变量覆盖

### 3.1 自动覆盖映射

`_ENV_OVERRIDES` 字典定义环境变量到配置键的映射：

| 环境变量 | 配置键 | 类型 |
|----------|--------|------|
| `TRADINGAGENTS_LLM_PROVIDER` | `llm_provider` | str |
| `TRADINGAGENTS_DEEP_THINK_LLM` | `deep_think_l` | str |
| `TRADINGAGENTS_QUICK_THINK_LLM` | `quick_think_l` | str |
| `TRADINGAGENTS_LLM_BACKEND_URL` | `backend_url` | str |
| `TRADINGAGENTS_OUTPUT_LANGUAGE` | `output_language` | str |
| `TRADINGAGENTS_MAX_DEBATE_ROUNDS` | `max_debate_rounds` | int |
| `TRADINGAGENTS_MAX_RISK_ROUNDS` | `max_risk_discuss_rounds` | int |
| `TRADINGAGENTS_CHECKPOINT_ENABLED` | `checkpoint_enabled` | bool |
| `TRADINGAGENTS_BENCHMARK_TICKER` | `benchmark_ticker` | str |
| `TRADINGAGENTS_TEMPERATURE` | `temperature` | float |

### 3.2 类型强制

- 布尔值：从 `"true"/"1"/"yes"/"on"` 解析
- 整数/浮点数：从字符串转换
- 字符串：直接使用

### 3.3 直接处理的环境变量

| 环境变量 | 说明 |
|----------|------|
| `TRADINGAGENTS_RESULTS_DIR` | 结果输出目录 |
| `TRADINGAGENTS_CACHE_DIR` | 缓存目录 |
| `TRADINGAGENTS_MEMORY_LOG_PATH` | 记忆日志路径 |

## 4. 数据流配置

**文件**: `tradingagents/dataflows/config.py`

### 4.1 模块级单例

```python
_config = DEFAULT_CONFIG.copy()

def set_config(config: dict) -> None:
    """合并配置到模块级单例"""
    # dict 值键一级深度合并（保留其他类别默认值）
    # 标量直接替换

def get_config() -> dict:
    """返回配置的深拷贝"""
```

### 4.2 配置推送

`TradingAgentsGraph.__init__()` 调用 `set_config(self.config)` 将用户配置推送到 dataflows 模块。

### 4.3 配置合并行为

- **dict 值**（如 `data_vendors`）：一级深度合并，部分更新保留其他类别默认值
- **标量值**：直接替换

## 5. CLI 配置

### 5.1 CLI 配置字典

**文件**: `cli/config.py`

```python
CLI_CONFIG = {
    "announcements_url": "https://api.tauric.ai/announcements",
    "announcements_timeout": 5,
    "announcements_fallback": "No announcements available.",
}
```

### 5.2 交互式配置

**文件**: `cli/utils.py`

CLI 提供交互式提示配置：
- Ticker 输入
- Analyst 选择
- LLM 提供商选择
- 模型选择
- API key 管理
- 研究深度
- 输出语言
- 区域选择（Qwen/GLM/MiniMax）
- Ollama 端点确认

## 6. Docker 配置

### 6.1 环境变量文件

| 文件 | 用途 |
|------|------|
| `.env.example` | 16 个 LLM 提供商 API key + FRED key + 10 个 TRADINGAGENTS_* 覆盖 |
| `.env.enterprise.example` | 3 个 Azure OpenAI 变量（endpoint, API key, deployment name） |

### 6.2 Docker Compose 服务

**文件**: `docker-compose.yml`

| 服务 | 说明 |
|------|------|
| `tradingagents` | 默认服务 |
| `ollama` | Ollama 镜像 |
| `tradingagents-ollama` | LLM_PROVIDER=ollama，depends_on ollama |

### 6.3 Dockerfile

**文件**: `Dockerfile`

多阶段构建：
1. 构建阶段：Python 3.12-slim
2. 运行阶段：非 root 用户 `appuser`
3. 入口点：`tradingagents`

## 7. 测试配置

### 7.1 pytest 配置

**文件**: `pyproject.toml`

```toml
[tool.pytest.ini_options]
markers = [
    "unit: fast isolated tests",
    "integration: tests requiring external services",
    "smoke: quick sanity checks",
]
```

### 7.2 ruff 配置

```toml
[tool.ruff]
line-length = 100
target-version = "py310"
```

## 8. CI/CD 配置

**文件**: `.github/workflows/ci.yml`

- 测试矩阵：Python 3.10-3.13
- 干净安装冒烟测试
- ruff lint（PR 时仅检查变更文件）

## 9. 配置使用示例

### 9.1 编程方式配置

```python
from tradingagents.graph import TradingAgentsGraph

config = {
    "llm_provider": "anthropic",
    "deep_think_l": "claude-opus-4-5",
    "quick_think_l": "claude-sonnet-4-5",
    "max_debate_rounds": 2,
    "data_vendors": {
        "core_stock_apis": "yfinance,alpha_vantage",
        "news_data": "alpha_vantage",
    },
}

graph = TradingAgentsGraph(config=config)
result = graph.propagate("AAPL", "2025-01-15", "stock")
```

### 9.2 环境变量配置

```bash
export TRADINGAGENTS_LLM_PROVIDER=anthropic
export TRADINGAGENTS_DEEP_THINK_LLM=claude-opus-4-5
export TRADINGAGENTS_QUICK_THINK_LLM=claude-sonnet-4-5
export TRADINGAGENTS_MAX_DEBATE_ROUNDS=2
export TRADINGAGENTS_CHECKPOINT_ENABLED=true
```

### 9.3 CLI 配置

```bash
# 交互式配置
tradingagents

# 环境变量驱动（跳过提示）
TRADINGAGENTS_LLM_PROVIDER=openai TRADINGAGENTS_DEEP_THINK_LLM=gpt-5 tradingagents
```

## 10. 配置优先级

```
编程 config 参数 > 环境变量 > DEFAULT_CONFIG 默认值
```

- 编程传入的 config 字典覆盖一切
- 环境变量覆盖 DEFAULT_CONFIG
- DEFAULT_CONFIG 提供所有默认值
