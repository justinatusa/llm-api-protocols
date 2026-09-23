# 五种原生格式：细节和坑

> 字段对照表在 [字段对照](../field-atlas.md)；这里是各格式容易踩的坑，以及 Gemini 两种格式怎么选。
> 链接文字写明来源的厂商和文档，完整列表见 [来源](sources.md)；“冲突”“未核实”汇总在 [冲突与未核实](conflicts.md)。

## 1. 各格式的坑

**Chat Completions**
- `max_tokens` 已废弃，推理模型用 `max_completion_tokens`（包含推理 token）（[OpenAI Chat Completions 参考][oa-chat]）。
- `tool_calls[].function.arguments` 是字符串，不保证是合法 JSON，要自己校验（[OpenAI OpenAPI 定义][oa-openapi]）。
- 从 GPT-5.4 起，`reasoning_effort` 不是 `none` 时 Chat 不支持工具调用；部分新模型的工具调用只能走 Responses（[OpenAI 迁移到 Responses 指南][oa-migrate]）。
- OpenAI 的态度：Chat 继续支持，新项目推荐 Responses，没有下线日期（[OpenAI 迁移到 Responses 指南][oa-migrate]；[OpenAI 弃用公告][oa-deprec]）。

**Responses**
- `previous_response_id` 串起来的历史照样按输入 token 计费（[OpenAI 迁移到 Responses 指南][oa-migrate]）。
- `truncation` 默认 `"disabled"`，超出上下文直接 400；设 `"auto"` 会从最早的 items 开始丢（[OpenAI Responses 参考][oa-resp]）。
- `background: true` 走异步，轮询 `GET /v1/responses/{id}`，可以用 `starting_after` 续上流（[OpenAI Responses 参考][oa-resp]）。
- Assistants API 已于 2026-08-26 下线，替代方案是 Responses + Conversations（[OpenAI 弃用公告][oa-deprec]）。

**Anthropic Messages**
- `max_tokens: 0` 只写缓存、不生成（[Anthropic Messages 参考][an-msg]）。
- 预填（最后一条放 assistant）会 400：工作指南写 Claude 4.6+，另一页写 Opus 4.7+（冲突）；4.7+ 采样参数只接受默认值，传别的会 400（[Anthropic Messages 使用指南][an-working]）；Messages 参考页另写 `top_p` 可接受 ≥ 0.99、`top_k` 传任何值都拒绝，与指南不完全一致（冲突）（[Anthropic Messages 参考][an-msg]）。
- `tool_choice` 关并行用 `disable_parallel_tool_use`；哪些型号不接受 `any` / `tool` 见 [字段对照 · 工具](../field-atlas.md#2-工具-重点)（[Anthropic 定义工具文档][an-tools]；[Anthropic 错误文档][an-errors]；[Claude Opus 5.5 新特性说明][an-opus55]）。
- 服务端工具不用你回传结果；跑到循环上限会返回 `pause_turn`，原样续发即可（[Anthropic 服务端工具文档][an-server-tools]）。

**Gemini `generateContent`**
- 模型名写在 URL 路径里，不在请求体里（[Gemini generateContent 参考][g-gen]）。
- 参考页用 camelCase（如 `finishReason`），官方 curl 示例里又有 snake_case（如 `system_instruction`）。Gemini 文档没说两种都收；Google API 底层的 proto3 JSON 规则要求服务端两种都接受，但没有针对 Gemini 实测（[Gemini generateContent 参考][g-gen]；[protobuf JSON 映射规则][proto-json]）。
- `Content.role` 写明只能 `user` / `model`，同页函数声明说明又提到 `"function"`（冲突，见 [冲突与未核实](conflicts.md)）（[Gemini generateContent 参考][g-gen]）。
- 被安全策略拦截时可能一个候选都没有，要看 `promptFeedback.blockReason`（[Gemini generateContent 参考][g-gen]）。
- Vertex AI：路径是 `/v1/projects/{P}/locations/{L}/publishers/google/models/{M}:generateContent`，用 OAuth Bearer；响应多出 `createTime` 和 `usageMetadata.trafficType`（[Vertex AI 函数调用文档][vx-fc]；[Vertex AI GenerateContentResponse 参考][vx-resp]）。

## 2. Gemini：用 Interactions 还是 `generateContent`

| | `generateContent` | Interactions |
|---|---|---|
| 官方定位 | 已称 legacy，但仍完全支持；GA 博客说可预见的将来仍会接新的主力模型 | 2026-06 GA，推荐所有新项目；迁移页说新模型、新工具和 agent 功能都会在这里发布 |
| 历史 | 每次全量发 `contents` | 服务端默认记；`store=false` 时不能续，也不能后台执行（写法见 [导读](../llm-api-protocols.md)、[字段对照 · 工具](../field-atlas.md#2-工具-重点)，流式见 [字段对照 · 流式](../field-atlas.md#3-流式)） |
| 错误体 | `{error:{code, message, status, details}}`，`code` 是数字 | `{error:{code, message}}`，`code` 是字符串；取消是 499 |
| 暂不支持 | — | 批处理、显式缓存、Python 自动函数调用、自定义安全设置（参考页仍列 `safety_settings`，冲突） |
| 什么时候用 | 需要批处理、显式缓存或自定义安全设置时 | 新项目、要服务端记历史、agent 类功能 |

来源：[Gemini Interactions 概览][g-int-ov]；[Gemini 迁移到 Interactions 指南][g-int-mig]；[Google Interactions GA 博客][g-int-blog]；[Gemini Interactions 参考][g-int-api]；[Gemini 流式文档（Interactions）][g-int-stream]；[Gemini API 错误文档（Interactions）][g-int-err]；[Gemini generateContent 错误文档][g-err]；[Gemini 缓存文档][g-cache2]。Gemini 的 OpenAI 兼容层（`/v1beta/openai/`）是另一个入口，文档没提它和 Interactions 的关系（[Gemini OpenAI 兼容文档][g-oai]）。

---

[an-errors]: https://platform.claude.com/docs/en/api/errors
[an-msg]: https://platform.claude.com/docs/en/api/messages
[an-opus55]: https://platform.claude.com/docs/en/models/opus-5-5/whats-new-opus-5-5
[an-server-tools]: https://platform.claude.com/docs/en/agents-and-tools/tool-use/server-tools
[an-tools]: https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools
[an-working]: https://platform.claude.com/docs/en/build-with-claude/working-with-messages
[g-cache2]: https://ai.google.dev/gemini-api/docs/caching
[g-err]: https://ai.google.dev/gemini-api/docs/generate-content/api-errors
[g-gen]: https://ai.google.dev/api/generate-content
[g-int-api]: https://ai.google.dev/api/interactions-api
[g-int-blog]: https://blog.google/innovation-and-ai/technology/developers-tools/interactions-api-general-availability/
[g-int-err]: https://ai.google.dev/gemini-api/docs/api-errors
[g-int-mig]: https://ai.google.dev/gemini-api/docs/migrate-to-interactions
[g-int-ov]: https://ai.google.dev/gemini-api/docs/interactions-overview
[g-int-stream]: https://ai.google.dev/gemini-api/docs/streaming
[g-oai]: https://ai.google.dev/gemini-api/docs/openai
[oa-chat]: https://developers.openai.com/api/reference/resources/chat/subresources/completions/methods/create/
[oa-deprec]: https://developers.openai.com/api/docs/deprecations
[oa-migrate]: https://developers.openai.com/api/docs/guides/migrate-to-responses
[oa-openapi]: https://platform.openai.com/docs/static/api-definition.yaml
[oa-resp]: https://developers.openai.com/api/reference/resources/responses/methods/create/
[proto-json]: https://protobuf.dev/programming-guides/json/
[vx-fc]: https://cloud.google.com/vertex-ai/generative-ai/docs/multimodal/function-calling
[vx-resp]: https://cloud.google.com/vertex-ai/generative-ai/docs/reference/rest/v1/GenerateContentResponse
