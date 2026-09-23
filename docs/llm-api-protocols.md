# LLM API 协议：学习用地图手册（导读）

> 这是第 0 层：只画全局地图——有哪几种标准格式、第三方从哪里分叉、最容易写错的名字。读完再按需查 [字段对照](field-atlas.md)（第 1 层），细节去 `details/` 下的各页（第 2 层）。
> 字段名、JSON 键、路径保持官方英文。来源链接的文字写明厂商和文档，完整列表见 [来源](details/sources.md)，全部于 2026-09-23 抓取。官方说法打架或查不到原文的条目，集中在 [冲突与未核实](details/conflicts.md)。

## 开篇先搞清的 4 件事

- **兼容≠行为等同**：第三方写“兼容 OpenAI / Anthropic”，只说明请求长得像；同一个字段可能被忽略、取值不同或直接报错。逐项差异见 [第 3 节](#3-第三方从哪里分叉)。
- **工具参数是字符串还是对象**：Chat Completions 和 Responses 返回的工具参数是 JSON 字符串，要自己解析；Anthropic Messages、`generateContent`、Interactions 返回的是 JSON 对象。见 [第 1 节](#1-全局地图五种原生格式)。
- **服务端是否记对话**：Chat Completions、Anthropic Messages、`generateContent` 每次都要自己发全部历史；Responses 和 Interactions 可以用 `previous_response_id` / `previous_interaction_id` 接着上一次。第三方的 Responses 入口不一定支持续历史，见 [第 3 节](#3-第三方从哪里分叉)。
- **推理/思考内容须按规则回传**：多轮对话里（尤其带工具时），上一轮返回的思考内容或签名（`reasoning_content`、`encrypted_content`、thinking 块、`thoughtSignature`）很多家要求原样带回。漏传轻则效果变差，重则直接 400，各家规则见 [第 4 节](#4-容易记混的名字口诀) 和 [字段对照 · 推理](field-atlas.md#5-推理思考怎么开要不要回传-重点)。

## 先认识这些词

| 词 | 一句话 |
|---|---|
| 原生格式 | 模型厂商自己定义的请求和响应格式，本手册只讲五种：Chat Completions、Responses、Anthropic Messages、`generateContent`、Interactions。 |
| 兼容入口 | 第三方照某种原生格式做的接口地址（如 DeepSeek 的 `https://api.deepseek.com/anthropic`），请求长得像，行为不一定一样。 |
| [轮次型](#1-全局地图五种原生格式) | 历史是一轮轮消息（`messages` / `contents`），每次请求都自己发全量历史。 |
| [条目型](#1-全局地图五种原生格式) | 历史是一串带类型的条目（items / steps），服务端可以替你保存，用上一次的 ID 接着聊。 |
| 工具调用（function calling） | 模型不直接回答，而是返回要调用的函数名和参数，由你执行后把结果发回去。 |
| `tool_choice` | 控制模型可以不调工具、必须调工具或必须调指定工具的请求字段，各家支持的取值不同。 |
| 结构化输出（`json_schema`） | 让模型按你给的 JSON Schema 输出；只有 `json_object` 的入口只保证输出是 JSON，不按 schema 检查字段。 |
| 推理 / 思考（reasoning / thinking） | 模型在最终回答前生成的中间推理，有的以明文返回，有的只给摘要、密文或签名。 |
| 签名 / 密文 | `signature`、`thoughtSignature`、`encrypted_content` 这类字段是服务端给思考内容附带的校验值或加密内容，要求回传时必须原样带回。 |
| 停止原因 | 响应里说明“为什么停了”的字段，如 `finish_reason`、`stop_reason`、`finishReason`，正常结束、到长度上限、要调工具对应不同的值。 |
| SSE | 流式输出用的 HTTP 传输方式，服务端一条条推送 `data:` 行（有的还带 `event:` 行）。 |
| 提示缓存 | 服务端缓存重复的请求前缀，命中的部分在用量里单独计数；有的自动，有的要自己标断点。 |

## 1. 全局地图：五种原生格式

常见的原生请求格式只有五种：OpenAI 的 Chat Completions 和 Responses、Anthropic Messages、Google 的 `generateContent` 和 Interactions（[OpenAI Chat Completions 参考][oa-chat]；[OpenAI Responses 参考][oa-resp]；[Anthropic Messages 参考][an-msg]；[Gemini generateContent 参考][g-gen]；[Gemini Interactions 参考][g-int-api]）。Google 在 2026 年 6 月把 Interactions 定为正式发布（GA）并推荐新项目使用，`generateContent` 改称 legacy，但仍完全支持（[Gemini Interactions 概览][g-int-ov]）。

按两个问题就能把它们摆开：

| | 工具参数是 JSON 字符串 | 工具参数是 JSON 对象 |
|---|---|---|
| **轮次型**：历史每次自己全量发 | Chat Completions | Anthropic Messages；Gemini `generateContent` |
| **条目型**：历史是带类型的条目，服务端可以帮你记 | OpenAI Responses | Gemini Interactions |

- 轮次型的历史是一轮轮 `messages` / `contents`；条目型的历史是一串 items / steps，可以用 `previous_response_id` / `previous_interaction_id` 接着上一次聊（[OpenAI 迁移到 Responses 指南][oa-migrate]；[Gemini Interactions 概览][g-int-ov]）。
- OpenAI 和 Google 都已把条目型设为新项目推荐，Anthropic 目前只有轮次型（[OpenAI 迁移到 Responses 指南][oa-migrate]；[Gemini Interactions 概览][g-int-ov]）。
- 工具结果放哪：Chat 是单独一条 `role: "tool"` 消息；Anthropic 和 `generateContent` 放进下一轮 user 消息；Responses 和 Interactions 是单独一个 item / step（[OpenAI 函数调用指南][oa-fc]；[Anthropic 定义工具文档][an-tools]；[Gemini generateContent 函数调用指南][g-fc]；[Gemini Interactions 参考][g-int-api]）。
- Chat 的 `store` 字段只是把记录存下来，不能代替重发历史（[OpenAI Chat Completions 参考][oa-chat]）。

**本文不讲**：OpenAI Realtime 和 Gemini Live（会话式协议）、embeddings、图片 / 视频生成、已下线的 OpenAI Assistants API、Bedrock 的 `Converse` 格式。

## 2. 五种格式速览

| | Chat Completions | Responses | Anthropic Messages | `generateContent` | Interactions |
|---|---|---|---|---|---|
| 路径 | `POST /v1/chat/completions` | `POST /v1/responses` | `POST /v1/messages` | `POST /v1beta/models/{model}:generateContent` | `POST /v1beta/interactions` |
| 服务端记历史 | 不记 | 可以（`previous_response_id`） | 不记 | 不记 | 默认记（`previous_interaction_id`） |
| 历史容器 | `messages[]` | `input` items | `messages[]` + 内容块 | `contents[]` + `parts[]` | `input` steps |
| 工具参数 | 字符串 | 字符串 | 对象 | 对象 | 对象 |

来源：[OpenAI Chat Completions 参考][oa-chat]；[OpenAI Responses 参考][oa-resp]；[Anthropic Messages 参考][an-msg]；[Gemini generateContent 参考][g-gen]；[Gemini Interactions 参考][g-int-api]；[Gemini Interactions 概览][g-int-ov]。更多行（鉴权、版本、角色、停止原因、用量）见 [字段对照 · 请求骨架](field-atlas.md#1-请求骨架五种原生格式)。

## 3. 第三方从哪里分叉

- **第三方基本不发明新格式，而是同时开好几个兼容入口**：DeepSeek、Kimi、Qwen、智谱、豆包、MiniMax 都有 Chat Completions、Responses、Anthropic Messages 三种入口（Kimi 参考页写后两种只支持 `kimi-k3`，但它的 Claude Code 接入指南另用 `kimi-k2.7-code`，见冲突页）（[DeepSeek API 文档首页][ds-home]；[Kimi API 总览][kimi-ov]；[阿里云百炼文本生成指南][ali-tg]；[智谱对话补全参考][zp-chat]；[火山方舟 Chat 接口参考][ark-chat]；[MiniMax 文本生成指南][mm-tg]）。MiniMax 官方推荐走 Anthropic 入口，xAI 推荐 Responses、Anthropic 入口已废弃（[MiniMax 文本生成指南][mm-tg]；[xAI 接口对比页][xai-cmp]；[xAI 旧版接口参考][xai-legacy]）。
- **同一家换个入口，行为可能不同**：比如同一个 MiniMax 模型，思考默认开还是关取决于走哪个入口，见 [字段对照 · 推理](field-atlas.md#5-推理思考怎么开要不要回传-重点)（[MiniMax OpenAI 兼容接口][mm-oai]；[MiniMax Anthropic 兼容接口][mm-anth]；[MiniMax Responses 接口参考][mm-resp]）。
- **叫 “Responses” 不一定会记历史**：DeepSeek、Kimi 等第三方的 Responses 不支持 `previous_response_id`，有的静默忽略，有的直接 400（[DeepSeek Responses 指南][ds-resp]；[Kimi Responses 接口参考][kimi-resp]；[OpenRouter Responses 错误处理][or-resp]）。
- **阿里 DashScope 另有一套原生格式**：外壳不同，但也是每次重发历史、工具参数为字符串，归在 Chat Completions 那一格（[阿里云百炼 DashScope 原生接口][ali-native]）。
- **“兼容”只保证请求长得像**。最容易出问题的六处：思考内容要不要回传、`tool_choice` 能不能强制、有没有 `json_schema`、流式里用量在哪一块、Responses 能不能续历史、采样参数传了是被忽略还是报错。逐家对照见 [字段对照 · 第三方同义字段](field-atlas.md#7-第三方同义字段chat-兼容入口) 和 [第三方厂商](details/vendors.md)。

## 4. 容易记混的名字（口诀）

- **思考开关**：OpenAI 用 `reasoning_effort`（Responses 里是 `reasoning.effort`）；Anthropic 用 `thinking` 加 `output_config.effort`；Gemini 用 `thinkingLevel` / `thinkingBudget`；DeepSeek、智谱、Kimi K2、豆包用 `thinking.type`；Qwen 用 `enable_thinking`（[OpenAI Chat Completions 参考][oa-chat]；[Anthropic Messages 参考][an-msg]；[Gemini generateContent 思考文档][g-think]；[DeepSeek 思考模式指南][ds-think]；[智谱思考模式指南][zp-think]；[Kimi Chat 接口参考][kimi-chat]；[火山方舟深度思考文档][ark-think]；[阿里云百炼深度思考指南][ali-think]）。同名的 `reasoning_effort` 在各家取值也不一样。
- **思考要不要带回下一轮**：DeepSeek、Kimi 看 `reasoning_content`；OpenAI、xAI、豆包看 `encrypted_content`；Anthropic 看 thinking 块和 `signature`；Gemini 看 `thoughtSignature`。该带回却没带，轻则效果变差，重则直接 400（[DeepSeek 思考模式指南][ds-think]；[Kimi 思考模型指南][kimi-think]；[OpenAI 推理指南][oa-reasoning]；[xAI 推理指南][xai-reason]；[火山方舟 Chat 接口参考][ark-chat]；[Anthropic Messages 参考][an-msg]；[Gemini 思考签名文档][g-sig]）。
- **工具结果**：Chat `role: "tool"` + `tool_call_id`；Responses `function_call_output` + `call_id`；Anthropic `tool_result` + `tool_use_id`；`generateContent` `functionResponse`；Interactions `function_result` + `call_id`（[OpenAI 函数调用指南][oa-fc]；[Anthropic 定义工具文档][an-tools]；[Gemini generateContent 函数调用指南][g-fc]；[Gemini Interactions 参考][g-int-api]）。
- **结构化输出**：Chat `response_format`；Responses `text.format`；Anthropic `output_config.format`；`generateContent` `responseJsonSchema`；Interactions `response_format`（[OpenAI Chat Completions 参考][oa-chat]；[OpenAI 迁移到 Responses 指南][oa-migrate]；[Anthropic 结构化输出文档][an-so]；[Gemini generateContent 参考][g-gen]；[Gemini Interactions 参考][g-int-api]）。

## 5. 调用时一定会遇到的几件事

- **停止**：各家都会告诉你“为什么停了”，只是字段名不同（`finish_reason`、`stop_reason`、`finishReason`、`status`）；正常说完、到长度上限、要调工具、被拦下是不同结局，不要都当失败。流式时看最后一块或结束事件（[OpenAI Chat Completions 参考][oa-chat]；[OpenAI Responses 参考][oa-resp]；[Anthropic Messages 参考][an-msg]；[Gemini generateContent 参考][g-gen]；[Gemini Interactions 参考][g-int-api]）。
- **流式**：各家基本都用 SSE，差别在结束标记（`[DONE]`、`message_stop`、`response.completed`；`generateContent` 没有结束标记，看 `finishReason`）、用量出现在哪一块，以及会不会插入空行或注释行（[OpenAI Chat Completions 参考][oa-chat]；[Anthropic 流式文档][an-stream]；[OpenAI Responses 流式事件参考][oa-resp-events]；[Gemini generateContent 参考][g-gen]；[DeepSeek 限速说明][ds-rate]）。
- **报错**：流开始前出错是普通 HTTP 状态码加 JSON；流开始后出错是流里的一个事件（[Anthropic 错误文档][an-errors]）。同一个 429 可能是限流也可能是额度用完，先看错误体（[OpenAI 错误码指南][oa-errors]；[Kimi 常见问题排查][kimi-trouble]）。
- **断开和重试**：流断了基本不能接着收，重试就是重新生成；已调过工具的要当心重复执行。OpenAI 的 SDK 不会在流开始后自动重试，因为重放可能生成重复内容；你自己重发，模型就会重新生成一遍（[OpenAI 后台模式指南][oa-bg]；[OpenAI Python SDK README][oa-py-readme]）。

这几件事的细节和断连检查清单见 [流式、停止与断开](details/streaming.md)，各家错误码见 [错误、重试与限流](details/ops.md)。

## 6. 你手上有哪些能力（细节后面查）

- **图片 / 文件 / 音频输入**：五种格式写法各不同，大小上限也不同；Chat 的音频只收 base64 的 `wav` / `mp3`；Responses 文档目前只写文本和图片输入，音频对话要用 Chat Completions（[OpenAI Chat 音频指南][oa-audio]）。→ [字段对照 · 图片文件音频](field-atlas.md#6-图片文件音频-重点)
- **内置工具**（联网、代码执行等）：Responses、Anthropic、`generateContent`、Interactions 都有，名单各不相同；Chat 只有 `web_search_options`（[OpenAI Responses 参考][oa-resp]；[Anthropic 服务端工具文档][an-server-tools]；[Gemini generateContent 参考][g-gen]；[Gemini Interactions 参考][g-int-api]；[OpenAI Chat Completions 参考][oa-chat]）。→ [原生格式细节](details/native-formats.md)
- **流式**：结束标记、用量位置、出错方式各家不同；比如 Chat 以 `data: [DONE]` 结束，Anthropic 以 `message_stop` 结束（[OpenAI Chat Completions 参考][oa-chat]；[Anthropic 流式文档][an-stream]）。→ [字段对照 · 流式](field-atlas.md#3-流式)、[流式、停止与断开](details/streaming.md)
- **缓存**：OpenAI 默认自动；Anthropic 可自动也可手动打断点；Gemini 可以先建缓存资源（[OpenAI 提示缓存指南][oa-cache]；[Anthropic 提示缓存文档][an-cache]；[Gemini generateContent 缓存文档][g-cache]）。→ [缓存](details/caching.md)
- **批处理**：OpenAI、Anthropic、`generateContent` 都有，价格约五折；Interactions 暂时没有（[OpenAI 批处理指南][oa-batch]；[Anthropic 批处理文档][an-batch]；[Gemini 批处理文档][g-batch]；[Gemini 迁移到 Interactions 指南][g-int-mig]）。→ [错误、重试与限流](details/ops.md)
- **云上的 Claude**：Bedrock、Vertex、Azure Foundry 的地址和功能都与官方不同，Bedrock 和 Vertex 的版本号写法、Bedrock 的流式格式也不同（[AWS Bedrock InvokeModel 参考][bedrock-invoke]；[Anthropic Vertex AI 说明][an-vertex]；[Azure Foundry Claude 模型说明][az-claude]）。→ [Claude 云托管](details/hosted-claude.md)

## 7. 怎么选

- 要服务端记历史 → OpenAI Responses 或 Gemini Interactions。
- 要现成的联网、代码执行等内置工具 → 同样是这两个，名单各不相同：Interactions 还有 `computer_use`、`mcp_server`、`google_maps` 等（[OpenAI Responses 参考][oa-resp]；[Gemini Interactions 参考][g-int-api]）。
- 要自己指定缓存位置、要带回思考签名 → Anthropic Messages。
- 要最多厂商都支持的入口 → Chat Completions；但 OpenAI 从 GPT-5.4 起，`reasoning_effort` 不是 `none` 时 Chat 不能调工具（[OpenAI 迁移到 Responses 指南][oa-migrate]）。
- Gemini 新项目用 Interactions；批处理、显式缓存暂时还得用 `generateContent`；自定义安全设置官方两处说法不一，见冲突页（[Gemini Interactions 概览][g-int-ov]；[Gemini 迁移到 Interactions 指南][g-int-mig]）。

## 8. 接一家新厂商前的检查清单

1. 把第 3 节那六处逐一测一遍。
2. 用 OpenAI SDK 接第三方时，思考字段要自己放回下一轮：Python SDK 会把未文档化字段留在 `model_extra` 里，但文档没说会自动带回（[OpenAI Python SDK 参考][oa-py]）。
3. 推理模型把超时调大：xAI 示例用 3600 秒，OpenAI Python SDK 默认 10 分钟（[xAI 异步请求指南][xai-async]；[OpenAI Python SDK 参考][oa-py]）。
4. token 数不能跨家比：分词器不同，Gemini 的 `totalTokenCount` 还包含思考 token（[Anthropic token 计数文档][an-count]；[Gemini generateContent 参考][g-gen]）。

## 往下读

| 层 | 文件 | 什么时候看 |
|---|---|---|
| 1 | [字段对照](field-atlas.md) | 写代码时查：同一个意思在各家叫什么、放哪、默认值 |
| 2 | [工具调用示例](details/tool-calls.md) | 要复制一段能用的请求体 |
| 2 | [原生格式细节](details/native-formats.md) | 各格式的坑；Gemini 两种格式怎么选 |
| 2 | [第三方厂商](details/vendors.md) | DeepSeek、智谱、Kimi、xAI、Qwen、MiniMax、豆包、Azure 的入口和差异 |
| 2 | [流式、停止与断开](details/streaming.md) | 停止原因、流内报错、断开和重试；第三方的流式差异 |
| 2 | [缓存](details/caching.md) | 要省钱、要命中缓存 |
| 2 | [Claude 云托管](details/hosted-claude.md) | 在 Bedrock / Vertex / Foundry 上用 Claude |
| 2 | [错误、重试与限流](details/ops.md) | 写重试、告警、限流逻辑 |
| 2 | [冲突与未核实](details/conflicts.md) | 查哪些说法官方自己都不一致 |
| — | [分类说明](taxonomy.md) | 想知道为什么这么分 |
| — | [来源](details/sources.md) | 全部来源链接 |

---

[ali-native]: https://www.alibabacloud.com/help/en/model-studio/qwen-api-via-dashscope
[ali-tg]: https://www.alibabacloud.com/help/en/model-studio/text-generation
[ali-think]: https://www.alibabacloud.com/help/en/model-studio/deep-thinking
[an-batch]: https://platform.claude.com/docs/en/build-with-claude/batch-processing
[an-cache]: https://platform.claude.com/docs/en/build-with-claude/prompt-caching
[an-count]: https://platform.claude.com/docs/en/build-with-claude/token-counting
[an-errors]: https://platform.claude.com/docs/en/api/errors
[an-msg]: https://platform.claude.com/docs/en/api/messages
[an-server-tools]: https://platform.claude.com/docs/en/agents-and-tools/tool-use/server-tools
[an-so]: https://platform.claude.com/docs/en/build-with-claude/structured-outputs
[an-stream]: https://platform.claude.com/docs/en/build-with-claude/streaming
[an-tools]: https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools
[an-vertex]: https://platform.claude.com/docs/en/build-with-claude/claude-on-vertex-ai
[ark-chat]: https://www.volcengine.com/docs/82379/1494384
[ark-think]: https://www.volcengine.com/docs/82379/1449737
[az-claude]: https://learn.microsoft.com/en-us/azure/foundry/foundry-models/concepts/claude-models
[bedrock-invoke]: https://docs.aws.amazon.com/bedrock/latest/APIReference/API_runtime_InvokeModel.html
[ds-home]: https://api-docs.deepseek.com/
[ds-rate]: https://api-docs.deepseek.com/quick_start/rate_limit
[ds-resp]: https://api-docs.deepseek.com/guides/responses_api
[ds-think]: https://api-docs.deepseek.com/guides/thinking_mode
[g-batch]: https://ai.google.dev/gemini-api/docs/batch-api
[g-cache]: https://ai.google.dev/gemini-api/docs/generate-content/caching
[g-fc]: https://ai.google.dev/gemini-api/docs/generate-content/function-calling
[g-gen]: https://ai.google.dev/api/generate-content
[g-int-api]: https://ai.google.dev/api/interactions-api
[g-int-mig]: https://ai.google.dev/gemini-api/docs/migrate-to-interactions
[g-int-ov]: https://ai.google.dev/gemini-api/docs/interactions-overview
[g-sig]: https://ai.google.dev/gemini-api/docs/thought-signatures
[g-think]: https://ai.google.dev/gemini-api/docs/generate-content/thinking
[kimi-chat]: https://platform.kimi.ai/docs/api/chat
[kimi-ov]: https://platform.kimi.ai/docs/api/overview
[kimi-resp]: https://platform.kimi.ai/docs/api/responses
[kimi-think]: https://platform.kimi.ai/docs/guide/use-thinking-models
[kimi-trouble]: https://platform.kimi.ai/docs/guide/troubleshooting
[mm-anth]: https://platform.minimax.io/docs/api-reference/text-anthropic-api
[mm-oai]: https://platform.minimax.io/docs/api-reference/text-openai-api
[mm-resp]: https://platform.minimax.io/docs/api-reference/responses-create
[mm-tg]: https://platform.minimax.io/docs/guides/text-generation
[oa-audio]: https://developers.openai.com/api/docs/guides/audio-chat-completions
[oa-batch]: https://developers.openai.com/api/docs/guides/batch
[oa-bg]: https://developers.openai.com/api/docs/guides/background
[oa-cache]: https://developers.openai.com/api/docs/guides/prompt-caching
[oa-chat]: https://developers.openai.com/api/reference/resources/chat/subresources/completions/methods/create/
[oa-errors]: https://developers.openai.com/api/docs/guides/error-codes
[oa-fc]: https://developers.openai.com/api/docs/guides/function-calling
[oa-migrate]: https://developers.openai.com/api/docs/guides/migrate-to-responses
[oa-py]: https://developers.openai.com/api/reference/python/
[oa-py-readme]: https://github.com/openai/openai-python/blob/main/README.md
[oa-reasoning]: https://developers.openai.com/api/docs/guides/reasoning
[oa-resp]: https://developers.openai.com/api/reference/resources/responses/methods/create/
[oa-resp-events]: https://developers.openai.com/api/reference/resources/responses/streaming-events/
[or-resp]: https://openrouter.ai/docs/api/reference/responses/error-handling
[xai-async]: https://docs.x.ai/developers/advanced-api-usage/async
[xai-cmp]: https://docs.x.ai/developers/model-capabilities/text/comparison
[xai-legacy]: https://docs.x.ai/developers/rest-api-reference/inference/legacy
[xai-reason]: https://docs.x.ai/developers/model-capabilities/text/reasoning
[zp-chat]: https://docs.bigmodel.cn/api-reference/%E6%A8%A1%E5%9E%8B-api/%E5%AF%B9%E8%AF%9D%E8%A1%A5%E5%85%A8
[zp-think]: https://docs.bigmodel.cn/cn/guide/capabilities/thinking
