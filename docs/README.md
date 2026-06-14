# TradingAgents 项目文档

> 本文档为 TradingAgents v0.2.5 的参考文档，供二次开发使用。

## 文档结构

```
docs/
├── README.md                          # 本文档（索引）
├── architecture/                      # 架构文档
│   ├── overview.md                    # 架构概览（入口）
│   ├── graph-flow.md                  # LangGraph 状态图工作流
│   ├── agents.md                      # Agent 定义与角色
│   ├── data-layer.md                  # 数据层架构
│   ├── llm-clients.md                 # LLM 客户端架构
│   └── configuration.md               # 配置系统
└── capabilities/                      # 能力与改造文档
    ├── trading-capabilities.md        # 交易能力深度分析
    └── ashare-adaptation.md           # A 股市场改造指南
```

## 阅读顺序

### 架构文档（从这里开始）

1. **[overview.md](architecture/overview.md)** — 项目定位、模块关系、数据流全景
2. **[graph-flow.md](architecture/graph-flow.md)** — LangGraph 状态图：节点、边、条件路由、辩论机制
3. **[agents.md](architecture/agents.md)** — 所有 Agent 的详细定义：角色、工具、提示词、输出格式
4. **[data-layer.md](architecture/data-layer.md)** — 数据厂商路由、适配器实现、前瞻防护
5. **[llm-clients.md](architecture/llm-clients.md)** — LLM 客户端抽象、17+ 提供商集成、能力矩阵
6. **[configuration.md](architecture/configuration.md)** — 配置系统、环境变量、Docker 部署

### 能力与改造文档

7. **[trading-capabilities.md](capabilities/trading-capabilities.md)** — 12 项核心能力详解（含交易场景示例）
8. **[ashare-adaptation.md](capabilities/ashare-adaptation.md)** — A 股市场改造路线图（P0-P3 优先级）

## 二次开发指南

### 扩展新的 Agent

1. 在 `tradingagents/agents/` 下创建新模块
2. 实现 `create_<role>(llm)` 工厂函数，返回 LangGraph 节点函数
3. 如需结构化输出，使用 `bind_structured()` + `invoke_structured_or_freetext()`
4. 在 `tradingagents/agents/__init__.py` 中导出工厂函数
5. 在 `tradingagents/graph/setup.py` 中注册节点和边

### 扩展新的数据厂商

1. 在 `tradingagents/dataflows/` 下创建新适配器模块
2. 实现与 `VENDOR_METHODS` 中方法签名兼容的函数
3. 在 `interface.py` 的 `VENDOR_METHODS` 中注册新厂商
4. 在 `default_config.py` 的 `data_vendors` 中添加默认配置

### 扩展新的 LLM 提供商

1. 在 `tradingagents/llm_clients/` 下创建新客户端模块
2. 继承 `BaseLLMClient`，实现 `get_llm()` 和 `validate_model()`
3. 在 `factory.py` 的 `create_llm_client()` 中添加路由
4. 在 `api_key_env.py` 中添加 API key 环境变量映射
5. 在 `model_catalog.py` 中添加模型目录
6. 在 `capabilities.py` 中添加能力声明（如有特殊怪癖）

### 添加新的辩论角色

1. 在 `tradingagents/agents/` 对应目录下创建新模块
2. 实现 `create_<role>(llm)` 工厂函数
3. 在 `tradingagents/graph/setup.py` 中注册节点
4. 修改 `conditional_logic.py` 中的路由条件
5. 更新 `agent_states.py` 中的状态定义（如需新字段）

## 技术栈

| 组件 | 技术 |
|------|------|
| 框架 | LangGraph (LangChain 生态) |
| 语言 | Python >= 3.10 |
| LLM 抽象 | LangChain Chat Models |
| 结构化输出 | Pydantic schemas |
| CLI | Typer + Rich + questionary |
| 数据缓存 | CSV 文件 + SQLite |
| 检查点 | SQLite (LangGraph SqliteSaver) |
| 测试 | pytest |
| Lint | ruff |
| 容器化 | Docker + Docker Compose |
| CI/CD | GitHub Actions |

## 版本历史

- **v0.2.5** — 当前版本
- 详见 `CHANGELOG.md`
