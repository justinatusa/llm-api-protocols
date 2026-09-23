# 流式、停止与断开

> 先讲写代码时都会遇到的停止、报错、断开和重试，再列第三方入口的流式差异。五种原生格式的流式对照在 [字段对照 · 流式](../field-atlas.md#3-流式)。
> 链接文字写明来源的厂商和文档，完整列表见 [来源](sources.md)；“冲突”“未核实”汇总在 [冲突与未核实](conflicts.md)。

## 流断了、你取消了、错误从哪冒出来

字段细节在 [字段对照](../field-atlas.md) 和 [错误、重试与限流](ops.md)，这里只讲写代码时要先想清楚的事。

**为什么停了**
- `finish_reason`（Chat）、`status` 加 `incomplete_details.reason`（Responses）、`stop_reason`（Anthropic）、`finishReason`（`generateContent`）、`status`（Interactions）说的都是“为什么停了”，只是名字不同（[OpenAI Chat Completions 参考][oa-chat]；[OpenAI Responses 参考][oa-resp]；[Anthropic Messages 参考][an-msg]；[Gemini generateContent 参考][g-gen]；[Gemini Interactions 参考][g-int-api]）。
- 流式时，中间的块里这个字段通常是空的，要看最后一块或结束事件（[OpenAI Chat Completions 参考][oa-chat]；[Anthropic 流式文档][an-stream]）。
- 正常说完、命中停止序列、到了长度上限、要调工具、被安全策略拦下，是不同的结局，不要都当成失败处理（[OpenAI Chat Completions 参考][oa-chat]；[Anthropic Messages 参考][an-msg]；[Gemini generateContent 参考][g-gen]）。
- 有些兼容入口命中 `stop_sequences` 时仍报告正常结束：Kimi 和 Qwen 的 Anthropic 入口返回 `end_turn`（[Kimi Messages 接口参考][kimi-msg]；[阿里云百炼 Anthropic 兼容接口][ali-anth]）。

**错误从哪冒出来**
- 流开始之前出错，拿到的是普通的 HTTP 状态码和 JSON 错误体（各家的错误体见 [错误、重试与限流](ops.md)）（[Anthropic 错误文档][an-errors]；[Gemini API 错误文档（Interactions）][g-int-err]）。
- 已经返回 HTTP 200、流也开始了之后，错误会作为流里的一个事件出现，不走上面那套；Anthropic 官方明确这么写（[Anthropic 错误文档][an-errors]），Responses 和 Interactions 也各有流内 `error` 事件（[OpenAI Responses 流式事件参考][oa-resp-events]；[Gemini API 错误文档（Interactions）][g-int-err]）。
- 同样是 HTTP 429，可能是限流，也可能是额度用完，要先看错误体里的类型再决定重不重试：OpenAI 看 `error.code`，Kimi 看 `error.type`，Interactions 分 `rate_limit_exceeded` / `quota_exceeded` / `too_many_requests`（[OpenAI 错误码指南][oa-errors]；[Kimi 常见问题排查][kimi-trouble]；[Gemini API 错误文档（Interactions）][g-int-err]）。

**连接断了、你取消了**
- 客户端中途关掉连接，通常拿不到干净的停止原因；Gemini 记为 499 `CANCELLED`，Kimi 记为 499（[Gemini generateContent 错误文档][g-err]；[Kimi 错误码][kimi-err]）。
- 已生成的部分会不会计费，各家都没写（未核实）；Kimi 只写了被 429 打断的请求不计费，这不是客户端主动断开（[Kimi 常见问题排查][kimi-trouble]）。
- OpenAI Responses 的同步请求，取消的办法就是断开连接；只有 `background: true` 的请求能调 `POST /v1/responses/{id}/cancel`；Interactions 的后台任务也有 `cancel`（[OpenAI 后台模式指南][oa-bg]；[OpenAI Responses 取消接口参考][oa-resp-cancel]；[Gemini 后台执行文档][g-bg]）。
- 流断了基本不能从断点接着收：只有 Responses 的后台流（`starting_after`）和 Interactions 的流（重新拉取时带 `last_event_id` 并设 `stream: true`，后台任务也能这样续）能续（[OpenAI 后台模式指南][oa-bg]；[Gemini 后台执行文档][g-bg]；[Gemini Interactions 参考][g-int-api]）。Anthropic 的建议是发一个新请求，让模型从已收到的文字接着写（4.6+ 要用 user 消息，不能预填）（[Anthropic 流式文档][an-stream]）。其他情况重试就是重新生成一遍，不要把两次的半截结果拼在一起。
- 如果半截结果里已经执行过工具，盲目重试可能让工具的副作用发生两次。

**超时和重试**
- 整个请求的超时和“多久没收到新字节”不是一回事；推理模型可能很久不出字，超时设短了会被误杀。OpenAI 和 Anthropic 的 Python SDK 默认整体超时 10 分钟，Anthropic 的 SDK 文档另外提醒有些网络会断掉空闲连接（[OpenAI Python SDK README][oa-py-readme]；[Anthropic Python SDK 文档][an-py-sdk]）。
- OpenAI 和 Anthropic 的 Python SDK 默认会对连接错误、408、409、429、≥500 自动重试 2 次；Gemini 的 Python SDK 是否默认重试，排查页和源码说法不一（见 [冲突与未核实](conflicts.md)）（[OpenAI Python SDK README][oa-py-readme]；[Anthropic Python SDK 文档][an-py-sdk]；[Gemini 排查指南][g-trouble]；[Google GenAI Python SDK 源码][g-sdk-client]）。OpenAI 的 README 写明，流已经开始接收后不会自动重试，因为重放请求可能生成重复内容（[OpenAI Python SDK README][oa-py-readme]）。只要是真的重新发了请求（SDK 重试或你自己重试），模型就会重新生成一遍，按常理也会再计一次费（计费规则官方没专门写）。
- 解析流时要跳过空行和以 `:` 开头的注释行，不要当 JSON 解析：DeepSeek 发 `: keep-alive`，Qwen 原生接口有 `:HTTP_STATUS/200`（[DeepSeek 限速说明][ds-rate]；[阿里云百炼流式输出文档][ali-stream]）；OpenAI 的 Python SDK 会忽略以 `:` 开头的行（[OpenAI Python SDK 源码（流式解析）][oa-py-stream]）。Anthropic 的 `ping` 不是注释行，而是一个正常事件（`event: ping`，`data` 里 `type` 为 `ping`），解析后忽略即可（[Anthropic 流式文档][an-stream]）。

### 断连检查清单

| 情况 | 你会看到什么 | 该怎么做 | 来源 |
|---|---|---|---|
| 排队、还没开始生成 | DeepSeek：非流式收到空行，流式收到 `: keep-alive`；10 分钟还没开始就断开 | 解析器跳过空行和注释行；别把它当错误 | [DeepSeek 限速说明][ds-rate] |
| 生成中长时间没新字节 | 连接可能被网络或代理当成空闲断掉 | 推理模型把超时调长（xAI 示例 3600 秒）；区分整体超时和空闲超时 | [Anthropic Python SDK 文档][an-py-sdk]；[xAI 异步请求指南][xai-async] |
| 你主动取消或客户端断开 | 没有干净的停止原因；Gemini、Kimi 记 499 | 当成“没完成”；是否计费官方没写 | [Gemini generateContent 错误文档][g-err]；[Kimi 错误码][kimi-err] |
| 断了想重来 | 只有 Responses 的后台流和 Interactions 的流（`last_event_id`）能续 | 其他情况重新请求；已执行过工具的，先查副作用再重试 | [OpenAI 后台模式指南][oa-bg]；[Gemini 后台执行文档][g-bg] |

## 第三方入口对照

| 入口 | 结束标记 | 流中出错 | 用量在哪 | 其他 | 来源 |
|---|---|---|---|---|---|
| DeepSeek Chat | `data: [DONE]` | 没写错误事件；中断时 `finish_reason` 为 `aborted` | 最后一个带 choice 的块，没有空 `choices` 块 | 排队时发 `: keep-alive` 注释行 | [DeepSeek Chat 接口参考][ds-chat]；[DeepSeek 限速说明][ds-rate] |
| DeepSeek Responses | `response.completed` / `incomplete` / `failed`，明说没有 `[DONE]` | `response.failed` 里带 `error` | `response.completed` 里 | `event:` 行和 `type` 一致 | [DeepSeek Responses 接口参考][ds-create-resp] |
| 智谱原生 | `data: [DONE]` | 原因看 `finish_reason`（如 `network_error`） | 最后一个带 `finish_reason` 的块 | `tool_stream: true` 时工具参数逐步返回 | [智谱对话补全参考][zp-chat]；[智谱流式输出指南][zp-streaming]；[智谱工具流式输出指南][zp-stream-tool] |
| Kimi Chat | `data: [DONE]` | 没写 | 参考页：`[DONE]` 前多一块，`choices` 为空；同页示例却放在结束块上（冲突） | 工具参数按 `index` 拼 | [Kimi Chat 接口参考][kimi-chat]；[Kimi 流式输出指南][kimi-stream]；[Kimi 工具调用指南][kimi-tc2] |
| Kimi Anthropic 入口 | `message_stop` | 没写流内错误事件 | 没写 | `event:` 与 `data.type` 一致 | [Kimi Messages 接口参考][kimi-msg] |
| xAI Chat | `data: [DONE]` | 没写 | 计费页：开 `include_usage` 后只在最后一块（`choices` 为空）；流式示例却每块都带（冲突） | 函数调用整个放在一块里下发，不跨块 | [xAI 流式指南][xai-stream]；[xAI 费用追踪文档][xai-cost]；[xAI 函数调用指南][xai-fc] |
| Qwen 兼容模式 | `data: [DONE]` | 没写 | 开 `include_usage` 后最后一块，`choices` 为空 | 工具参数分块到达，自己拼 | [阿里云百炼流式输出文档][ali-stream]；[阿里云百炼函数调用指南][ali-fc] |
| Qwen 原生 | 看 `finish_reason` 不为空；示例里没有 `[DONE]` | 没写 | 每块都带实时用量 | 请求头 `X-DashScope-SSE: enable`；`event:` 固定为 `result`；有 `:HTTP_STATUS/200` 注释行 | [阿里云百炼流式输出文档][ali-stream] |
| MiniMax OpenAI 入口 | 没写 | 没写 | 要开 `include_usage`，只在最后一块 | 工具参数流式没写 | [MiniMax OpenAI 对话接口参考][mm-oai-chat] |
| 豆包 Chat | 通常以 `data: [DONE]` 结束 | 中途出错也能拿到已生成的内容，错误事件名没写 | 开 `include_usage` 后 `[DONE]` 前多一块；`chunk_include_usage` 每块给累计值 | — | [火山方舟流式输出文档][ark-stream]；[火山方舟 Chat 接口参考][ark-chat] |
| 豆包 Responses | `response.completed` 之后还有 `data: [DONE]` | 没写 | `response.completed` 里 | `event:` 行和 `type` 一致 | [火山方舟流式输出文档][ark-stream] |

DeepSeek、智谱、MiniMax 的 Anthropic 入口都没写流式事件名和错误格式（[DeepSeek Anthropic 兼容指南][ds-anth]；[智谱 Claude 兼容说明][zp-claude-compat]；[MiniMax Anthropic 兼容接口][mm-anth]）。

---

[ali-anth]: https://www.alibabacloud.com/help/en/model-studio/anthropic-api-messages
[ali-fc]: https://www.alibabacloud.com/help/en/model-studio/qwen-function-calling
[ali-stream]: https://www.alibabacloud.com/help/en/model-studio/stream
[an-errors]: https://platform.claude.com/docs/en/api/errors
[an-msg]: https://platform.claude.com/docs/en/api/messages
[an-py-sdk]: https://platform.claude.com/docs/en/api/sdks/python
[an-stream]: https://platform.claude.com/docs/en/build-with-claude/streaming
[ark-chat]: https://www.volcengine.com/docs/82379/1494384
[ark-stream]: https://docs.volcengine.com/docs/82379/2123275
[ds-anth]: https://api-docs.deepseek.com/guides/anthropic_api
[ds-chat]: https://api-docs.deepseek.com/api/create-chat-completion
[ds-create-resp]: https://api-docs.deepseek.com/api/create-response
[ds-rate]: https://api-docs.deepseek.com/quick_start/rate_limit
[g-bg]: https://ai.google.dev/gemini-api/docs/background-execution
[g-err]: https://ai.google.dev/gemini-api/docs/generate-content/api-errors
[g-gen]: https://ai.google.dev/api/generate-content
[g-int-api]: https://ai.google.dev/api/interactions-api
[g-int-err]: https://ai.google.dev/gemini-api/docs/api-errors
[g-sdk-client]: https://github.com/googleapis/python-genai/blob/main/google/genai/_api_client.py
[g-trouble]: https://ai.google.dev/gemini-api/docs/troubleshooting
[kimi-chat]: https://platform.kimi.ai/docs/api/chat
[kimi-err]: https://platform.kimi.ai/docs/api/errors
[kimi-msg]: https://platform.kimi.ai/docs/api/messages
[kimi-stream]: https://platform.kimi.ai/docs/guide/utilize-the-streaming-output-feature-of-kimi-api
[kimi-tc2]: https://platform.kimi.ai/docs/guide/use-kimi-api-to-complete-tool-calls
[kimi-trouble]: https://platform.kimi.ai/docs/guide/troubleshooting
[mm-anth]: https://platform.minimax.io/docs/api-reference/text-anthropic-api
[mm-oai-chat]: https://platform.minimax.io/docs/api-reference/text-chat-openai
[oa-bg]: https://developers.openai.com/api/docs/guides/background
[oa-chat]: https://developers.openai.com/api/reference/resources/chat/subresources/completions/methods/create/
[oa-errors]: https://developers.openai.com/api/docs/guides/error-codes
[oa-py-readme]: https://github.com/openai/openai-python/blob/main/README.md
[oa-py-stream]: https://github.com/openai/openai-python/blob/main/src/openai/_streaming.py
[oa-resp]: https://developers.openai.com/api/reference/resources/responses/methods/create/
[oa-resp-cancel]: https://developers.openai.com/api/reference/resources/responses/methods/cancel
[oa-resp-events]: https://developers.openai.com/api/reference/resources/responses/streaming-events/
[xai-async]: https://docs.x.ai/developers/advanced-api-usage/async
[xai-cost]: https://docs.x.ai/developers/cost-tracking
[xai-fc]: https://docs.x.ai/developers/tools/function-calling
[xai-stream]: https://docs.x.ai/developers/model-capabilities/text/streaming
[zp-chat]: https://docs.bigmodel.cn/api-reference/%E6%A8%A1%E5%9E%8B-api/%E5%AF%B9%E8%AF%9D%E8%A1%A5%E5%85%A8
[zp-claude-compat]: https://docs.bigmodel.cn/cn/guide/develop/claude/introduction
[zp-stream-tool]: https://docs.bigmodel.cn/cn/guide/capabilities/stream-tool
[zp-streaming]: https://docs.bigmodel.cn/cn/guide/capabilities/streaming
