# 字段对照（第 1 层）

> 写代码时查的地图：同一个意思，在各家叫什么、放在哪、默认值是什么。标 **重点** 的是最容易写错或最常用的。先读过 [导读](llm-api-protocols.md) 再来查。
> 链接文字写明来源的厂商和文档，完整列表见 [来源](details/sources.md)；“冲突”“未核实”汇总在 [冲突与未核实](details/conflicts.md)。

## 0. 速查：同一个意思，五种格式怎么写

**重点** = 最常用或最容易写错。第三方列只写和 OpenAI / Anthropic 写法不同的地方，完整对照在第 7 节；错误体和重试规则见 [错误、重试与限流](details/ops.md)。

| 意思 | Chat Completions | Responses | Anthropic Messages | `generateContent` | Interactions | 第三方常见差异 | 来源 |
|---|---|---|---|---|---|---|---|
| 系统指令 | `developer` / `system` 消息 | 顶层 `instructions`（续历史时不继承） | 顶层 `system` | `systemInstruction` | `system_instruction` | Qwen 兼容模式 system 只能放第一条 | [OpenAI Chat Completions 参考][oa-chat]；[OpenAI 迁移到 Responses 指南][oa-migrate]；[Anthropic Messages 参考][an-msg]；[Gemini generateContent 参考][g-gen]；[Gemini Interactions 参考][g-int-api]；[阿里云百炼 OpenAI 兼容说明][ali-compat] |
| 最大输出长度 | `max_completion_tokens`（含推理；`max_tokens` 已废弃） | `max_output_tokens`（含推理） | `max_tokens`，**必填** | `generationConfig.maxOutputTokens`（含思考 token） | `generation_config.max_output_tokens` | DeepSeek Chat 只认 `max_tokens`；豆包多数模型的 `max_tokens` 不限思考 | [OpenAI Chat Completions 参考][oa-chat]；[OpenAI Responses 参考][oa-resp]；[Anthropic Messages 参考][an-msg]；[Gemini generateContent 参考][g-gen]；[Gemini generateContent 思考文档][g-think]；[Gemini Interactions 参考][g-int-api]；[DeepSeek Chat 接口参考][ds-chat]；[火山方舟 Chat 接口参考][ark-chat] |
| 停止序列 | `stop`，最多 4 个（`o3` / `o4-mini` 不支持） | 没有 `stop` 字段 | `stop_sequences` | `generationConfig.stopSequences`，最多 5 个 | `generation_config.stop_sequences` | DeepSeek 最多 16 个；Kimi 5 个且每个 ≤ 32 字节；xAI 推理模型传了报错 | [OpenAI Chat Completions 参考][oa-chat]；[OpenAI Responses 参考][oa-resp]；[Anthropic Messages 参考][an-msg]；[Gemini generateContent 参考][g-gen]；[Gemini Interactions 参考][g-int-api]；[DeepSeek Chat 接口参考][ds-chat]；[Kimi Chat 接口参考][kimi-chat]；[xAI 推理指南][xai-reason] |
| 采样参数 | `temperature` 0–2、`top_p` 等 | `temperature` 0–2、`top_p` | 指南写 Claude 4.7+ 只接受默认值、传别的 400；参考页写 `top_p` ≥ 0.99 也接受（冲突） | `temperature` 0–2 | `generation_config.temperature` | DeepSeek 思考模式下不生效也不报错；Kimi 是固定值，传别的报错；xAI 推理模型传 penalty 会报错 | [OpenAI Chat Completions 参考][oa-chat]；[OpenAI Responses 参考][oa-resp]；[Anthropic Messages 使用指南][an-working]；[Anthropic Messages 参考][an-msg]；[Gemini generateContent 参考][g-gen]；[Gemini 文本生成文档（Interactions）][g-int-text]；[DeepSeek 思考模式指南][ds-think]；[Kimi 模型总览][kimi-models]；[xAI 推理指南][xai-reason] |
| **思考开关 / 力度** | `reasoning_effort` | `reasoning.effort`，摘要要开 `reasoning.summary` | `thinking: {type:"adaptive"}` + `output_config.effort` | `thinkingConfig.thinkingLevel`（3+）/ `thinkingBudget`（2.5） | `generation_config.thinking_level` | `thinking.type`（DeepSeek、智谱、Kimi K2、豆包、MiniMax）；`enable_thinking`（Qwen）；同名 `reasoning_effort` 取值各家不同 | [OpenAI Chat Completions 参考][oa-chat]；[OpenAI Responses 参考][oa-resp]；[Anthropic Messages 参考][an-msg]；[Gemini generateContent 思考文档][g-think]；[Gemini Interactions 参考][g-int-api]；[DeepSeek 思考模式指南][ds-think]；[智谱思考模式指南][zp-think]；[Kimi Chat 接口参考][kimi-chat]；[火山方舟深度思考文档][ark-think]；[MiniMax OpenAI 兼容接口][mm-oai]；[阿里云百炼深度思考指南][ali-think] |
| **思考回传** | 无推理文本 | reasoning item + `encrypted_content` | thinking 块 + `signature`，原样按序带回 | `thoughtSignature` | 不存历史时带回 `thought` step | `reasoning_content`（DeepSeek 带 `tools` 时必须带回，否则 400）；豆包、xAI Responses 用 `encrypted_content` | [OpenAI 推理指南][oa-reasoning]；[Anthropic Messages 参考][an-msg]；[Gemini 思考签名文档][g-sig]；[Gemini 文本生成文档（Interactions）][g-int-text]；[DeepSeek 思考模式指南][ds-think]；[火山方舟 Chat 接口参考][ark-chat]；[xAI 推理指南][xai-reason] |
| 声明工具 | `tools[].function.{name, parameters}` | `tools[].{type:"function", name, parameters}` | `tools[].{name, input_schema}` | `tools[].functionDeclarations[]` | `tools[].{type:"function", name, parameters}` | 智谱另有 `web_search` 等工具类型 | [OpenAI 函数调用指南][oa-fc]；[Anthropic 定义工具文档][an-tools]；[Gemini generateContent 函数调用指南][g-fc]；[Gemini Interactions 参考][g-int-api]；[智谱函数调用指南][zp-fc] |
| **回传工具结果** | `role:"tool"` + `tool_call_id` | `function_call_output` + `call_id` | user 轮里 `tool_result` + `tool_use_id` | user 轮里 `functionResponse` | 单独的 `function_result` step + `call_id` | 同 OpenAI / Anthropic 写法 | [OpenAI 函数调用指南][oa-fc]；[Anthropic 定义工具文档][an-tools]；[Gemini generateContent 函数调用指南][g-fc]；[Gemini Interactions 参考][g-int-api] |
| 强制调用工具 | `tool_choice: "required"` 或指定函数 | 同左（指定函数是平铺写法） | `tool_choice: {type:"any"}` / `{type:"tool", name}` | `functionCallingConfig.mode: "ANY"` | `generation_config.tool_choice: "any"` | 智谱只有 `"auto"`；DeepSeek 思考模式下强制会 400 | [OpenAI Chat Completions 参考][oa-chat]；[OpenAI 函数调用指南][oa-fc]；[Anthropic Messages 参考][an-msg]；[Gemini generateContent 函数调用指南][g-fc]；[Gemini Interactions 参考][g-int-api]；[智谱函数调用指南][zp-fc]；[DeepSeek Chat 接口参考][ds-chat] |
| 结构化输出 | `response_format: {type:"json_schema", …}` | `text.format` | `output_config.format` | `responseMimeType` + `responseJsonSchema` | `response_format` | DeepSeek、智谱 Chat 只有 `json_object` | [OpenAI Chat Completions 参考][oa-chat]；[OpenAI 迁移到 Responses 指南][oa-migrate]；[Anthropic 结构化输出文档][an-so]；[Gemini generateContent 参考][g-gen]；[Gemini Interactions 参考][g-int-api]；[DeepSeek Chat 接口参考][ds-chat]；[智谱对话补全参考][zp-chat] |
| **图片输入** | `{type:"image_url", image_url:{url, detail}}` | `{type:"input_image", image_url 或 file_id}` | `{type:"image", source:{type:"base64" / "url" / "file"}}` | `{inlineData:{mimeType, data}}` / `{fileData:{fileUri}}` | `{type:"image", data, mime_type}`（或给 `uri`） | DeepSeek `detail` 多 `original`；Kimi 用 `ms://<file_id>` | [OpenAI 图片输入指南][oa-vision]；[Anthropic 图片输入文档][an-vision]；[Gemini generateContent 参考][g-gen]；[Gemini Interactions 参考][g-int-api]；[DeepSeek 图片输入指南][ds-vision]；[Kimi Chat 接口参考][kimi-chat] |
| 流式开关 | `stream: true`；要用量加 `stream_options.include_usage` | `stream: true` | `stream: true` | 换路径 `:streamGenerateContent?alt=sse` | `stream: true` | 用量出现在哪一块各家不同 | [OpenAI Chat Completions 参考][oa-chat]；[OpenAI Responses 参考][oa-resp]；[Anthropic 流式文档][an-stream]；[Gemini generateContent 参考][g-gen]；[Gemini 流式文档（Interactions）][g-int-stream] |
| 缓存 | 自动；可给 `prompt_cache_key` | 同左 | `cache_control`（自动或手动断点） | 自动；或建 `cachedContents` 再传 `cachedContent` | 只有自动 | DeepSeek、智谱自动；Kimi Anthropic 入口默认不写缓存 | [OpenAI 提示缓存指南][oa-cache]；[Anthropic 提示缓存文档][an-cache]；[Gemini generateContent 缓存文档][g-cache]；[Gemini 缓存文档][g-cache2]；[DeepSeek 上下文缓存指南][ds-cache]；[智谱上下文缓存指南][zp-cache]；[Kimi Messages 接口参考][kimi-msg] |

## 1. 请求骨架（五种原生格式）

| | Chat Completions | Responses | Anthropic Messages | Gemini `generateContent`（legacy） | Gemini Interactions（推荐） |
|---|---|---|---|---|---|
| 入口（路径） | `POST /v1/chat/completions`（[OpenAI Chat Completions 参考][oa-chat]） | `POST /v1/responses`（[OpenAI Responses 参考][oa-resp]） | `POST /v1/messages`（[Anthropic Messages 参考][an-msg]） | `POST /v1beta/models/{model}:generateContent`<br>流式：改成 `:streamGenerateContent?alt=sse`（[Gemini generateContent 参考][g-gen]） | `POST /v1beta/interactions`（`/v1/interactions` 也已 GA）；流式同一路径，body 里 `"stream": true`（[Gemini Interactions 参考][g-int-api]；[Gemini API 版本说明][g-ver]） |
| 对话状态 | 服务端不替你续对话，每次发全量历史 | 可用 `previous_response_id` 或 `conversation`（二选一）；`store` 默认开（[OpenAI 迁移到 Responses 指南][oa-migrate]） | 不存 | 不存（可引用 `cachedContent`）（[Gemini generateContent 参考][g-gen]） | 默认存（`store=true`，付费 55 天、免费 1 天），用 `previous_interaction_id` 续；`tools` `system_instruction` `generation_config` 每轮要重发（[Gemini Interactions 概览][g-int-ov]） |
| 历史怎么装 | `messages[]` | `input`：字符串或 items 数组 | `messages[]`，内容是 content blocks | `contents[]`，内容是 `parts[]` | `input`：字符串、Content 或 steps 数组；历史是一串 steps（`user_input` `thought` `function_call` `function_result` `model_output`）（[Gemini Interactions 参考][g-int-api]；[Gemini Interactions 概览][g-int-ov]） |
| 工具参数 | JSON 字符串（`function.arguments`） | JSON 字符串（`arguments`） | JSON 对象（`input`） | JSON 对象（`args`） | JSON 对象（`arguments`；流式时是字符串片段） |

来源：[OpenAI Chat Completions 参考][oa-chat]；[OpenAI Responses 参考][oa-resp]；[Anthropic Messages 参考][an-msg]；[Gemini generateContent 参考][g-gen]；[Gemini Interactions 参考][g-int-api]；工具参数的完整对照见第 2 节。

更多行：

| | Chat Completions | Responses | Anthropic Messages | Gemini `generateContent`（legacy） | Gemini Interactions（推荐） |
|---|---|---|---|---|---|
| 鉴权 | `Authorization: Bearer`（[OpenAI API 参考总览][oa-overview]） | 与 Chat Completions 相同 | `Authorization: Bearer` 为主，`x-api-key` 作为旧写法仍支持（[Anthropic API 总览][an-overview]） | API 总览要求请求头 `x-goog-api-key`，但部分官方 curl 只用 `?key=`（两处不一致）；Vertex 用 OAuth Bearer，`generateContent` 也可用 API key（[Gemini API 总览][g-api]；[Gemini generateContent 参考][g-gen]；[Vertex AI 鉴权文档][vx-auth]） | 请求头 `x-goog-api-key`（[Gemini API 总览][g-api]） |
| 版本 | 路径 `/v1`（[OpenAI API 参考总览][oa-overview]） | 与 Chat Completions 相同 | 必填头 `anthropic-version`（如 `2023-06-01`），beta 功能用 `anthropic-beta`（[Anthropic beta 请求头文档][an-beta]） | 路径 `v1` / `v1beta`；缓存、Live 等只在 `v1beta`（[Gemini API 版本说明][g-ver]） | 路径 `v1` / `v1beta`；快速入门 curl 带的 `Api-Revision` 头在 2026-06-08 旧 schema 下线后已被忽略（[Gemini Interactions 快速入门][g-int-qs]；[Gemini Interactions 2026-05 不兼容变更说明][g-int-break]） |
| 系统指令 | `developer` 或 `system` 消息（o1 起推荐 `developer`）（[OpenAI Chat Completions 参考][oa-chat]） | 顶层 `instructions`（用 `previous_response_id` 时**不会**继承，要重发）（[OpenAI 迁移到 Responses 指南][oa-migrate]） | 顶层 `system`；部分新模型允许对话中途插 `role:"system"`（[Anthropic 对话中途 system 消息文档][an-midsys]） | 顶层 `systemInstruction`（只能文本）（[Gemini generateContent 参考][g-gen]） | 顶层 `system_instruction`（[Gemini Interactions 参考][g-int-api]） |
| 角色 | `developer` `system` `user` `assistant` `tool` | `developer` `system` `user` `assistant`；没有 `tool`，工具结果是 `function_call_output` item（[OpenAI Responses 参考][oa-resp]） | `user` `assistant` | `user` `model` | 没有角色，用 step 类型区分（[Gemini Interactions 概览][g-int-ov]） |
| 必填 | `model`、`messages` | 参考页把 `model`、`input` 都标为可选（[OpenAI Responses 参考][oa-resp]） | `model`、`max_tokens`、`messages`（[Anthropic Messages 参考][an-msg]） | `contents` | `input`，以及 `model` 或 `agent` 二选一（[Gemini Interactions 参考][g-int-api]） |
| 输出 | `choices[].message`，`n` 可多条（[OpenAI OpenAPI 定义][oa-openapi]） | `output[]` items，无 `n`；SDK 提供 `output_text`（[OpenAI 迁移到 Responses 指南][oa-migrate]） | `content[]` blocks | `candidates[].content.parts[]` | `steps[]`（create 只返回模型生成的 steps）（[Gemini Interactions 概览][g-int-ov]） |
| 停止原因 | `finish_reason`：`stop` `length` `tool_calls` `content_filter`（另有已废弃的 `function_call`）（[OpenAI Chat Completions 参考][oa-chat]） | `status` + `incomplete_details.reason`（[OpenAI Responses 参考][oa-resp]） | `stop_reason`：`end_turn` `max_tokens` `stop_sequence` `tool_use` `pause_turn` `refusal` `model_context_window_exceeded`（[Anthropic Messages 参考][an-msg]） | `finishReason`：`STOP` `MAX_TOKENS` `SAFETY` `MALFORMED_FUNCTION_CALL`，完整枚举见参考页（[Gemini generateContent 参考][g-gen]） | `status`（如 `completed`、`requires_action`）（[Gemini Interactions 参考][g-int-api]） |
| 用量字段 | `prompt_tokens` / `completion_tokens`；缓存在 `prompt_tokens_details.cached_tokens`，推理在 `completion_tokens_details.reasoning_tokens`（[OpenAI OpenAPI 定义][oa-openapi]） | `input_tokens` / `output_tokens`（[OpenAI Responses 参考][oa-resp]） | `input_tokens`（不含缓存）/ `output_tokens` / `cache_creation_input_tokens` / `cache_read_input_tokens`（[Anthropic Messages 参考][an-msg]） | `usageMetadata.promptTokenCount` / `candidatesTokenCount` / `thoughtsTokenCount`（[Gemini generateContent 参考][g-gen]） | `usage.total_input_tokens` / `total_output_tokens` / `total_thought_tokens`（[Gemini Interactions 参考][g-int-api]） |
| 内置工具 | `web_search_options`（[OpenAI Chat Completions 参考][oa-chat]） | `web_search` `file_search` `code_interpreter` `computer_use_preview` `image_generation` `mcp` `shell` `apply_patch` 等（[OpenAI Responses 参考][oa-resp]） | 服务端工具，类型名带日期（如 `web_search_20260318`）（[Anthropic 服务端工具文档][an-server-tools]） | `googleSearch` `codeExecution` `urlContext` `fileSearch` 等（[Gemini generateContent 参考][g-gen]） | 有：参考页定义了 `google_search`、`code_execution`、`url_context`、`file_search` 等（[Gemini Interactions 参考][g-int-api]） |

## 2. 工具 **重点**

| | 声明工具 | 模型发起调用 | 你回传结果 | 强制调用 |
|---|---|---|---|---|
| Chat | `tools[].function.{name,description,parameters,strict}` | `message.tool_calls[]`，`function.arguments` 是 JSON **字符串** | `{role:"tool", tool_call_id, content}` | `tool_choice: "required"` 或 `{type:"function", function:{name}}` |
| Responses | `tools[].{type:"function", name, parameters, strict}`（**平铺**，没有 `function` 这层）| `output[]` 里的 `{type:"function_call", call_id, arguments}`，也是字符串 | `{type:"function_call_output", call_id, output}`，按 `call_id` 对应（不是 `id`）| `"required"` 或 `{type:"function", name}` |
| Anthropic | `tools[].{name, description, input_schema}` | `content[]` 里的 `{type:"tool_use", id, input}`，`input` 是**对象** | 下一条 **user** 消息里放 `{type:"tool_result", tool_use_id, content}`；同一条 user 消息里，文字块要排在 `tool_result` 后面 | `tool_choice: {type:"any"}` 或 `{type:"tool", name}`（Fable 5.1、Mythos 5.1、Opus 5.5 不接受这两种） |
| Gemini `generateContent` | `tools[].functionDeclarations[].{name, parameters \| parametersJsonSchema}` | `parts[]` 里的 `{functionCall:{id, name, args}}`，`args` 是**对象** | **user** 轮的 `parts[]` 里放 `{functionResponse:{id, name, response}}` | `toolConfig.functionCallingConfig.mode: "ANY"` |
| Gemini Interactions | `tools[].{type:"function", name, description, parameters}` | `steps[]` 里的 `{type:"function_call", id, name, arguments}`，`arguments` 是**对象**；`status` 变为 `requires_action` | 单独一个 `{type:"function_result", call_id, name, result}` step 作为下一次的 `input`（不是放进 user 轮） | `generation_config.tool_choice`：`auto` / `any` / `none` / `validated` |

来源：[OpenAI 函数调用指南][oa-fc]；[OpenAI 迁移到 Responses 指南][oa-migrate]；[Anthropic 定义工具文档][an-tools]；[Anthropic 处理工具调用指南][an-handle]；[Gemini generateContent 函数调用指南][g-fc]；[Gemini Interactions 参考][g-int-api]；[Gemini Interactions 快速入门][g-int-qs]。

可直接复制的第二轮请求示例见 [工具调用示例](details/tool-calls.md)。

## 3. 流式

| | Chat Completions | Responses | Anthropic Messages | Gemini `generateContent` | Gemini Interactions |
|---|---|---|---|---|---|
| 传输 | SSE（服务器推送事件），只有 `data:` 行（[OpenAI Chat Completions 参考][oa-chat]） | SSE，官方示例里 `event:` 行与 JSON 的 `type` 一致；另有 WebSocket 模式（[OpenAI Responses 参考][oa-resp]；[OpenAI WebSocket 模式指南][oa-ws]） | SSE，`event:` + `data:`（[Anthropic 流式文档][an-stream]） | SSE（要加 `?alt=sse`）；实时语音走 Live API（WebSocket）（[Gemini generateContent 参考][g-gen]；[Gemini Live API 参考][g-live]） | SSE：body 里 `"stream": true`（快速入门 URL 另加 `?alt=sse`）（[Gemini 流式文档（Interactions）][g-int-stream]；[Gemini Interactions 快速入门][g-int-qs]） |
| 事件 | 不分类型，每块都是 `chat.completion.chunk` | 50 多种带类型事件，如 `response.output_text.delta`（[OpenAI Responses 流式事件参考][oa-resp-events]） | `message_start` → `content_block_start/delta/stop` → `message_delta` → `message_stop`，另有 `ping`、`error`（[Anthropic 流式文档][an-stream]） | 每块都是一个完整的 `GenerateContentResponse` | `interaction.created` `interaction.status_update` `step.start` `step.delta` `step.stop` `interaction.completed`（[Gemini 流式文档（Interactions）][g-int-stream]） |
| 怎么算结束 | `data: [DONE]` | `response.completed` / `failed` / `incomplete`（官方示例没有 `[DONE]`） | `message_stop` | 没有结束标记，看 `finishReason` | `event: done` / `data: [DONE]`（[Gemini 流式文档（Interactions）][g-int-stream]） |
| 用量在哪 | 要开 `stream_options.include_usage`，`[DONE]` 前多一块，`choices` 为空（[OpenAI Chat Completions 参考][oa-chat]） | `response.completed` 带的 response 里 | `message_start` 带输入量（输出量是占位值），`message_delta.usage` 是**累计值**（[Anthropic 流式文档][an-stream]） | 官方示例中间块也带 `usageMetadata`（[Gemini generateContent 文本生成文档][g-textgen]） | `interaction.completed` 事件带 `usage`（[Gemini 流式文档（Interactions）][g-int-stream]） |
| 工具参数 | `delta.tool_calls[i].function.arguments` 碎片按 `index` 拼接（拼接规则 REST 文档没写，只在 Node SDK 文档里）（[OpenAI Chat 流式事件参考][oa-chat-stream]；[OpenAI Node SDK helpers 文档][oa-node-helpers]） | `response.function_call_arguments.delta` / `.done`（[OpenAI Responses 流式事件参考][oa-resp-events]） | `input_json_delta.partial_json` 分片，拼完是对象（[Anthropic 流式文档][an-stream]） | `functionCall.args` 是对象；流式时会不会跨块增长，文档没写（[Gemini generateContent 函数调用指南][g-fc]） | `arguments_delta` 是 JSON **字符串**片段，要自己累加；拼完的 step 里 `arguments` 是对象（[Gemini 流式文档（Interactions）][g-int-stream]） |
| 思考内容 | 没有 | `response.reasoning_summary_text.delta`（要开 `reasoning.summary`）（[OpenAI 推理指南][oa-reasoning]） | `thinking_delta` + `signature_delta`（[Anthropic 流式文档][an-stream]） | `thought: true` 的 part（[Gemini generateContent 思考文档][g-think]） | `thought` step，delta 里有 `thought_signature`（[Gemini 流式文档（Interactions）][g-int-stream]） |
| 中途出错 | REST 文档没写；官方 Python SDK 把带 `error` 键的数据块当错误抛出（[OpenAI Python SDK 源码（流式解析）][oa-py-stream]） | `error` 事件 | 流里以 `event: error` 下发（例如 `overloaded_error`，非流式时对应 HTTP 529）（[Anthropic 流式文档][an-stream]） | 文档没描述流内错误事件；看 `finishReason` 或 HTTP 状态 | `event: error`，带 `error.message`、`error.code`（[Gemini 流式文档（Interactions）][g-int-stream]） |

第三方入口的流式差异见 [流式、停止与断开](details/streaming.md)。

## 4. 结构化输出

| | 怎么开 | 严格模式允许的 JSON Schema | 来源 |
|---|---|---|---|
| Chat | `response_format: {type:"json_schema", json_schema:{name, schema, strict}}`；老的 `json_object` 还要求提示里写明“输出 JSON” | 所有字段必须进 `required`、`additionalProperties:false`、根必须是 object；支持 `anyOf` `$defs/$ref` `pattern` `format` 数值与数组上下界；不支持 `allOf` `not` `if/then/else` | [OpenAI Chat Completions 参考][oa-chat]；[OpenAI 结构化输出指南][oa-so] |
| Responses | `text.format: {type:"json_schema", name, schema, strict}`（字段和 `type` 平级）；function 工具**不写 `strict` 时默认尝试严格**，Chat 相反 | 同上 | [OpenAI 迁移到 Responses 指南][oa-migrate] |
| Anthropic | `output_config.format: {type:"json_schema", schema}`；工具上写 `strict: true` | `additionalProperties` 只能 `false`；不支持数值上下界、字符串长度、递归、外部 `$ref`；`minItems` 只能 0 或 1 | [Anthropic 结构化输出文档][an-so] |
| Gemini `generateContent` | `generationConfig.responseMimeType: "application/json"` + `responseJsonSchema`（`responseSchema` 已标 deprecated；参考页对这几个字段的说明自相矛盾，见 [冲突与未核实](details/conflicts.md)）；工具没有 `strict`，用 `VALIDATED` / `ANY` 模式约束 | 支持 `$ref/$defs` `anyOf/oneOf` `prefixItems` 数值与数组上下界、非标准 `propertyOrdering`；`$ref` 旁不能写别的属性 | [Gemini generateContent 参考][g-gen]；[Gemini generateContent 函数调用指南][g-fc] |
| Gemini Interactions | 顶层 `response_format`：文本格式写 `type: "text"`，`mime_type` 为 `application/json` 或 `text/plain`；`schema` 只在 `application/json` 时生效；另有音频、图片、视频输出格式 | 参考页没列关键字限制 | [Gemini Interactions 参考][g-int-api] |

## 5. 推理（思考）：怎么开，要不要回传 **重点**

| | 怎么开 / 调力度 | 推理内容在哪 | 下一轮要不要回传 | 来源 |
|---|---|---|---|---|
| OpenAI Chat | `reasoning_effort`（`none` 到 `max` 共 7 档，因模型而异） | 不返回推理文本，只有 `reasoning_tokens` 计数 | 不涉及 | [OpenAI Chat Completions 参考][oa-chat] |
| OpenAI Responses | `reasoning.effort`；`reasoning.summary` 要主动开 | `reasoning` item（摘要）+ `encrypted_content`（密文） | 不存对话（`store:false` 或零数据保留）时要把 reasoning item 原样带回。推理指南和迁移指南说这时密文默认返回、`include` 不再必需；create 参考页的字段说明写的是“默认填充”，范围更宽。显式加 `include: ["reasoning.encrypted_content"]` 最稳（冲突，见 [冲突与未核实](details/conflicts.md)） | [OpenAI Responses 参考][oa-resp]；[OpenAI 推理指南][oa-reasoning]；[OpenAI 迁移到 Responses 指南][oa-migrate] |
| Anthropic | `thinking: {type:"adaptive"}` + `output_config.effort`；老写法 `{type:"enabled", budget_tokens}` 在 4.7+ 直接 400（Mythos Preview 除外） | `thinking` / `redacted_thinking` 块，带 `signature` | 必须原样、按原顺序带回，否则 400 | [Anthropic Messages 参考][an-msg]；[Anthropic 扩展思考文档][an-think]；[Anthropic 错误文档][an-errors] |
| Gemini `generateContent` | Gemini 3 用 `thinkingLevel`（仍兼容 `thinkingBudget`，Gemini 3 页要求两者别同时传），2.5 用 `thinkingBudget`；`includeThoughts` 返回摘要 | `thought: true` 的 part + `thoughtSignature` | Gemini 3：当前这一轮里每个 step 的第一个 `functionCall` 缺签名就 400（并行调用时签名只在第一个上）；2.5 可选 | [Gemini generateContent 思考文档][g-think]；[Gemini 3 开发指南][g-3]；[Gemini 思考签名文档][g-sig] |
| Gemini Interactions | `generation_config.thinking_level`：`minimal/low/medium/high`；`thinking_summaries`：`auto/none` | `thought` step，带 `signature` | 用 `previous_interaction_id` 时不用重发；不存历史时必须把 `thought`、`function_call` 等模型生成的 steps 原样带回（里面有签名） | [Gemini Interactions 参考][g-int-api]；[Gemini 文本生成文档（Interactions）][g-int-text] |
| DeepSeek | `thinking.type`（默认开，OpenAI SDK 放 `extra_body`）+ `reasoning_effort`：`none/low/high/max`（`none` 关闭思考），`medium` 会被当成 `high`；思考指南的 Chat 部分只列 `low/high/max`，与 API 页不一致 | `reasoning_content` | 请求**不带** `tools`：不用回传（传了也被忽略，不进上下文）；**带** `tools`：之后每轮都必须回传，否则 400 | [DeepSeek 思考模式指南][ds-think]；[DeepSeek Chat 接口参考][ds-chat] |
| 智谱 GLM | `thinking: {type, clear_thinking}` + `reasoning_effort` | `reasoning_content` | 默认会丢掉历史思考；设 `clear_thinking:false` 时要原样回传 | [智谱思考模式指南][zp-think] |
| Kimi | K3 用 `reasoning_effort`：`low/high/max`，默认 `max`；K2.x 用 `thinking: {type, keep}` | `reasoning_content`；Anthropic 入口是 thinking 块 | 开了保留思考（K3 始终开）就要原样回传 | [Kimi 模型总览][kimi-models]；[Kimi 思考模型指南][kimi-think] |
| xAI Grok | `reasoning.effort` / `reasoning_effort`：grok-4.6/4.7 为 `low/medium/high/xhigh`，默认 `high`，**关不掉**；grok-4.5 只有 `low/medium/high`（`xhigh` 当 `high`）；grok-4.3 支持 `none`，默认 `low` | Chat：对比页写不返回推理内容，流式指南的 Chat 示例里却有 `delta.reasoning_content`（冲突）；Responses：`encrypted_content`（`grok-4.7` 每次都返回） | Responses 自管历史时把密文 item 原样放回 `input`；或开 `store` 后用 `previous_response_id` | [xAI 推理指南][xai-reason]；[xAI grok-4.3 模型页][xai-43]；[xAI 接口对比页][xai-cmp]；[xAI 流式指南][xai-stream] |
| Qwen | Chat：`enable_thinking` + `thinking_budget`（Python SDK 放 `extra_body`，Node/curl 放顶层）；Responses：`reasoning.effort`（`enable_thinking` 将废弃） | `reasoning_content` | `preserve_thinking` 为 true 时（默认 false，部分型号默认 true，各页列的型号不一致）要把历史思考放回 `reasoning_content` 字段，不要拼进 `content` | [阿里云百炼深度思考指南][ali-think]；[阿里云百炼 Chat 兼容接口参考][ali-chat]；[阿里云百炼 Responses 兼容接口][ali-resp] |
| MiniMax | `thinking.type`：`adaptive` / `disabled`；M3 在 Chat Completions 入口默认开，在 Anthropic 和 Responses 入口默认关 | `reasoning_split:true` 时在 `reasoning_content` + `reasoning_details`；否则以 `<think>…</think>` 混在 `content` 里 | 整条 assistant 消息原样带回，**不要**把 `<think>` 剥掉 | [MiniMax OpenAI 兼容接口][mm-oai]；[MiniMax Anthropic 兼容接口][mm-anth]；[MiniMax Responses 接口参考][mm-resp]；[MiniMax M3 函数调用指南][mm-fc] |
| 豆包 | `thinking.type`：`enabled` / `disabled` / `auto` + `reasoning_effort` | `reasoning_content`；新模型另有 `encrypted_content` | doubao-seed-2-0-lite-260428 起要带回 `encrypted_content`：不带不会报错，但推理效果下降；密文被篡改则无法还原（Chat 参考页写会报 `Invalid signature`，思考页没写，冲突）；同时给两者时以密文为准 | [火山方舟深度思考文档][ark-think]；[火山方舟 Chat 接口参考][ark-chat] |

## 6. 图片、文件、音频 **重点**

| | 图片 | 文件 / PDF | 音频 | 大小上限 | 来源 |
|---|---|---|---|---|---|
| Chat Completions | `{type:"image_url", image_url:{url, detail}}`，`url` 可以是网址或 base64 data URL | `{type:"file", file:{file_data 或 file_id}}`，只收 PDF，不支持 `file_url` | `{type:"input_audio", input_audio:{data, format}}`，只收 base64 的 `wav` / `mp3` | 单请求最多 512 MB、1500 张图；文件每个和合计都 ≤ 50 MB | [OpenAI 图片输入指南][oa-vision]；[OpenAI 文件输入指南][oa-file]；[OpenAI Chat 音频指南][oa-audio] |
| Responses | `{type:"input_image", image_url 或 file_id, detail}` | `{type:"input_file", file_url 或 file_id 或 file_data}` | 没有音频输入；音频对话文档要求改走 Chat Completions | 同上 | [OpenAI 图片输入指南][oa-vision]；[OpenAI 文件输入指南][oa-file]；[OpenAI Chat 音频指南][oa-audio] |
| Anthropic | `{type:"image", source:{type:"base64" / "url" / "file", …}}` | `{type:"document", source:…}`，PDF 可用网址、base64 或 `file_id` | — | 单图 base64 后 ≤ 10 MB（Bedrock、Vertex 上是 5 MB）、≤ 8000×8000 px；每请求最多 600 张（200k 上下文的模型 100 张）；整个请求 ≤ 32 MB | [Anthropic 图片输入文档][an-vision]；[Anthropic PDF 支持文档][an-pdf] |
| `generateContent` | `{inlineData:{mimeType, data}}` 或 `{fileData:{mimeType, fileUri}}` | 和图片相同的两种 part，`mimeType` 填 `application/pdf` | 同样两种 part，`mimeType` 填 `audio/*` | 图片页写整请求 ≤ 20 MB，文件页写内联 ≤ 100 MB（冲突，见 [冲突与未核实](details/conflicts.md)）；Files API 单文件 2 GB | [Gemini generateContent 参考][g-gen]；[Gemini 图片理解文档][g-image]；[Gemini 文件输入方式文档][g-file] |
| Interactions | `{type:"image", data, mime_type}`，也可给 `uri` | `{type:"document", data 或 uri, mime_type}` | `{type:"audio", data 或 uri, mime_type}` | 图片大小上限未核实 | [Gemini Interactions 参考][g-int-api] |
| 第三方 Chat 兼容 | DeepSeek `detail` 多一个 `original`；Kimi 用 `ms://<file_id>`，无 `detail`；智谱无 `detail`，单图 ≤ 5 MB | DeepSeek `{type:"file", file_id}`（id 在顶层）；智谱 `file_url` 最多 50 个 | Qwen `input_audio` 可以给网址 | 各家不同，见来源 | [DeepSeek 图片输入指南][ds-vision]；[Kimi Chat 接口参考][kimi-chat]；[智谱对话补全参考][zp-chat]；[阿里云百炼 Chat 兼容接口参考][ali-chat] |

Kimi、Qwen、智谱还接受 `video_url`（Qwen 另有 `video` 图片序列）（[Kimi Chat 接口参考][kimi-chat]；[阿里云百炼 Chat 兼容接口参考][ali-chat]；[智谱对话补全参考][zp-chat]）。

## 7. 第三方同义字段（Chat 兼容入口）

✗ = 文档明确不支持；“未列出” = 角色表 / 参数表里没有，但没写会被拒绝；— = 文档没写。推理相关见 [第 5 节](#5-推理思考怎么开要不要回传-重点)，流式用量见 [流式、停止与断开](details/streaming.md)。

| 厂商 | `tool_choice` | `strict` | `json_schema` | `developer` 角色 | 并行工具调用 |
|---|---|---|---|---|---|
| OpenAI（对照） | `auto` `none` `required` 指定函数 `allowed_tools` | 有，默认关 | 有 | 有 | 有 |
| DeepSeek | 思考模式下 `tool_choice` 用 `required` 或指定函数名，都会 400 | Beta 功能，指南要求改用 `/beta` 地址 | Chat ✗，只有 `json_object`；Responses 的 `text.format` 标为完全支持 | ✗ | Chat —；Responses 恒为并行 |
| 智谱 GLM | 只有 `"auto"` | — | ✗，只有 `json_object` | 未列出 | — |
| Kimi | `auto` `none` `required` 指定函数都有，但 `required` 只有 `kimi-k3` 支持；开思考时指定函数会 400 | 有，**默认开**（只接受它的 JSON Schema 子集） | 有，`strict` 默认 true | 未列出 | 可以一次返回多个；文档没写 `parallel_tool_calls` |
| xAI Grok | `auto` `none` `required` 指定函数都有 | **始终开**（文档说隐式为 true；传 `false` 会怎样没写） | 有 | — | 默认并行，`parallel_tool_calls: false` 关闭 |
| Qwen | `auto` `none` 可用；`required` 各官方页说法互相矛盾（Chat 参考页、函数调用指南、错误码页）；思考模式下 `required` 或指定函数会失败 | — | 有（看模型） | — | `parallel_tool_calls` **默认 false** |
| MiniMax | 未列出（Anthropic 入口标为完全支持） | — | 未列出 | 未列出 | — |
| 豆包 | `auto` `none` `required` 指定函数都有 | 有 | 有（beta） | ✗（传了返回 400） | 有，默认 true |

来源：[DeepSeek Chat 接口参考][ds-chat]；[DeepSeek 工具调用指南][ds-tools]；[DeepSeek Responses 指南][ds-resp]；[DeepSeek Responses 接口参考][ds-create-resp]；[DeepSeek oh-my-pi 接入指南][ds-omp]；[智谱函数调用指南][zp-fc]；[智谱对话补全参考][zp-chat]；[智谱流式输出指南][zp-streaming]；[Kimi Chat 接口参考][kimi-chat]；[Kimi 模型总览][kimi-models]；[Kimi tool_choice 指南][kimi-tc]；[xAI 函数调用指南][xai-fc]；[xAI 结构化输出指南][xai-so]；[xAI 流式指南][xai-stream]；[xAI 费用追踪文档][xai-cost]；[阿里云百炼 Chat 兼容接口参考][ali-chat]；[阿里云百炼函数调用指南][ali-fc]；[阿里云百炼错误码][ali-err]；[MiniMax OpenAI 对话接口参考][mm-oai-chat]；[火山方舟 Chat 接口参考][ark-chat]；[火山方舟 Messages 接口][ark-msg]；[火山方舟 Coding Plan 常见问题][ark-coding]。

---

[ali-chat]: https://www.alibabacloud.com/help/en/model-studio/qwen-api-via-openai-chat-completions
[ali-compat]: https://www.alibabacloud.com/help/en/model-studio/compatibility-of-openai-with-dashscope
[ali-err]: https://help.aliyun.com/zh/model-studio/error-code
[ali-fc]: https://www.alibabacloud.com/help/en/model-studio/qwen-function-calling
[ali-resp]: https://www.alibabacloud.com/help/en/model-studio/qwen-api-via-openai-responses
[ali-think]: https://www.alibabacloud.com/help/en/model-studio/deep-thinking
[an-beta]: https://platform.claude.com/docs/en/api/beta-headers
[an-cache]: https://platform.claude.com/docs/en/build-with-claude/prompt-caching
[an-errors]: https://platform.claude.com/docs/en/api/errors
[an-handle]: https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls
[an-midsys]: https://platform.claude.com/docs/en/build-with-claude/mid-conversation-system-messages
[an-msg]: https://platform.claude.com/docs/en/api/messages
[an-overview]: https://platform.claude.com/docs/en/api/overview
[an-pdf]: https://platform.claude.com/docs/en/build-with-claude/pdf-support
[an-server-tools]: https://platform.claude.com/docs/en/agents-and-tools/tool-use/server-tools
[an-so]: https://platform.claude.com/docs/en/build-with-claude/structured-outputs
[an-stream]: https://platform.claude.com/docs/en/build-with-claude/streaming
[an-think]: https://platform.claude.com/docs/en/build-with-claude/extended-thinking
[an-tools]: https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools
[an-vision]: https://platform.claude.com/docs/en/build-with-claude/vision
[an-working]: https://platform.claude.com/docs/en/build-with-claude/working-with-messages
[ark-chat]: https://www.volcengine.com/docs/82379/1494384
[ark-coding]: https://docs.volcengine.com/docs/ark/coding-plan-personal-faq
[ark-msg]: https://docs.volcengine.com/docs/ark/messages-api
[ark-think]: https://www.volcengine.com/docs/82379/1449737
[ds-cache]: https://api-docs.deepseek.com/guides/kv_cache
[ds-chat]: https://api-docs.deepseek.com/api/create-chat-completion
[ds-create-resp]: https://api-docs.deepseek.com/api/create-response
[ds-omp]: https://api-docs.deepseek.com/zh-cn/quick_start/agent_integrations/oh_my_pi
[ds-resp]: https://api-docs.deepseek.com/guides/responses_api
[ds-think]: https://api-docs.deepseek.com/guides/thinking_mode
[ds-tools]: https://api-docs.deepseek.com/guides/tool_calls
[ds-vision]: https://api-docs.deepseek.com/guides/vision/
[g-3]: https://ai.google.dev/gemini-api/docs/gemini-3
[g-api]: https://ai.google.dev/api
[g-cache]: https://ai.google.dev/gemini-api/docs/generate-content/caching
[g-cache2]: https://ai.google.dev/gemini-api/docs/caching
[g-fc]: https://ai.google.dev/gemini-api/docs/generate-content/function-calling
[g-file]: https://ai.google.dev/gemini-api/docs/generate-content/file-input-methods
[g-gen]: https://ai.google.dev/api/generate-content
[g-image]: https://ai.google.dev/gemini-api/docs/generate-content/image-understanding
[g-int-api]: https://ai.google.dev/api/interactions-api
[g-int-break]: https://ai.google.dev/gemini-api/docs/interactions-breaking-changes-may-2026
[g-int-ov]: https://ai.google.dev/gemini-api/docs/interactions-overview
[g-int-qs]: https://ai.google.dev/gemini-api/docs/interactions/quickstart
[g-int-stream]: https://ai.google.dev/gemini-api/docs/streaming
[g-int-text]: https://ai.google.dev/gemini-api/docs/text-generation
[g-live]: https://ai.google.dev/api/live
[g-sig]: https://ai.google.dev/gemini-api/docs/thought-signatures
[g-textgen]: https://ai.google.dev/gemini-api/docs/generate-content/text-generation
[g-think]: https://ai.google.dev/gemini-api/docs/generate-content/thinking
[g-ver]: https://ai.google.dev/gemini-api/docs/api-versions
[kimi-chat]: https://platform.kimi.ai/docs/api/chat
[kimi-models]: https://platform.kimi.ai/docs/api/models-overview
[kimi-msg]: https://platform.kimi.ai/docs/api/messages
[kimi-tc]: https://platform.kimi.ai/docs/guide/use-tool-choice
[kimi-think]: https://platform.kimi.ai/docs/guide/use-thinking-models
[mm-anth]: https://platform.minimax.io/docs/api-reference/text-anthropic-api
[mm-fc]: https://platform.minimax.io/docs/guides/text-m3-function-call
[mm-oai]: https://platform.minimax.io/docs/api-reference/text-openai-api
[mm-oai-chat]: https://platform.minimax.io/docs/api-reference/text-chat-openai
[mm-resp]: https://platform.minimax.io/docs/api-reference/responses-create
[oa-audio]: https://developers.openai.com/api/docs/guides/audio-chat-completions
[oa-cache]: https://developers.openai.com/api/docs/guides/prompt-caching
[oa-chat]: https://developers.openai.com/api/reference/resources/chat/subresources/completions/methods/create/
[oa-chat-stream]: https://developers.openai.com/api/reference/resources/chat/subresources/completions/streaming-events/
[oa-fc]: https://developers.openai.com/api/docs/guides/function-calling
[oa-file]: https://developers.openai.com/api/docs/guides/file-inputs
[oa-migrate]: https://developers.openai.com/api/docs/guides/migrate-to-responses
[oa-node-helpers]: https://github.com/openai/openai-node/blob/main/docs/helpers.md
[oa-openapi]: https://platform.openai.com/docs/static/api-definition.yaml
[oa-overview]: https://developers.openai.com/api/reference/overview/
[oa-py-stream]: https://github.com/openai/openai-python/blob/main/src/openai/_streaming.py
[oa-reasoning]: https://developers.openai.com/api/docs/guides/reasoning
[oa-resp]: https://developers.openai.com/api/reference/resources/responses/methods/create/
[oa-resp-events]: https://developers.openai.com/api/reference/resources/responses/streaming-events/
[oa-so]: https://developers.openai.com/api/docs/guides/structured-outputs
[oa-vision]: https://developers.openai.com/api/docs/guides/images-vision
[oa-ws]: https://developers.openai.com/api/docs/guides/websocket-mode
[vx-auth]: https://docs.cloud.google.com/vertex-ai/docs/authentication
[xai-43]: https://docs.x.ai/developers/models/grok-4.3
[xai-cmp]: https://docs.x.ai/developers/model-capabilities/text/comparison
[xai-cost]: https://docs.x.ai/developers/cost-tracking
[xai-fc]: https://docs.x.ai/developers/tools/function-calling
[xai-reason]: https://docs.x.ai/developers/model-capabilities/text/reasoning
[xai-so]: https://docs.x.ai/developers/model-capabilities/text/structured-outputs
[xai-stream]: https://docs.x.ai/developers/model-capabilities/text/streaming
[zp-cache]: https://docs.bigmodel.cn/cn/guide/capabilities/cache
[zp-chat]: https://docs.bigmodel.cn/api-reference/%E6%A8%A1%E5%9E%8B-api/%E5%AF%B9%E8%AF%9D%E8%A1%A5%E5%85%A8
[zp-fc]: https://docs.bigmodel.cn/cn/guide/capabilities/function-calling
[zp-streaming]: https://docs.bigmodel.cn/cn/guide/capabilities/streaming
[zp-think]: https://docs.bigmodel.cn/cn/guide/capabilities/thinking
