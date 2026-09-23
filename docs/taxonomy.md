# LLM API 协议分类

> 这份分类是从各层文档的调研结果里归纳出来的，不是事先定好的框架；新证据推翻它时会重写（Gemini Interactions 的出现已经让第一层重写过一次）。事实和来源在 [导读](llm-api-protocols.md)、[字段对照](field-atlas.md) 和 `details/` 下各页，这里只讲“怎么分、为什么这么分”。

## 1. 为什么这么分

调研发现，真正会让代码出错的差异集中在三处：

1. **请求长什么样**：原生格式只有五种，第三方基本都是照着做的。
2. **厂商从哪个入口提供**：同一家常常同时开三种入口，但每个入口的模型范围、是否记历史、字段支持都不一样。
3. **同一格式下的差别**：字段名一样，取值、默认值、是否生效不一样。

所以分三层：格式 → 入口 → 同一格式下的差别。

## 2. 第一层：五种原生格式

按两个实际影响代码结构的问题来排：

| | 工具参数是 JSON **字符串** | 工具参数是 JSON **对象** |
|---|---|---|
| **轮次型**：历史是一轮轮消息，每次全量发，服务端不记 | OpenAI Chat Completions（`messages[]`） | Anthropic Messages（content blocks）<br>Gemini `generateContent`（`parts[]`，官方现称 legacy） |
| **条目型**：历史是一串带类型的条目，服务端可以记 | OpenAI Responses（items，`previous_response_id`） | Gemini Interactions（steps，`previous_interaction_id`，2026-06 GA） |

- 条目型不等于一定有状态：两者都能关掉存储（`store=false`），这时要像轮次型一样自己带全部历史，包括带签名的推理条目。
- `generateContent` 另有托管版本 Vertex AI：请求体相同，地址和鉴权不同。
- 未纳入网格：OpenAI Realtime 和 Gemini Live——它们是 WebSocket / WebRTC 会话，收发事件，不是一问一答的请求格式。
- DashScope 原生请求体：外壳和 Chat Completions 不同，但按两个轴看归 Chat Completions 那一格，见 [第三方厂商](details/vendors.md)。

## 3. 第二层：协议入口

| 入口类型 | 是什么 | 例子 |
|---|---|---|
| 原生入口 | 格式的发明者自己的 API | OpenAI、Anthropic、Google；智谱原生 API 本身就是 Chat Completions 格式；阿里 DashScope 另有一套原生格式 |
| 兼容入口 | 第三方照 Chat Completions / Responses / Anthropic Messages 做的版本 | 见下表 |
| 套餐专用入口 | 编程套餐单独的地址和 key，额度走套餐；用错地址就用不上套餐额度 | 智谱 Coding Plan、Kimi Code、豆包 Agent Plan / Coding Plan |
| 大厂自带的 Chat 兼容层 | 大厂把自家模型再用 Chat Completions 的格式包一层 | Anthropic 的 OpenAI SDK 兼容、Gemini `/v1beta/openai/` |
| 聚合 / 自托管 | 转发或自己部署 | OpenRouter、vLLM、Ollama |
| 云托管 | 云厂商代管，地址、鉴权、版本号、流式格式都可能不同 | Azure OpenAI；Gemini 和 Claude 上 Vertex AI；Claude 上 Bedrock 和 Azure Foundry（见 [Claude 云托管](details/hosted-claude.md)） |

主流第三方厂商开了哪些入口（★ = 官方推荐）：

| 厂商 | Chat Completions | Responses | Anthropic Messages | 最需要注意的差别 |
|---|---|---|---|---|
| DeepSeek | ✓ | ✓ 不存对话 | ✓ | 带 `tools` 时必须回传 `reasoning_content`；Chat 只有 `json_object`（Responses 有 `json_schema`）；按并发限流 |
| 智谱 GLM | ✓（原生） | ✓，`store` 默认 false，可开启 | ✓ 字段支持没文档 | `tool_choice` 只有 `auto`；错误码是数字字符串 |
| Kimi | ✓ | ✓ 仅 `kimi-k3`，不存对话 | ✓（参考页仅 `kimi-k3`，接入指南与之冲突） | 采样参数固定，乱传报错；`strict` 默认开 |
| xAI Grok | ✓（已标 Deprecated） | ★ 存对话 | 已废弃 | grok-4.5 起推理关不掉；`strict` 始终开；批处理只有部分文本模型 8 折 |
| Qwen | ✓ | ✓ 存对话 | ✓ | `tool_choice: required` 三处官方页说法冲突；`parallel_tool_calls` 默认 false；Anthropic 入口签名为空 |
| MiniMax | ✓ | ✓ 请求里没有 `previous_response_id` | ★ | 思考默认值随入口不同（见 [字段对照 · 推理](field-atlas.md)）；`<think>` 不能剥 |
| 豆包 | ✓ | ✓ 存对话 | ✓ | 新模型不回传 `encrypted_content` 会降低推理质量（不报错），密文被篡改则无法还原 |
| Azure（Foundry） | ✓ | ✓ | ✓ 仅 Claude 模型，另一地址 | `model` 填部署名；`api-key` 头；`retry-after-ms` |

## 4. 第三层：同一格式下的差别

按调研中出现的频率和后果排序：

| # | 差别 | 典型表现 | 后果 |
|---|---|---|---|
| 1 | 推理内容回传 | 明文 `reasoning_content`、密文 `encrypted_content`、thinking 块 + `signature`、`thoughtSignature`、`<think>` 标签，各家规则不同 | 漏传直接 400，或效果变差 |
| 2 | 思考开关与力度 | `thinking.type` 和 `reasoning_effort` 并存；同名 `reasoning_effort` 的取值各不相同 | 传了不生效，或被映射成别的档 |
| 3 | 工具强制与 `strict` | `tool_choice` 只支持一部分；`strict` 默认开、默认关、始终开、只在 beta | 强制调用失败或 400 |
| 4 | 结构化输出 | 只有 `json_object`，没有 `json_schema` | 拿不到约束过的 JSON |
| 5 | 流式：用量位置 / 结束标记 / 保活空行 | 用量在哪一块、有没有 `[DONE]`、会不会发保活空行 | 解析器丢用量或断流 |
| 6 | 采样参数处理 | 忽略 / 报错 / 固定值 | 行为和预期不一样 |
| 7 | 第三方 Responses 能不能续历史 | 形状像 Responses，但不支持 `previous_response_id`（有的忽略，有的 400） | 依赖服务端历史的逻辑失效 |
| 8 | 鉴权、错误、限流 | 头名不同；402 / 422 / 529 / 数字业务码；按并发限流而不是 RPM | 重试和告警逻辑判断错 |
