# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

TradingAgents is a multi-agent LLM financial trading framework (v0.2.5) by Tauric Research. It simulates real-world trading firms by deploying specialized LLM-powered agents that collaboratively evaluate market conditions and produce trading decisions (Buy/Hold/Sell). Built with LangGraph on Python 3.10+.

## Commands

```bash
# Install (production)
pip install .

# Install (development with dev tools)
pip install -e ".[dev]"

# Run all tests
pytest -q

# Run specific test markers
pytest -m unit              # fast isolated tests
pytest -m integration       # tests requiring external services
pytest -m smoke             # quick sanity checks

# Run a single test file
pytest tests/test_foo.py

# Lint (ruff, line-length=100, target py310)
ruff check .

# Run CLI
tradingagents               # after pip install
python -m cli.main          # alternative

# Programmatic usage
python main.py              # runs propagation on NVDA 2024-05-10

# Docker
cp .env.example .env        # add API keys
docker compose run --rm tradingagents
docker compose --profile ollama run --rm tradingagents-ollama
```

## Architecture

### LangGraph State Graph Flow

```
START → [Analyst 1..N] → [Bull ↔ Bear debate] → [Research Manager]
       → [Trader] → [Aggressive ↔ Conservative ↔ Neutral risk debate]
       → [Portfolio Manager] → END
```

Each analyst node loops with a ToolNode until no tool calls remain (per-analyst tool loop).

### Key Modules

| Module | Purpose |
|--------|---------|
| `tradingagents/graph/` | LangGraph workflow orchestration — `TradingAgentsGraph` is the main entry point |
| `tradingagents/agents/` | All LLM agent definitions (analysts, researchers, trader, risk managers, portfolio manager) |
| `tradingagents/agents/utils/` | Shared state (`AgentState`), tool definitions, structured output helpers, memory |
| `tradingagents/agents/schemas.py` | Pydantic schemas for structured output (ResearchPlan, TraderProposal, PortfolioDecision, SentimentReport) |
| `tradingagents/dataflows/` | Vendor-agnostic data fetching — `interface.py` routes to yfinance/Alpha Vantage/FRED/etc. |
| `tradingagents/llm_clients/` | Multi-provider LLM abstraction — factory creates OpenAI/Anthropic/Google/Azure/Bedrock clients |
| `tradingagents/llm_clients/capabilities.py` | Per-model capability table handling provider quirks (tool_choice, json_schema, reasoning) |
| `tradingagents/default_config.py` | `DEFAULT_CONFIG` dict with `TRADINGAGENTS_*` env-var overrides |
| `cli/main.py` | Typer + Rich interactive CLI with live dashboard |

### Data Flow

- `interface.py` is the vendor-routing facade — tool calls route through it to the configured vendor
- `data_vendors` and `tool_vendors` config keys control which vendor serves each data category
- Ordered fallback chains are supported (e.g., `"yfinance,alpha_vantage"`)
- Ticker is resolved to company identity once via yfinance before any agent runs

### LLM Client Factory

- `create_llm_client()` in `llm_clients/factory.py` uses lazy imports — test collection doesn't require API keys
- OpenAI-compatible providers (DeepSeek, Qwen, Ollama, etc.) are routed through `openai_client.py`
- Each client extends `BaseLLMClient` ABC with a `normalize_content()` method
- Provider quirks are declared in `capabilities.py`, not scattered if/else ladders

### Structured Output

Decision agents use Pydantic schemas bound per-provider:
- OpenAI: `json_schema` response format
- Anthropic: tool-use binding
- Gemini: `response_schema`
- Render functions convert structured output back to markdown for downstream agents

### Configuration

- `DEFAULT_CONFIG` in `default_config.py` holds all defaults
- `TRADINGAGENTS_*` env vars override config keys with automatic type coercion
- CLI skips interactive prompts when the corresponding env var is set
- See `.env.example` for all supported API keys and overrides

### Memory / Reflection

- Decisions persist to `~/.tradingagents/memory/trading_memory.md`
- On re-run for same ticker: realized returns are fetched, reflections generated, and context injected into Portfolio Manager prompt
- Opt-in checkpointing via SQLite per-ticker databases (crash resume from last successful node)

## Development Notes

- ruff is the linter (line-length=100, target Python 3.10)
- Tests use pytest with markers: `unit`, `integration`, `smoke`
- CI lints only changed Python files in PRs
- The `[bedrock]` optional extra installs `langchain-aws` for AWS Bedrock support
