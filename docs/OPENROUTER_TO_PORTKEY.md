# OpenRouter → Portkey 替换梳理与可行性评估

## 一、程序架构与 OpenRouter 功能梳理

### 1.1 项目结构（与 LLM 相关）

```
polyclaw/
├── lib/
│   └── llm_client.py      # 唯一 LLM 封装，当前实现为 OpenRouter
├── scripts/
│   ├── polyclaw.py        # CLI 入口，hedge 命令转发到 hedge.py
│   └── hedge.py           # hedge scan / hedge analyze，调用 LLMClient
├── README.md
└── SKILL.md
```

### 1.2 OpenRouter 在项目中的职责

| 位置 | 作用 |
|------|------|
| **lib/llm_client.py** | 封装 OpenRouter：配置 base_url、API key、默认模型；提供 `LLMClient.complete(messages, temperature, max_tokens)`，内部 POST `/chat/completions`，返回 `choices[0].message.content`；429/网络错误重试；单例 `get_llm_client()` / `close_llm_client()` |
| **scripts/hedge.py** | 唯一调用方：`LLMClient(model=args.model)`，在 `extract_implications_for_market()` 中 `await llm.complete([{"role":"user","content":prompt}], temperature=0.1)`，只依赖「返回一段字符串」 |
| **scripts/polyclaw.py** | help 中提示 `OPENROUTER_API_KEY`（hedge 所需） |
| **README.md / SKILL.md** | 环境变量说明、故障排查提到 OpenRouter |

### 1.3 LLM 调用链（核心逻辑）

1. 用户执行：`polyclaw hedge scan` 或 `polyclaw hedge analyze <id1> <id2>`  
2. `hedge.py` 拉取市场列表，对每个目标市场调用 `extract_implications_for_market(target, others, llm)`  
3. 该函数用 `IMPLICATION_PROMPT` 拼好 prompt，调用 `await llm.complete([{"role":"user","content":prompt}], temperature=0.1)`  
4. 将返回的文本用 `extract_json_from_response()` 解析为 JSON，再 `derive_covers_from_implications()` 转为 cover 关系并算 coverage  
5. 输出 hedge 组合表或 JSON  

**结论**：与「外部世界」的耦合只有 **lib/llm_client.py** 的 HTTP 调用（URL、认证、请求/响应格式）。hedge 业务不关心具体是 OpenRouter 还是 Portkey。

---

## 二、Portkey 文档要点（与替换相关）

### 2.1 通过 Header 指定配置即可调用

Portkey 通过 API 调用时，**只需在请求 Header 中指定配置**，无需在代码里维护具体 provider 或后端 API Key。

- **必填 Header**  
  - `x-portkey-api-key`: Portkey 账号 API Key（从 Portkey 控制台获取）
- **指定路由与认证的 Header（二选一或与 Config 配合）**  
  - `x-portkey-config`: **Config ID**（在 Portkey 后台创建的配置 ID，如 `pc-xxxxx-edx21x`）或 Config 的 JSON 对象。  
  - Config 在 Portkey 后台维护，其中包含：用哪个 **provider**（OpenRouter、OpenAI、Anthropic 等）、如何认证（Virtual Key / api_key 等）、以及可选的重试、缓存、fallback 等网关行为。  

因此：**我们只需要多传一个 header 参数 `x-portkey-config`（传 Config ID），具体用哪个 provider、密钥如何保管，全部在 Portkey 后台的该 Config 里维护，代码里不保留 OpenRouter 或任何具体 provider。**

参考：  
- [Headers - Portkey Docs](https://docs.portkey.ai/docs/api-reference/inference-api/headers)（Provider Authentication 方式 3：Config）  
- [Configs - Portkey Docs](https://docs.portkey.ai/docs/product/ai-gateway/configs)（通过 `x-portkey-config` 传 Config ID）  
- [101 on Portkey's Gateway Configs](https://docs.portkey.ai/docs/guides/getting-started/101-on-portkey-s-gateway-configs)（UI 创建 Config → 用 Config ID 通过 header 引用）

### 2.2 Chat Completions API

- **端点**: `POST https://api.portkey.ai/v1/chat/completions`  
- **请求体**: 与 OpenAI/OpenRouter 兼容（`model`, `messages`, `temperature`, `max_tokens` 等）。`model` 由请求体传入，需与 Config 里配置的 provider 对应（例如 Config 指向 OpenRouter 时用 OpenRouter 的 model id）。  
- **响应**: 与 OpenAI 一致，取 `choices[0].message.content` 即可。  

文档：[Portkey Chat API](https://docs.portkey.ai/docs/api-reference/inference-api/chat)

### 2.3 小结：代码侧只需两个 Header

| Header | 来源 | 说明 |
|--------|------|------|
| `x-portkey-api-key` | 环境变量 `PORTKEY_API_KEY` | Portkey 账号认证 |
| `x-portkey-config` | 环境变量 `PORTKEY_CONFIG_ID` | 在 Portkey 后台创建的 Config ID；provider 与密钥均在后台该 Config 中维护 |

不需要在代码或环境变量中保留 OpenRouter API Key 或 Virtual Key；不需要在代码里指定 provider 名称。

### 2.4 与官方 Configs 文档的对照确认

依据 [Configs - Portkey Docs](https://portkey.ai/docs/product/ai-gateway/configs)：

- **REST 使用方式**：文档写明 “Configs are supported … **Via the REST API through the `x-portkey-config` header**”，与我们的对接方式一致。
- **Config 的用途**：可在 UI 创建 Config 并得到 **config id**（如 `pc-***`），请求时通过 header 传入该 id 即可应用该配置。
- **可选：Default Config**：若在 [API Keys](https://app.portkey.ai/api-keys) 里为某个 Portkey API Key 设置了 **Default Config**，则使用该 key 的请求即使不传 `x-portkey-config` 也会自动套用该 Config。我们采用显式传 `PORTKEY_CONFIG_ID`，便于切换不同 Config。

依据 [Headers - Portkey Docs](https://docs.portkey.ai/docs/api-reference/inference-api/headers)：

- Provider Authentication 有 **4 种方式**，其中 **方式 3 为 Config**：`x-portkey-config` 接受 **config ID 或 JSON 对象**，且可包含 “gateway configuration settings, **and provider details**”。即仅通过 Config（ID）即可同时提供「用哪个 provider」与「如何认证」，无需在请求里再传 `x-portkey-provider` 或 `Authorization: Bearer <后端 key>`。
- 因此 **仅传 `x-portkey-api-key` + `x-portkey-config`（Config ID）** 在官方定义下是完整、正确的对接方式。

**关于 Configs 页 cURL 示例**：该示例同时带有 `Authorization: Bearer $OPENAI_API_KEY`、`x-portkey-provider: openai` 和 `x-portkey-config`，对应的是「用 OpenAI SDK 兼容方式、应用自带 OpenAI key 并显式指定 provider」的用法；Config 在此处主要提供网关能力（如重试）。我们采用的是「全部由 Config 定义 provider 与认证」的用法，与 Portkey 原生 SDK 的 `apiKey + config: "pc-***"` 等价，不需要在请求中传后端 key 或 provider。

---

## 三、替换可行性评估

### 3.1 接口兼容性

| 项目 | OpenRouter（当前） | Portkey（仅 Header 指定 Config） | 结论 |
|------|--------------------|-----------------------------------|------|
| 请求 URL | `https://openrouter.ai/api/v1/chat/completions` | `https://api.portkey.ai/v1/chat/completions` | 仅改 base_url |
| 认证与路由 | `Authorization: Bearer OPENROUTER_API_KEY` | `x-portkey-api-key` + `x-portkey-config`（Config ID） | 改两个 header；provider 在后台 Config 维护 |
| 请求体 | `model`, `messages`, `temperature`, `max_tokens` | 相同 | 无需改 |
| 响应解析 | `data["choices"][0]["message"]["content"]` | 相同 | 无需改 |

因此：**在不改 `LLMClient.complete()` 的入参和返回值的前提下，只需在 llm_client 内更换 base_url 并增加两个 header（`x-portkey-api-key`、`x-portkey-config`）即可。**

### 3.2 方案说明（仅 Header + 后台 Config）

- **具体用哪个 provider（OpenRouter、OpenAI 等）以及对应 API Key / Virtual Key，全部在 Portkey 后台的 Config 里配置。**  
- 代码只负责：Portkey 的 base_url、`x-portkey-api-key`、`x-portkey-config`（Config ID）。  
- 请求体中的 `model` 仍由调用方传入（如 hedge 的 `--model`），需与你在 Portkey 该 Config 里配置的 provider 的 model 命名一致（例如 Config 配的是 OpenRouter 则用 OpenRouter 的 model id）。

### 3.3 需要改动的范围

| 文件 | 改动类型 | 说明 |
|------|----------|------|
| **lib/llm_client.py** | 逻辑 + 配置 | base_url → Portkey；请求头改为 `x-portkey-api-key`（来自 `PORTKEY_API_KEY`）与 `x-portkey-config`（来自 `PORTKEY_CONFIG_ID`）；不再使用 `OPENROUTER_API_KEY` |
| **scripts/polyclaw.py** | 文档/帮助 | help 中把 OPENROUTER_API_KEY 改为 PORTKEY_API_KEY、PORTKEY_CONFIG_ID |
| **README.md** | 文档 | 环境变量、故障排查改为 Portkey（PORTKEY_API_KEY、PORTKEY_CONFIG_ID；说明 Config 在 Portkey 后台维护） |
| **SKILL.md** | 文档 | 同上，环境变量与依赖说明 |

**hedge.py 无需改**：仍只依赖 `LLMClient` 与 `complete()` 的接口。

### 3.4 依赖与实现方式

- 当前用 **httpx** 直接发 POST，无 OpenRouter 专用 SDK。  
- 替换为 Portkey 可继续用 **httpx**，仅改 base_url 与 headers（多传 `x-portkey-api-key`、`x-portkey-config`），不必引入 Portkey SDK。

### 3.5 结论与建议

- **替换可行**：Portkey 通过 API 调用时只需在 Header 中指定 Config（Config ID），具体 provider 在 Portkey 后台自行维护，代码只需多传一个 header 参数 `x-portkey-config`。  
- **实施步骤建议**：  
  1. 在 Portkey 后台创建 Config（选择 provider、配置认证等），保存后得到 **Config ID**。  
  2. 修改 `lib/llm_client.py`：base_url 改为 Portkey；headers 设为 `x-portkey-api-key`、`x-portkey-config`（环境变量 `PORTKEY_API_KEY`、`PORTKEY_CONFIG_ID`）。  
  3. 更新 README、SKILL、polyclaw help 中的环境变量说明（PORTKEY_API_KEY、PORTKEY_CONFIG_ID；不再使用 OPENROUTER_API_KEY）。  
  4. 用 `hedge scan --limit 2` 或 `hedge analyze <id1> <id2>` 做一次回归验证。
