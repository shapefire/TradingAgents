# LLM 客户端架构

> 多提供商抽象层、能力矩阵、结构化输出绑定、提供商特定处理

## 1. 客户端抽象层

### 1.1 基类

**文件**: `tradingagents/llm_clients/base_client.py`

`BaseLLMClient` 是抽象基类：

| 方法 | 说明 |
|------|------|
| `__init__(model, base_url, **kwargs)` | 构造函数 |
| `get_llm()` | 抽象方法，返回配置好的 LangChain Chat 实例 |
| `validate_model()` | 抽象方法，检查模型是否为该提供商已知模型 |
| `warn_if_unknown_model()` | 如果模型不在已知列表中，发出 `RuntimeWarning` |

**模块级函数**：
- `normalize_content()` — 将 LLM 响应内容从块列表（推理、文本等）标准化为纯字符串

### 1.2 工厂函数

**文件**: `tradingagents/llm_clients/factory.py`

```python
create_llm_client(provider, model, base_url, **kwargs) -> BaseLLMClient
```

使用懒加载导入，确保测试收集不需要 API key：

| 条件 | 返回客户端 |
|------|-----------|
| `provider == "anthropic"` | `AnthropicClient` |
| `provider == "google"` | `GoogleClient` |
| `provider == "azure"` | `AzureOpenAIClient` |
| `provider == "bedrock"` | `BedrockClient` |
| `is_openai_compatible(provider)` | `OpenAIClient` |
| 其他 | 抛出 `ValueError` |

## 2. 提供商实现

### 2.1 OpenAI 客户端（最复杂）

**文件**: `tradingagents/llm_clients/openai_client.py`

使用声明式 `ProviderSpec` 注册表（`OPENAI_COMPATIBLE_PROVIDERS`），包含 16+ 提供商：

每个 `ProviderSpec` 定义：
- `chat_class` — 使用的 Chat 类
- `base_url` — 默认基础 URL
- `base_url_env` — 覆盖基础 URL 的环境变量
- `key_optional` — 是否允许无 key
- `placeholder_key` — 无 key 时的占位符
- `require_base_url` — 是否必须提供 base_url
- `use_responses_api` — 是否使用 OpenAI Responses API

**类层次结构**：

```
ChatOpenAI (LangChain)
  └── NormalizedChatOpenAI — 内容标准化，能力感知的 with_structured_output
        ├── DeepSeekChatOpenAI — 捕获/恢复 reasoning_content
        └── MinimaxChatOpenAI — 注入 reasoning_split=True
```

**特殊处理**：

| 提供商 | 特殊处理 |
|--------|----------|
| `openai` | 原生端点使用 Responses API（`use_responses_api=True`） |
| `deepseek` | `DeepSeekChatOpenAI` 子类捕获/恢复 `reasoning_content` |
| `minimax`/`minimax-cn` | `MinimaxChatOpenAI` 子类注入 `reasoning_split=True` |
| `ollama` | key 可选，占位符 "ollama"，base_url 来自 `OLLAMA_BASE_URL` 环境变量 |
| `openai_compatible` | 必须提供 base_url，key 可选 |

**base_url 优先级**：显式客户端 base_url > 提供商环境变量覆盖 > 提供商默认值

**API Key 解析**：通过 `api_key_env.get_api_key_env()` 查找环境变量，必需时缺失则抛出异常，key 可选时发送占位符。

### 2.2 Anthropic 客户端

**文件**: `tradingagents/llm_clients/anthropic_client.py`

- `NormalizedChatAnthropic(ChatAnthropic)` 内容标准化
- 透传 kwargs：timeout, max_retries, api_key, max_tokens, temperature, effort
- `effort` 参数（扩展思考）受 `_supports_effort()` 限制：仅 Claude Opus 4.5+ 和 Sonnet 4.5+ 支持（Haiku 返回 400）

### 2.3 Google 客户端

**文件**: `tradingagents/llm_clients/google_client.py`

- `NormalizedChatGoogleGenerativeAI(ChatGoogleGenerativeAI)` 内容标准化
- `thinking_level` 配置映射到模型特定 API 参数：
  - Gemini 3 模型：直接传递 `thinking_level`（Gemini 3 Pro 将 "minimal" 重映射为 "low"）
  - Gemini 2.5：映射到 `thinking_budget`（0=禁用，-1=动态）

### 2.4 Azure 客户端

**文件**: `tradingagents/llm_clients/azure_client.py`

- `NormalizedAzureChatOpenAI(AzureChatOpenAI)` 内容标准化
- 读取 `AZURE_OPENAI_DEPLOYMENT_NAME` 环境变量
- `validate_model()` 始终返回 True（Azure 接受任何已部署的模型名称）

### 2.5 Bedrock 客户端

**文件**: `tradingagents/llm_clients/bedrock_client.py`

- 懒加载导入 `langchain-aws`（可选 `[bedrock]` extra）
- `NormalizedChatBedrockConverse(ChatBedrockConverse)` 内容标准化
- 使用 AWS 凭证链（环境变量、`~/.aws/credentials`、IAM 角色）
- 区域默认 `us-west-2`

## 3. 能力矩阵

**文件**: `tradingagents/llm_clients/capabilities.py`

`ModelCapabilities` 数据类声明每个模型的 API 约束：

| 字段 | 类型 | 说明 |
|------|------|------|
| `supports_tool_choice` | bool | 是否接受 `tool_choice` 参数 |
| `supports_json_mode` | bool | 是否支持 `response_format={"type":"json_object"}` |
| `supports_json_schema` | bool | 是否支持 `response_format={"type":"json_schema",...}` |
| `preferred_structured_method` | str | `"function_calling"` / `"json_mode"` / `"json_schema"` / `"none"` |
| `requires_reasoning_content_roundtrip` | bool | DeepSeek 思考模型需要 reasoning_content 回传 |
| `requires_reasoning_split` | bool | MiniMax M2.x 推理模型需要 `reasoning_split=True` |

**解析顺序**：精确 ID 匹配 → 正则模式匹配 → 默认值

**模型特定怪癖**：

| 模型 | 怪癖 |
|------|------|
| DeepSeek V4/reasoner | `supports_tool_choice=False`，`requires_reasoning_content_roundtrip=True` |
| MiniMax M2.x | `supports_tool_choice=False`，`requires_reasoning_split=True`，`supports_json_mode=False` |
| 默认 | 所有能力启用，`function_calling` 优先 |

`NormalizedChatOpenAI.with_structured_output()` 在运行时查询此表选择方法并抑制 `tool_choice`。

## 4. 模型验证

### 4.1 验证器

**文件**: `tradingagents/llm_clients/validators.py`

| 提供商 | 验证行为 |
|--------|----------|
| ollama, openrouter, openai_compatible, mistral, kimi, groq, nvidia, bedrock | 接受任何模型字符串（无警告） |
| openai, anthropic, google, xai, deepseek, qwen, glm, minimax | 对照共享模型目录验证 |

### 4.2 模型目录

**文件**: `tradingagents/llm_clients/model_catalog.py`

`MODEL_OPTIONS` 字典映射每个提供商到其 quick/deep 模式模型列表：

| 提供商 | 示例模型 |
|--------|----------|
| OpenAI | GPT-5.x |
| Anthropic | Claude Opus/Sonnet/Haiku |
| Google | Gemini 3.x/2.5 |
| xAI | Grok |
| DeepSeek | V4/reasoner |
| Qwen | 多个版本 |
| GLM | 多个版本 |
| MiniMax | M2.x |
| Ollama | 本地模型 |

## 5. API Key 管理

**文件**: `tradingagents/llm_clients/api_key_env.py`

提供商到环境变量名的单一真实映射，支持 20+ 提供商：

| 特殊条目 | 说明 |
|----------|------|
| `bedrock` → `None` | 使用 AWS 凭证链 |
| `ollama` → `None` | 本地部署，无需认证 |
| `openai_compatible` → `OPENAI_COMPATIBLE_API_KEY` | key 可选（无键本地服务器） |
| `qwen`/`qwen-cn` | 双区域，国际和中国账户不能共享凭证 |
| `glm`/`glm-cn` | 双区域 |
| `minimax`/`minimax-cn` | 双区域 |

## 6. 完整请求流程

### 6.1 LLM 客户端初始化

```
用户创建 TradingAgentsGraph(config=...)
  → DEFAULT_CONFIG 加载 + TRADINGAGENTS_* 环境变量覆盖
  → set_config(config) 推送到 dataflows 模块
  → _get_provider_kwargs() 提取提供商特定参数
  → create_llm_client(provider, model, base_url, **kwargs) × 2
  → 工厂懒加载对应提供商客户端
  → get_llm():
      → warn_if_unknown_model() 验证
      → 解析 base_url（显式 > 环境变量 > 提供商默认）
      → 解析 API key（环境变量 > 占位符）
      → 配置提供商特定功能
      → 返回 Normalized* LangChain Chat 实例
```

### 6.2 结构化输出绑定

```
Agent 创建时:
  bind_structured(llm, Schema, agent_name)
    → llm.with_structured_output(schema)
    → 如果不支持: 返回 None + 警告
    → 如果支持: 返回结构化 LLM 实例

Agent 执行时:
  invoke_structured_or_freetext(structured_llm, plain_llm, prompt, render, agent_name)
    → 优先尝试结构化 LLM
    → 如果失败: 回退到 plain_llm.invoke(prompt)
    → 渲染结果为 markdown
```

## 7. 提供商特定思考配置

| 配置键 | 默认值 | 适用提供商 | 说明 |
|--------|--------|-----------|------|
| `google_thinking_level` | None | Google | Gemini 3: "high"/"minimal" |
| `openai_reasoning_effort` | None | OpenAI | 推理模型: "medium"/"high"/"low" |
| `anthropic_effort` | None | Anthropic | Claude Opus/Sonnet: "high"/"medium"/"low" |
| `temperature` | None | 所有 | 通用温度参数 |

## 8. 客户端类一览

| 客户端类 | 文件 | 基类 | 特殊处理 |
|----------|------|------|----------|
| `NormalizedChatOpenAI` | openai_client.py | ChatOpenAI | 内容标准化，能力感知结构化输出 |
| `DeepSeekChatOpenAI` | openai_client.py | NormalizedChatOpenAI | reasoning_content 往返 |
| `MinimaxChatOpenAI` | openai_client.py | NormalizedChatOpenAI | reasoning_split 注入 |
| `NormalizedChatAnthropic` | anthropic_client.py | ChatAnthropic | effort 参数门控 |
| `NormalizedChatGoogleGenerativeAI` | google_client.py | ChatGoogleGenerativeAI | thinking_level 映射 |
| `NormalizedAzureChatOpenAI` | azure_client.py | AzureChatOpenAI | 部署名称配置 |
| `NormalizedChatBedrockConverse` | bedrock_client.py | ChatBedrockConverse | AWS 凭证链 |
